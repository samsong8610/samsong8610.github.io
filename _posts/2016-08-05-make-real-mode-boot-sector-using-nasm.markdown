---
layout: post
title:  "使用nasm制作和调试实模式引导扇区程序"
date:   2016-08-05
categories: linux assembly kernel
---

## 写在前面

很久之前，我就有计划要学习Linux的内核代码，但是自己的工作内容和内核没有关系，用一些零散时间用来研读内核代码感觉总是浅尝辄止，结果就是口号喊了不知道多少次，自己还是在原地踏步。

最近接触到赵炯博士的免费书《Linux内核完全注释》，基于早期的内核版本0.11编写。这个老版本的内核代码量不大但是核心的组件都有涉及。遇到这么好的入门指南，又激起了我的雄心壮志，打算再试着啃一啃Linux内核代码（虽然是婴儿期版本）。这次我也不想只是欣赏Linus们当年的足迹，打算仿照大师的思路，自己动手把核心的组件实现一遍，过程中的一些心得经验就记录在这个主页上，希望自己能坚持下来。

## 关于boot sector

boot sector是引导磁盘的第一个扇区里的程序，大小是512B，最后两个字节是0x55, 0xAA。当计算机上电后，BIOS会从引导扇区把它加载到0x7C00内存地址处，并跳转到这里执行。Linux内核中的boot sector会进一步加载setup程序和系统内核，并最终将控制权交给内核程序。

boot sector是一个16位实模式程序，gas不支持编译生成该模式的程序。Linux 0.11使用了as86和ld86来完成编译链接，fedora上可以安装dev86包获得。此外nasm也可以生成16位程序，且相关的资料比较多，这篇文章就是主要介绍怎么使用nasm以及gdb和qemu模拟器制作调试引导扇区程序。

## 仿制的boot sector

本文重点讲解工具的使用，示例程序没有像实际的bootsect一样实现各种功能，仅仅是向终端输出一行字符串。

{% highlight assembly %}
        BITS 16                         ; 16 bits real mode                
        DATASEG equ 0x07C0              ; BIOS load this to 0x7C00,
                                        ; so set DS=0x07C0 to make label msg1 works
        global _start

_start:
        cli                             ; disable interrupt
.go:
        mov ax, DATASEG                 ; get current code segment
        mov ds, ax
        mov ss, ax                      ; set stack to 0x07C0:0xFFFF
        mov sp, 0xFFFF
        
        mov si, msg1                                                       
        call print                                                         
        
        hlt                                                                

;
; print: print a string
; input: si - the address of the string to output
;
print:
        mov ah, 0x0E                                                       
.char:
        lodsb
        cmp al, 0                                                          
        jz .ret
        int 0x10
        jmp .char                                                          
.ret:
        ret                                                                

msg1:
        DB "Loading system ...", 0x0D, 0x0A

        TIMES 510 - ($ - $$) DB 0
        DW 0xAA55
{% endhighlight %}

以上代码设置DS、SS和代码段重叠，堆栈设置到0x7CFF。然后调用print子程序通过BIOS中断0x10输出字符串 `Loading system ...` 。最后一个字设置成0xAA55，表示本扇区是有效的引导扇区，中间全部用0填充，这样这段程序刚好填满启动扇区。关于`INT 0x10`的功能以及参数说明，参考[这篇wiki][int-10h]。

## 编译链接

nasm的-f选项可以指定输出格式，bin是二进制输出，直接将编译后的字节码保存，直接写入到磁盘第一个扇区就可以用于启动了。boot sector是直接运行在实模式下的16位代码，实际上不需要链接处理，为了gdb调试时能够使用符号，我们也链接生成一个elf格式的文件，专门用于调试。
编译和链接使用make命令完成，Makefile如下：

{% highlight makefile %}
LD=ld
AS=nasm

.PHONY: all clean

all: boot.img boot.elf

clean:
	rm -f *.o *.elf *.img

boot.o: boot.s
	${AS} -f elf -F dwarf -o $@ $<

boot.img: boot.s
	${AS} -f bin -o $@ $<

boot.elf: boot.o
	${LD} -Ttext=0x7c00 -melf_i386 -o $@ $<
{% endhighlight %}

生成boot.elf的规则，必须使用-Ttext=0x7c00指定代码段的绝对位置，不然的话ld会作为32位elf来处理，并把代码段定位到0x08048000，导致链接时报 `relocation truncated to fit: R_386_16 against '.text'` 的错误，详细解释参见[这篇stackoverflow][relocation-truncated]。

## 使用gdb和qemu调试

boot sector必须运行在实模式下，不能直接在linux系统内启动。我们可以通过虚拟机来模拟运行。推荐使用qemu，它内置gdb stub支持，可以远程调试。

首先，把boot.img作为hda启动qemu，-s参数是监听1234端口等待gdb连接，-S参数是默认中断执行，等待continue命令继续。

{% highlight bash %}
qemu-system-i386 -hda boot.img -s -S&
{% endhighlight %}

然后，使用gdb连接qemu，进行调试。 `set architecture i8086` 使gdb能够正确反汇编8086指令，详细说明参见[这篇stackoverflow][debug-16-bit-assembly]。

{% highlight bash %}
gdb boot.elf -ex 'target remote :1234' -ex 'set architecture i8086'\
 -ex 'layout asm' -ex 'layout regs'\
 -ex 'b _start'
{% endhighlight %}

[int-10h]:https://en.wikipedia.org/wiki/INT_10H
[relocation-truncated]:http://stackoverflow.com/questions/34995239/nasm-ld-relocation-truncated-to-fit-r-386-16
[debug-16-bit-assembly]:http://stackoverflow.com/questions/32955887/how-to-disassemble-16-bit-x86-boot-sector-code-in-gdb-with-x-i-pc-it-gets-tr


