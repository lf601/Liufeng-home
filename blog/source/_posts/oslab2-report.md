---
title: oslab2_report
date: 2023-07-08 14:45:24
tags: os实验报告
categories: 操作系统
banner_img: /img/os_report.jpg
---

### Lab2实验报告

#### 代码

##### 开启trap处理

`head.S`

```assembly
.extern start_kernel

    .section .text.init
    .globl _start
_start:
    
    # set stvec = _traps   
    la t0,_traps
    csrw stvec,t0
    
    # set sie[STIE] = 1
    csrr t0,sie     # STIE为sie寄存器的第6位
    ori t0,t0,1<<5
    csrw sie,t0

    # set first time interrupt
    # 将a0设置成现在的时钟周期加1s(10000000 cycles)
    rdtime a0
    li a1,10000000
    add a0,a0,a1
    mv a1,zero
    mv a2,zero
    mv a3,zero
    mv a4,zero
    mv a5,zero
    mv a6,zero
    mv a7,zero
    ecall

    # set sstatus[SIE] = 1
    csrsi sstatus,1<<1  # SIE为sstatus寄存器的第2位

    # --------
    # Lab1 Code
    # --------
    la sp,boot_stack_top
    call start_kernel

    .section .bss.stack
    .globl boot_stack
boot_stack:
    .space 4096 # <-- change to your stack size

    .globl boot_stack_top
boot_stack_top:.extern start_kernel

    .section .text.init
    .globl _start
_start:
    
    # set stvec = _traps   
    la t0,_traps
    csrw stvec,t0
    
    # set sie[STIE] = 1
    csrr t0,sie
    ori t0,t0,1<<5
    csrw sie,t0

    # set first time interrupt
    rdtime a0
    li a1,10000000
    add a0,a0,a1
    mv a1,zero
    mv a2,zero
    mv a3,zero
    mv a4,zero
    mv a5,zero
    mv a6,zero
    mv a7,zero
    ecall

    # set sstatus[SIE] = 1
    csrsi sstatus,1<<1

    # --------
    # Lab1 Code
    # --------
    la sp,boot_stack_top
    call start_kernel

    .section .bss.stack
    .globl boot_stack
boot_stack:
    .space 4096 # <-- change to your stack size

    .globl boot_stack_top
boot_stack_top:
```

##### 实现上下文切换

`entry.S`

```assembly
    .section .text.entry
    .align 2
    .globl _traps 
_traps:
    # YOUR CODE HERE
    # -----------

        # 1. save 32 registers and sepc to stack

    addi sp,sp,-248
    csrr a0,sepc
    sd a0,0(sp)

    sd x1,8(sp)
    sd x3,16(sp)
    sd x4,24(sp)
    sd x5,32(sp)
    sd x6,40(sp)
    sd x7,48(sp)
    sd x8,56(sp)
    sd x9,64(sp)
    sd x10,72(sp)
    sd x11,80(sp)
    sd x12,88(sp)
    sd x13,96(sp)
    sd x14,104(sp)
    sd x15,112(sp)
    sd x16,120(sp)
    sd x17,128(sp)
    sd x18,136(sp)
    sd x19,144(sp)
    sd x20,152(sp)
    sd x21,160(sp)
    sd x22,168(sp)
    sd x23,176(sp)
    sd x24,184(sp)
    sd x25,192(sp)
    sd x26,200(sp)
    sd x27,208(sp)
    sd x28,216(sp)
    sd x29,224(sp)
    sd x30,232(sp)
    sd x31,240(sp)
    # since x0 == 0 and x2 == sp, so x0 and x2 do not need to be stored
    # -----------

        # 2. call trap_handler
    csrr a0,scause
    csrr a1,sepc
    call trap_handler
    # -----------

        # 3. restore sepc and 32 registers (x2(sp) should be restore last) from stack
    ld a0,0(sp)
    csrw sepc,a0

    ld x1,8(sp)
    ld x3,16(sp)
    ld x4,24(sp)
    ld x5,32(sp)
    ld x6,40(sp)
    ld x7,48(sp)
    ld x8,56(sp)
    ld x9,64(sp)
    ld x10,72(sp)
    ld x11,80(sp)
    ld x12,88(sp)
    ld x13,96(sp)
    ld x14,104(sp)
    ld x15,112(sp)
    ld x16,120(sp)
    ld x17,128(sp)
    ld x18,136(sp)
    ld x19,144(sp)
    ld x20,152(sp)
    ld x21,160(sp)
    ld x22,168(sp)
    ld x23,176(sp)
    ld x24,184(sp)
    ld x25,192(sp)
    ld x26,200(sp)
    ld x27,208(sp)
    ld x28,216(sp)
    ld x29,224(sp)
    ld x30,232(sp)
    ld x31,240(sp)

    addi sp,sp,248
    # -----------

        # 4. return from trap
    sret
    # -----------

```

#####  实现trap处理函数

`trap.c`

```c
// trap.c 
#include "printk.h"

extern void clock_set_next_event();

void trap_handler(unsigned long scause, unsigned long sepc) {
    // 通过 `scause` 判断trap类型
    // 如果是interrupt 判断是否是timer interrupt
    // 如果是timer interrupt 则打印输出相关信息, 并通过 `clock_set_next_event()` 设置下一次时钟中断
    // `clock_set_next_event()` 见 4.5 节
    // 其他interrupt / exception 可以直接忽略

    // YOUR CODE HERE
    if(scause == 0x8000000000000005)
    {
        printk("[S] Supervisor Mode Timer Interrupt \n");
        clock_set_next_event();
    }
}
```

首先判断是否是`interrupt`再判断是否为`timer interrupt`

#####  实现时钟中断相关函数

`clock.c`

```c
// clock.c
#include "sbi.h"
// QEMU中时钟的频率是10MHz, 也就是1秒钟相当于10000000个时钟周期。
unsigned long TIMECLOCK = 10000000;

unsigned long get_cycles() {
    // 编写内联汇编，使用 rdtime 获取 time 寄存器中 (也就是mtime 寄存器 )的值并返回
    // YOUR CODE HERE
    unsigned long _cycles;
    __asm__ volatile("rdtime %0" : "=r"(_cycles));
    return _cycles;
}

void clock_set_next_event() {
    // 下一次 时钟中断 的时间点
    unsigned long next = get_cycles() + TIMECLOCK;

    // 使用 sbi_ecall 来完成对下一次时钟中断的设置
    // YOUR CODE HERE
    sbi_ecall(0,0,next,0,0,0,0,0);
} 
```

##### 编译测试结果

![编译测试结果](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221031103742307.png)

#### 思考题

`mideleg`和`medeleg`合称为Machine Interrupt Delegation Register，为委托寄存器。

通常状态下，RISC-V架构下的所有trap都跳转到M-Mode进行处理。为了提高性能，RISC-V支持将低权限mode产生的trap委托给对应mode处理，其中涉及到`mideleg`和`medeleg`两个寄存器，其中`mideleg`寄存器控制将哪些中断委托给S-Mode处理。`mideleg`寄存器的结构与`mip`类似，`mideleg`的值为`0x0000000000000222`表示1、5、9位分别为1，分别对应SSIP、STIP、SEIP，表示这3种类型中断在S-Mode下委托给S-Mode处理。