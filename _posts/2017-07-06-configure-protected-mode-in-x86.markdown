---
layout: post
title:  "x86平台启用保护模式"
date:   2017-07-06
categories: assembly protected-mode
---

## 分段segmentation

当处理器还处于16位的8086年代时，地址线只有16位，可以直接寻址的内存最大只能64K。但是硬件发展总是出乎设计师们的意料，突破过去的一些假设，这也给设计师带来了难题，即要保持向后兼容，又得支持更新更强的硬件。为了解决16位地址线如何寻址更大内存的问题，intel引入了分段机制。

应用程序直接使用的地址不再是物理地址，而是由`seg:offset`两部分组成的**逻辑地址**，其中seg是段寄存器，offset是段内偏移量。段寄存器存储最终物理地址的高16位，这样通过分段，应用程序可以使用20位地址线，最大寻址1M内存。这种寻址模式叫做 **实模式** （real-address mode）。需要注意的是段内偏移量是16位的，比地址线除去段选择符后只剩下4位空闲，这就导致相临的段是重叠的。

比如，物理地址`0x00010`即可以用`0x0000:0x0010`逻辑地址访问，也可以通过`0x0001:0x0000`逻辑地址访问。

从80386开始，地址线升级到了32位，可寻址内存也扩展到了4G。分段机制已经不再是为了解决内存寻址问题了，而是为系统软件或者应用软件提供内存按用途分段使用、数据保护等目的。这个分段机制和分页机制共同组成了保护模式。

保护模式中的分段是通过 **描述符表（Descriptor Table）** 来定义的，分为全局描述符表（GDT）和局部描述符表（LDT）。每个描述符定义了段的基地址、限长、类型、访问权限信息，长度8字节。访问内存时还是使用`seg:offset`格式的逻辑地址，而其中的seg是16位长的一个段选择符（Segment Selector），包含该段的描述符在描述符表中的索引、段类型（G/L）以及访问权限。

段选择符中索引部分占用13位，段类型（G/L）占用1位，因此最多支持8K个全局段和8K个局部段。每个段的限长占用20位，还支持字节和4K两种粒度，因此最多可以支持4G。

当程序使用逻辑地址访问一个内存字节时，处理器从逻辑地址中的段选择符确定段描述符，再从该描述符中得到段的基地址，加上逻辑地址中的偏移量，得到内存字节的**线性地址**，不考虑分页机制的情况下，这个地址就是字节的物理地址。期间处理器会自动完成保护模式的权限判断。

全局描述符表的起始线性地址和限长由GDTR寄存器保存。局部描述符表是需要在全局描述符表中作为一个段描述符来定义的，段选择符由LDTR寄存器保存。两个寄存器分别使用LGDT/SGDT和LLDT/SLDT来加载、保存。

![实模式和保护模式寻址方式比较]({{site.url}}/assets/real-mode-vs-protected-mode.png)

图片来源：《Linux内核完全注释》

这张图对比了两种模式下寻址方式的差异，可以看出保护模式的寻址处理要复杂不少。

## 保护

在实模式下，所有的段对任意程序都是可以读、写和可执行的。而在保护模式下，分段和分页都提供了多层次的内存访问保护，这里只是说明分段所提供的保护措施。

1. 限长检查：防止访问段之外的地址
2. 类型检查：判断指令与所操作的段类型是否冲突，例如，只能加载一个代码段到CS段寄存器
3. 特权级检查：

    处理器支持0－3四级特权级别，数字越小特权越大。0级一般是用于操作系统内核，1，2级可以用于操作系统服务，3级用于应用程序。如果只使用2个级别，需要选择0和3。Linux就只使用了0和3两个级别。

    特权级检查防止低特权的程序访问高特权的段。这个过程涉及几类特权级别。

    - CPL(Current Privilege Level)：指当前运行的程序的特权级，由程序所运行指令所在代码段的特权级确定。
    - DPL(Descriptor Privilege Level)：指段描述符或者门的特权级。
    - RPL(Requested Privilege Level)： 指段选择符中指定的特权级，如果值比CPL大，RPL会代替CPL的值执行检查，即选择更低的特权级。

    特权级检查针对不同类型的段，处理逻辑也不同。

    - 访问数据段： max(RPL, CPL) <= DPL时，可以访问
    - 跳转代码段： 对于非一致代码段(nonconforming)，CPL == DPL时，可以跳转；对于一致代码段(conforming)，CPL >= DPL时，可以跳转，但是CPL保持不变。一致代码段用于允许应用访问内核中的不需要执行权限控制的功能模块，例如math以及异常处理器。
    - 门调用：可以切换到DPL <= CPL的代码段。如果切换到高特权级代码，处理器会自动完成栈切换。各个特权级的栈SS和ESP都保存在当前任务的TSS中。

## 保护模式切换示例

为了保持向后兼容，处理器重启后会自动进入实地址模式，需要程序主动切换进入保护模式。配置保护模式就是准备好它需要的数据结构，然后启用CR0的bit0 PE就可以了。

基本的保护模式数据结构就是定义不同限长、不同用途、不同访问权限的段。在这个示例中定义了两个段，一个代码段，一个数据段，但是这两个段是重叠的。为了段内寻址简单，两个段的基地址都设置在0x0处，长度2M。

进入保护模式后，BIOS的中断服务就失效了，要向终端输出字符串只能直接向显存按照格式要求写入要输出的字符串。

```nasm
;;
;; Load a simple kernel from disk and switch into protected mode
;; The kernel size must be less than a sector, 512 bytes.
;;

    bits 16             ; bootsect is 16 bits code
    org 0x7c00          ; bios jmp here to run application

welcome:
    mov ah, 0x13        ; write string
    mov al, 0x01        ; string only contains character, update cursor
    mov bh, 0           ; page number
    mov bl, 0x07        ; format, lightgrey font on black background
    mov cx, [welcome_str_len] ; string length
    mov dx, 0           ; start from (0,0)
    mov bp, welcome_str ; string to write
    int 0x10

enable_a20:
    mov ax, 0x2401
    int 0x15

switch_to_protected_mode:
    cli                 ; disable interrupts
    lgdt [gdtr_value]   ; load gdt
    lidt [idtr_value]   ; load idt
    mov eax, cr0        ; enable PE in CR0:0
    or eax, 0x0001
    mov cr0, eax
    jmp 0x08:start      ; goto the new segment at once after enable PE

start:
    bits 32             ; now we can use 32 bits code
    ; setup data segment
    xor eax, eax
    mov ax, 0x10
    mov ds, ax
    mov es, ax
    mov fs, ax
    mov gs, ax
    ; setup stack
    mov ss, ax
    mov esp, 0x1fffff

    ; say hello from kernel, BIOS service is gone!
    mov esi, hello_kernel_str
    mov edi, 0xb8000
_print:
    mov al, [esi]
    cmp al, 0
    jz endless
    mov ah, 0x9c            ; char format, bg:blue fg:red highlight
    mov [edi], ax
    add edi, 2
    inc esi
    jmp _print

endless:
    jmp endless

    align 8
gdt_start:
    dq 0                    ; null descriptor
    dq 0x00c09a0000000200   ; kernel code segment, base 0x0, limit 0x200*4K=2M, DPL=0
    dq 0x00c0920000000200   ; kernel data segment, base 0x0, limit 0x200*4K=2M, DPL=0
gdt_end:

gdtr_value:
    dw gdt_start - gdt_end - 1  ; base+limit is the address of the LAST byte, so minus 1
    dd gdt_start

; Ignore IDT at all
idtr_value:
    dw 0
    dd 0

welcome_str:
    db "Welcome to use protected-mode demo.", 13, 10
welcome_str_len:
    dw welcome_str_len - welcome_str
hello_kernel_str:
    db "Hello, kernel!", 0

    times 510 - ($ - $$) db 0   ; fill the rest of the sector with 0
    dw 0xAA55                   ; boot signature
```
代码编译运行方法参见[Makefile](http://github.com/samsong8610/asm-notes/blob/master/protected-mode/Makefile)。