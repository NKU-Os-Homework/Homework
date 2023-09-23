# 练习
## 练习1
```
#include <mmu.h>
#include <memlayout.h>

    .section .text,"ax",%progbits
    .globl kern_entry
kern_entry:
    la sp, bootstacktop

    tail kern_init

.section .data
    # .align 2^12
    .align PGSHIFT
    .global bootstack
bootstack:
    .space KSTACKSIZE
    .global bootstacktop
bootstacktop:
```
1. la sp, bootstacktop完成了将bootstacktop标签处的地址值赋给sp的操作，该汇编语句的作用是初始化操作系统内核栈

2. tail kern_init用于实现函数调用，这是一种特殊的函数调用，称为尾调用（tail call）。尾调用是一种优化技术，它在调用一个函数之后，不需要在返回时执行额外的操作，而是直接跳转到被调用函数的入口点，这有助于减少栈空间的使用，同时可以确保内核不会返回到引导加载程序或操作系统之外。该语句用于将控制权转移给真正的入口点kern_init。 

## 练习2
1.实现过程：
 + 在init.c中初始化时钟中断clock_init();
 + 在trap.c中首先设置下次时钟中断- clock_set_next_event()
 + 计数器（ticks）加一
 + 当计数器加到100的时候，我们会输出一个`100ticks`表示我们触发了100次时钟中断，同时打印次数（num）加一
 + 判断打印次数，当打印次数为10时，调用<sbi.h>中的关机函数关机

2.中断处理流程：
 + 在init.c中初始化时钟中断clock_init();
 + 触发时钟中断后，stvec寄存器中保存了中断异常处理函数的入口地址
 + 在trapentry.S中保存寄存器值，然后将控制权交给真正处理函数入口点
 + 在trap.c中根据中断异常处理类型调用相关函数进行处理
### challenge1
1. ucore中断异常的处理流程：当时钟中断产生时，操作系统会保存pc中触发中断异常的指令地址，同时将pc值设为stvec寄存器的值（中断处理程序的入口点），跳转到`kern/trap/trapentry.S`的`__alltraps`标记，保存当前执行流的上下文，并通过函数调用，切换为`kern/trap/trap.c`的中断处理函数`trap()`的上下文，进入`trap()`的执行流。
切换前的上下文作为一个结构体，传递给`trap()`作为函数参数 -> `kern/trap/trap.c`按照中断类型进行分发(`trap_dispatch(), interrupt_handler()`)->执行时钟中断对应的处理语句，累加计数器，设置下一次时钟中断->完成处理，返回到`kern/trap/trapentry.S`->恢复原先的上下文，中断处理结束。
<br>

2. mov a0，sp的目的：该语句其实是在为下面这条汇编调用c函数的汇编语句jal trap，准备执行环境，a0寄存器在riscv体系结构中是用来传递函数调用参数的，将sp值赋给a0，其实就是将保存切换前的上下文的栈地址传递给函数void trap(struct trapframe *tf) { trap_dispatch(tf);}，将栈指针的值传递给异常处理程序，以便它可以访问当前的栈帧和栈上的数据。
 ```
 __alltraps:
    SAVE_ALL

    move  a0, sp
    jal trap
    # sp should be the same as before "jal trap"
 ```

3. SAVE_ALL中寄寄存器保存在栈中的位置由当前栈顶指针sp确定
 ```
 addi sp, sp, -36 * REGBYTES
 # save x registers
 STORE x0, 0*REGBYTES(sp)
 ```

4. 不需要，保存寄存器的需求取决于中断的类型和处理程序的具体需求，不同类型的中断可能需要保存不同的寄存器，保存和恢复所有寄存器可能会引入额外的开销，包括内存访问和指令执行。如果中断处理程序可以在不保存所有寄存器的情况下正常工作，那么可以减小中断处理的开销，提高性能。一般来说，中断和异常可以分为两种类型：可屏蔽中断（Maskable Interrupts）和非可屏蔽中断和异常（Non-Maskable Interrupts and Exceptions）。对于可屏蔽中断，可以选择在进入中断处理程序之前保存和恢复一部分寄存器，而不是所有寄存器。对于非可屏蔽中断和异常，通常会保存所有寄存器，因为无法信任中断处理程序的行为，需要确保所有寄存器状态都能够被完全恢复。

### challenge2

1. csrw sscratch, sp；csrrw s0, sscratch, x0实现了什么操作：csrw sscratch, sp将sp寄存器中的值赋给sscratch，csrrw s0, sscratch, x0将sscratch的当前值写入目标寄存器`s0`，并将CSR `sscratch` 的值设置为零。
目的：保存之前写到sscratch里的sp的值
 ```
     .macro SAVE_ALL

    csrw sscratch, sp

    addi sp, sp, -36 * REGBYTES
    ......
    # get sr, epc, badvaddr, cause
    # Set sscratch register to 0, so that if a recursive exception
    # occurs, the exception vector knows it came from the kernel
    csrrw s0, sscratch, x0
--------------------------------------------------------
    .macro RESTORE_ALL

    LOAD s1, 32*REGBYTES(sp)
    LOAD s2, 33*REGBYTES(sp)

    csrw sstatus, s1
    csrw sepc, s2

 ```

2. 这主要是因为badvaddr寄存器和cause寄存器中保存的分别是出错的地址以及出错的原因，在处理中断时可以利用这些信息，当我们处理完这个中断的时候，也就不需要这两个寄存器中保存的值，所以可以不用恢复这两个寄存器。而当由新的中断发生时，系统会自动设置这两个寄存器的值

### challenge3

1. 出现中断时，中断返回地址 mepc 的值被更新为下一条尚未执行的指令

 出现异常时，中断返回地址 mepc 的值被更新为当前发生异常的指令 PC，在异常处理程序中软件改变 mepc 指向下一条指令，由于现在 ecall/ebreak（或 c.ebreak）是 4（或 2）字节指令，因此改写设定 mepc=mepc+4（或+2）即可。

2. 代码见编程部分

# 操作系统知识点与原理

 + riscv中断相关知识
 + 中断前后如何进行上下文环境的保存与恢复
 + + 对应操作系统知识点：处理中断和异常时的上下文切换
 + ucore如何处理最简单的断点中断和时钟中断

# 该实验未涉及知识点
 + ucore为riscv架构，和x86有所不同。在x86架构计算机中，操作系统处理中断和异常前会先初始化idt，即中断异常处理向量表，来保存中断和异常处理对应的函数。
