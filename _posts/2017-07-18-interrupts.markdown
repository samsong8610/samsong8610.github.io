---
layout: post
title:  "x86平台通过中断实现一个系统调用"
date:   2017-07-18
categories: assembly interrupts
---

中断和异常是系统外部、处理器或者正在运行的程序在一定条件下发生的事件，这些事件要求处理器优先响应。发生中断的典型结果会导致当前程序切换到一个中断或异常处理程序执行。

中断由硬件信号随机触发产生，也可以由软件使用`INT n`指令生成。异常是当处理器执行指令过程中检测到错误时生成的。中断处理程序完成中断处理后会恢复被中断程序的执行，除非发生了无法恢复的异常情况。

每个中断和异常都有一个唯一标识号，中断号，代表中断描述符表IDT（interrupt descriptor table）中的索引，取值范围0－255。其中0－31预留给处理器定义的中断和异常。其它32－255供用户使用。

## 中断和异常来源

中断有两个来源：处理器外部硬件产生和软件产生。中断又可以根据是否可以屏蔽分为两类。EFLAGS的IF位可以控制屏蔽状态，清除IF可以屏蔽可屏蔽中断。

异常有三个来源：处理器检测到程序错误、软件产生和机器检测产生。

异常分为三类：
- 故障（Fault）：通常可以纠正的异常，中断处理的返回地址是产生故障的指令以便重新执行。
- 陷阱（Trap）：由特殊指令产生，如调试、断点等，返回地址是产生陷阱指令的下一条指令。
- 中止（Abort）：用于报告严重错误，且不允许继续执行。

## x86中断编程数据结构

x86中断编程的数据结构在`Intel 64 and IA-32 Architectures Software Developer's Manual Volume 3`中有详细的说明，这是只是简要地摘取了各别关键点。

#### IDTR

x86平台下中断或异常处理程序是由一个类似全局描述符表的数据结构定义的，叫做**中断描述符表**，每个中断号一条记录，长度8字节，总共256个。这个数据结构可以位于线性地址的任意位置，处理器根据IDTR寄存器的值来定位IDT。IDTR寄存器共有48位，高32位是IDT的基地址，低16位是IDT以字节为单位的限长，基地址＋限长是IDT最后一个字节的地址。

IDT与GDT不同的是，0索引处也可以正常保存描述符。

IDTR通过LIDT和SIDT指令加载和保存。指令只允许在CPL=0时执行。

#### IDT

IDT是由一组8字节长描述符组成的数组，可以保存三种类型的门描述符（gate descriptor）：
- 中断门描述符(Interrupt gate)
- 陷阱门描述符(Trap gate)
- 任务门描述符(Task gate)

中断门和陷阱门都包含一个长调用指针（段选择符和入口偏移量），两者的区别在于中断门处理中会清除`EFLAGS`的`IF`标志位防止中断门受其它中断的干扰，而陷阱门则不会。任务门是通过切换到一个特定任务来处理中断，可以实现中断处理环境的完全隔离，但是由于处理器在切换任务时需要保存大量状态，任务门处理中断的性能有一定劣势。

## 中断和异常处理过程

中断和异常处理过程的伪代码如下：

```
if 中断处理程序描述符的DPL < CPL then
    从当前TSS中选择CPL对应的栈，作为中断处理程序运行所使用的栈，并将被中断程序的`SS`和`ESP`入栈
end
把`EFLAGS`、`CS`、`EIP`入栈
if 异常有错误码 then
    将错误码入栈
end
```

#### 特权保护规则

中断和异常处理过程在执行之前会进行以下特权检查：

- 如果CPL大于中断处理程序的DPL，产生一般保护异常
- 如果是硬件中断或者处理器生成的异常，忽略DPL的检查

## 示例

全部的示例代码可以参见[interrupts项目](http://github.com/samsong8610/asm-notes/tree/master/interrupts)，这里只摘录一些关键点。

interrupts示例中的内核定义在`kernel.s`文件，同时也链接了一个应用程序`app.c`，应用程序运行在level 3特权级，并通过`int $0x21`调用输出字符的系统调用。

内核被bootsect加载到物理地址0x0处并在保护模式下启动，然后通过`setup_idt`和`setup_gdt`分别完成IDT和GDT设置。setup_idt只是使用一个默认的中断处理程序初始化了34个只允许level 0调用的中断门（主要是方便内核加载）。

下面这段代码预留了IDT空间，`idtr_value`是加载进IDTR寄存器的值。注意limit是最后一个字节的偏移量，因此要减一。

```unixassembly
    .align 8
idt:
    .fill 34,8,0                /* only set 34 idt descriptors, each is 8 bytes */

idtr_value:
    .word 8*34 - 1
    .long idt
```

`setup_idt`使用`default_int_handler`中断处理程序初始化idt，并加载`idtr_value`到IDTR寄存器。默认使用了中断门，运行在level 0 特权级。

```unixassembly
setup_idt:
    lea default_int_handler, %edx   /* handler address */
    mov $0x00080000, %eax           /* select kernel code segment */
    mov %dx, %ax                    /* ax: handler address low 16 bits */
    mov $0x8e00, %dx                /* dpl=0, interrupt gate */
    mov $34, %ecx
    lea idt, %edi
1:
    mov %eax, (%edi)                /* interrupt gate low 16 bits */
    mov %edx, 4(%edi)               /* interrupt gate high 16 bits */
    add $8, %edi
    dec %ecx
    jne 1b
    lidt idtr_value                 /* load idt */
    ret
```

设置完默认中断处理程序后，又重新设置了系统调用0x21的中断处理程序，注意系统调用的中断处理程序的DPL=3，以便允许从level 3应用程序中正常调用。

```unixassembly
    /* init put_char system call */
    lea sys_put_char, %edx
    mov $0x00080000, %eax           /* select kernel code segment */
    mov %dx, %ax                    /* ax: handler address low 16 bits */
    mov $0xee00, %dx                /* dpl=3, allow level 3 app invoke, interrupt gate */
    mov $0x21, %ecx                 /* int 0x21: put a char(al) to the screen */
    lea idt(,%ecx,8), %edi
    mov %eax, (%edi)
    mov %edx, 4(%edi)
```

切换到应用程序是通过模拟一次中断返回完成的，因为直接跳转或者调用会因为保护原因报错。模拟中断返回需要构建一个中断栈，首先将应用程序的堆栈段SS和ESP入栈，然后是EFLAGS，最后是应用程序的代码段CS和返回地址，即应用的入口地址。EFLAGS中的`NT(Nested Task)`置位表示当前执行的任务是嵌套在另一个任务中的，前一个任务的描述符可以从当前任务的TSS中的link字段得到。这里需要清除NT标志位，因为应用程序是我们的第一个任务。

```unixassembly
switch:
    /* switch to level 3 app */
    pushfl                          /* clear NT flag */
    andl $0xffffbfff, (%esp)
    popfl
    /* load current tss */
    mov $0x28, %ax
    ltr %ax
    push $0x23
    pushl $0x8ffff
    pushfl
    push $0x1b
    pushl $main
    iret
```

最后来看一下应用程序中如何调用系统调用，还是需要通过嵌入汇编执行`INT n`指令实现。

```c
void main(void) {
    /*
      print 'A' endless
    */
    char c = 'A';
    int count = 0;
    while (1) {
        asm volatile (
            "\tmov $0x07, %%ah\n"
            "\tint $0x21"
            ::"a"(c)
        );
        if (++count == 80) {
            if (++c > 'z') {
                c = 'A';
            }
            count = 0;
        }
    }
}
```

## 总结

中断服务程序的设置与GDT设置类似，描述符不一样。使用中断实现系统调用时需要特别注意特权级设置，应该和调用方应用程序的CPL相同。