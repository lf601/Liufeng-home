---
title: oslab5_report
date: 2023-07-08 14:45:38
tags: os实验报告
categories: 操作系统
banner_img: /img/os_report.jpg
---

## lab5实验报告

### 1.准备工程

* 此次试验根据lab4所实现代码进行
* 需要修改 `vmlinux.lds.S`，将用户态程序 `uapp` 加载至 `.data` 段。按如下修改：

```
.data : ALIGN(0x1000){
        _sdata = .;

        *(.sdata .sdata*)
        *(.data .data.*)

        _edata = .;

        . = ALIGN(0x1000);
        uapp_start = .;
        *(.uapp .uapp*)
        uapp_end = .;
        . = ALIGN(0x1000);

    } >ramv AT>ram
```

* 需要修改 `defs.h`，在 `defs.h` `添加` 如下内容：

```c
#define USER_START (0x0000000000000000) // user space start virtual address
#define USER_END   (0x0000004000000000) // user space end virtual address
```

* 从 `repo` 同步以下文件夹: `user`， `Makefile`。并按照以下步骤将这些文件正确放置。

```
.
├── arch
│   └── riscv
│       └── Makefile
└── user
    ├── Makefile
    ├── getpid.c
    ├── link.lds
    ├── printf.c
    ├── start.S
    ├── stddef.h
    ├── stdio.h
    ├── syscall.h
    └── uapp.S
```

* 修改**根目录**下的 Makefile, 将 `user` 纳入工程管理。
* 在根目录下 `make` 会生成 `user/uapp.o` `user/uapp.elf` `user/uapp.bin`。 通过 `objdump` 我们可以看到 uapp 使用 ecall 来调用 SYSCALL (在 U-Mode 下使用 ecall 会触发environment-call-from-U-mode异常)。从而将控制权交给处在 S-Mode 的 OS， 由内核来处理相关异常。

### 2.创建用户态进程

* 本次实验只需要创建 4 个用户态进程，修改 `proc.h` 中的 `NR_TASKS` 为5。
* 由于创建用户态进程要对 `sepc` `sstatus` `sscratch` 做设置，我们将其加入 `thread_struct` 中。
* 由于多个用户态进程需要保证相对隔离，因此不可以共用页表。我们为每个用户态进程都创建一个页表。修改 `task_struct` 如下。

```c
// proc.h 

typedef unsigned long* pagetable_t;

struct thread_struct {
    uint64_t ra;
    uint64_t sp;                     
    uint64_t s[12];

    uint64_t sepc, sstatus, sscratch; 
};

struct task_struct {
    struct thread_info* thread_info;
    uint64_t state;
    uint64_t counter;
    uint64_t priority;
    uint64_t pid;

    struct thread_struct thread;

    pagetable_t pgd;
    uint64 kernel_sp,user_sp;
};
```

* 修改task_init
  * 对每个用户态进程，其拥有两个 stack： `U-Mode Stack` 以及 `S-Mode Stack`， 其中 `S-Mode Stack` 在 `lab3` 中我们已经设置好了。我们可以通过 `kalloc` 接口申请一个空的页面来作为 `U-Mode Stack`。
  * 为每个用户态进程创建自己的页表 并将 `uapp` 所在页面，以及 `U-Mode Stack` 做相应的映射，同时为了避免 `U-Mode` 和 `S-Mode` 切换的时候切换页表，我们也将内核页表 （ `swapper_pg_dir` ） 复制到每个进程的页表中。
  * 对每个用户态进程我们需要将 `sepc` 修改为 `USER_START`， 设置 `sstatus` 中的 `SPP` （ 使得 sret 返回至 U-Mode ）， `SPIE` （ sret 之后开启中断 ）， `SUM` （ S-Mode 可以访问 User 页面 ）， `sscratch` 设置为 `U-Mode` 的 sp，其值为 `USER_END` （即  `U-Mode Stack` 被放置在 `user space` 的最后一个页面）。

```c
void task_init() {
    //删去了前面实验的代码,只保留了本次实验新增代码
    for(;i<NR_TASKS;i++)
    {
        //通过kalloc接口申请一个空页面作为U-Mode Stack
        task[i]->kernel_sp = (unsigned long)(task[i])+PGSIZE;
        task[i]->user_sp = kalloc();

        //将内核页表映射到每个进程的pgtbl中
        pagetable_t pgtbl = (pagetable_t)kalloc();
        memcpy(pgtbl,swapper_pg_dir,PGSIZE);

        //映射uapp所在页面，权限设置为U|X|W|R|V
        uint64 va = USER_START;
        uint64 pa = (uint64)uapp_start-PA2VA_OFFSET;
        create_mapping(pgtbl,va,pa,(uint64)(uapp_end)-(uint64)(uapp_start),31);

        //映射U-Mode Stack，权限设置为 U|-|W|R|V
        va = USER_END-PGSIZE;
        pa = task[i]->user_sp-PA2VA_OFFSET;
        create_mapping(pgtbl,va,pa,PGSIZE,23);

        uint64 satp = csr_read(satp);
        satp = (satp>>44)<<44;
        satp |= ((uint64)pgtbl-PA2VA_OFFSET)>>12;
        task[i]->pgd = satp;
        //设置sepc,sstatus,sscratch
        task[i]->thread.sepc = USER_START;
        task[i]->thread.sscratch = USER_END;
        uint64 sstatus = csr_read(sstatus);
        sstatus &= ~(1<<8); // SPP置为0
        sstatus |= 1<<5;    // SPIE置为1
        sstatus |= 1<<18;   // sscratch置为1
        task[i]->thread.sstatus = sstatus;
    }

    const uint64 OffsetOfThreadInTask = offsetof(struct task_struct, thread);
    const uint64 OffsetOfRaInTask = OffsetOfThreadInTask+offsetof(struct thread_struct,ra);
    const uint64 OffsetOfSpInTask = OffsetOfThreadInTask+offsetof(struct thread_struct, sp);
    const uint64 OffsetOfSInTask = OffsetOfThreadInTask+offsetof(struct thread_struct, s);
    const uint64 OffsetOfSepcInTask = OffsetOfThreadInTask+offsetof(struct thread_struct, sepc);

    //printk("OffsetOfRaInTask = %d\n", OffsetOfRaInTask);
    //printk("OffsetOfSpInTask = %d\n", OffsetOfSpInTask);
    //printk("OffsetOfSInTask = %d\n", OffsetOfSInTask);
    //printk("OffsetOfSepcInTask = %d\n",OffsetOfSepcInTask);
    
    printk("...proc_init done!\n");
}
```

* 修改 __switch_to， 需要加入 保存/恢复 `sepc` `sstatus` `sscratch` 以及 切换页表的逻辑。

添加了以下代码

```
addi t0,a0,152
csrr t1,sepc
sd t1,0(t0)
csrr t1,sstatus
sd t1,8(t0)
csrr t1,sscratch
sd t1,16(t0)
csrr t1,satp
sd t1,24(t0)

addi t0,a1,152
ld t1,0(t0)
csrw sepc,t1
ld t1,8(t0)
csrw sstatus,t1
ld t1,16(t0)
csrw sscratch,t1
ld t1,24(t0)
csrw satp, t1
```

### 3.修改中断入口/返回逻辑 ( _trap ) 以及中断处理函数 （ trap_handler ）

* 与 ARM 架构不同的是，RISC-V 中只有一个栈指针寄存器( sp )，因此需要我们来完成用户栈与内核栈的切换。
* 由于我们的用户态进程运行在 `U-Mode` 下， 使用的运行栈也是 `U-Mode Stack`， 因此当触发异常时， 我们首先要对栈进行切换 （ `U-Mode Stack` -> `S-Mode Stack` ）。同理 让我们完成了异常处理， 从 `S-Mode` 返回至 `U-Mode`， 也需要进行栈切换 （ `S-Mode Stack` -> `U-Mode Stack` ）。
* 修改 `__dummy`。在 **4.2** 中 我们初始化时， `thread_struct.sp` 保存了 `S-Mode sp`， `thread_struct.sscratch` 保存了 `U-Mode sp`， 因此在 `S-Mode -> U->Mode` 的时候，我们只需要交换对应的寄存器的值即可。

```
__dummy:
    STACK_CHANGE t0
    # la t0,dummy
    # csrw sepc,t0
    sret
```

* 修改 `_trap` 。同理 在 `_trap` 的首尾我们都需要做类似的操作。**注意如果是 内核线程( 没有 U-Mode Stack ) 触发了异常，则不需要进行切换。（内核线程的 sp 永远指向的 S-Mode Stack， sscratch 为 0）**

```
_traps:
    # YOUR CODE HERE
    # -----------
    csrr t0,sscratch
    beq t0,zero,_traps_switch
    STACK_CHANGE t0
_traps_switch:
	#details omitted
    
    csrr t0,sscratch
    beq t0,zero,_traps_end
    STACK_CHANGE t0
_traps_end:
```

* `uapp` 使用 `ecall` 会产生 `ECALL_FROM_U_MODE` **exception**。因此我们需要在 `trap_handler` 里面进行捕获。修改 `trap_handler` 如下：

```c
void trap_handler(uint64_t scause, uint64_t sepc, struct pt_regs *regs) {
	...
}
```

regs参数用来传递`sepc`和`a0~a7`寄存器的值，下面补充`struct pt_regs`的定义和`trap_handler`的实现。

```c
struct pt_regs{
    unsigned long sepc;
    unsigned long x[30];
};
```

```c
void trap_handler(unsigned long scause, unsigned long sepc, struct pt_regs *regs) {
    // YOUR CODE HERE
    if(scause == 0x8000000000000005)
    {
        //printk("[S] Supervisor Mode Timer Interrupt \n");
        clock_set_next_event();
        do_timer();
    }
    else if(scause == 0x8)
    {
        //printk("syscall\n");
        syscall(regs);
    }
    else
    {
        printk("unhandled trap:%lx\n",scause);
        printk("sepc:%lx\n",sepc);
        printk("regs:%lx\n",&regs);
        while(1);
    }
}
```

### 4.添加系统调用

* 本次实验要求的系统调用函数原型以及具体功能如下：
  * 64 号系统调用 `sys_write(unsigned int fd, const char* buf, size_t count)` 该调用将用户态传递的字符串打印到屏幕上，此处fd为标准输出（1），buf为用户需要打印的起始地址，count为字符串长度，返回打印的字符数。( 具体见 user/printf.c )
  * 172 号系统调用 `sys_getpid()` 该调用从current中获取当前的pid放入a0中返回，无参数。（ 具体见 user/getpid.c ）
* 增加 `syscall.c` `syscall.h` 文件， 并在其中实现 `getpid` 以及 `write` 逻辑。
* 系统调用的返回参数放置在 `a0` 中 (不可以直接修改寄存器， 应该修改 regs 中保存的内容)。
* 针对系统调用这一类异常， 我们需要手动将 `sepc + 4` （ `sepc` 记录的是触发异常的指令地址， 由于系统调用这类异常处理完成之后， 我们应该继续执行后续的指令，因此需要我们手动修改 `spec` 的地址，使得 `sret` 之后 程序继续执行）。

```c
//syscall.h
#pragma once

#include "defs.h"
#define SYS_WRITE   64
#define SYS_GETPID  172

struct pt_regs{
    unsigned long sepc;
    unsigned long x[30];
};

void syscall( struct pt_regs* regs );
```

```c
//syscall.c
#include "syscall.h"
#include "proc.h"
#include "printk.h"

extern struct task_struct *current;

void syscall(struct pt_regs* regs)
{
    /*
    for(int i=0;i<30;i++)
        printk("reg[%d]:%d\n",i,regs->x[i]);
    while(1);
    */
    if (regs->x[15] == SYS_WRITE)
    {
        char *str = (char*)regs->x[9];
        uint64 len = regs->x[10];
        str[len] = '\0';
        printk(str);
    }
    else if (regs->x[15] == SYS_GETPID)
    {
        regs->x[8] = current->pid;
    }
    regs->sepc += 0x4;
    return;
}
```

### 5.修改head.S以及start_kernel

* 之前 lab 中， 在 OS boot 之后，我们需要等待一个时间片，才会进行调度。我们现在更改为 OS boot 完成之后立即调度 uapp 运行。
* 在 start_kernel 中调用 schedule() 注意放置在 test() 之前。
* 将 head.S 中 enable interrupt sstatus.SIE 逻辑注释，确保 schedule 过程不受中断影响。

```
# set sstatus[SIE] = 1
# csrsi sstatus,1<<1
```

```c
int start_kernel() {
    
    printk("[S-MODE]%d Hello RISC-V\n",2022);
    //printk("idle process is running\n");
    schedule();
    test(); // DO NOT DELETE !!!
    
	return 0;
}
```



### 编译与测试

![编译运行结果](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221228162941403.png)



### 思考题

1. 我们在实验中使用的用户态线程和内核态线程的对应关系是怎样的？（一对一，一对多，多对一还是多对多）

   多对一

2. 为什么 Phdr 中，`p_filesz` 和 `p_memsz` 是不一样大的？

   `p_filesz`对应文件中该段的大小，`p_memsz`对应该段的内存中大小，可加载段可能包含`.bss`节，该节包含未初始化的数据。将此数据存储在磁盘上会很浪费，因此仅在ELF文件加载到内存后才占用空间。

3. 为什么多个进程的栈虚拟地址可以是相同的？用户有没有常规的方法知道自己栈所在的物理地址？

   因为虚拟地址是通过`create_mapping`函数创建的与物理地址的映射，不同进程对应的所在的物理地址是不同的，所以栈虚拟地址相同也没有关系。可以在`task_init`函数中将不同进程的栈所在的物理地址print出来。