---
title: oslab3_report
date: 2023-07-08 14:45:28
tags: os实验报告
categories: 操作系统
banner_img: /img/os_report.jpg
---

### Lab3实验报告

#### 代码

##### 线程初始化

```c++
void task_init() {
    // 1. 调用 kalloc() 为 idle 分配一个物理页
    idle = (struct task_struct*)kalloc();

    // 2. 设置 state 为 TASK_RUNNING;
    idle->state = TASK_RUNNING;
    
    // 3. 由于 idle 不参与调度 可以将其 counter / priority 设置为 0
    idle->counter = 0;
    idle->priority = 0;

    // 4. 设置 idle 的 pid 为 0
    idle->pid = 0;

    // 5. 将 current 和 task[0] 指向 idle
    current = idle;
    task[0] = idle;

    

    int i = 1;
    // 1. 参考 idle 的设置, 为 task[1] ~ task[NR_TASKS - 1] 进行初始化
    for(;i<NR_TASKS;i++)
    {
        task[i]=(struct task_struct*)kalloc();
        // 2. 其中每个线程的 state 为 TASK_RUNNING, counter 为 0, priority 使用 rand() 来设置, pid 为该线程在线程数组中的下标。
        task[i]->state = TASK_RUNNING;
        task[i]->counter = 0;
        task[i]->priority = rand()%(PRIORITY_MAX-PRIORITY_MIN+1)+PRIORITY_MIN;
        task[i]->pid = i;
        // 3. 为 task[1] ~ task[NR_TASKS - 1] 设置 `thread_struct` 中的 `ra` 和 `sp`,
        // 4. 其中 `ra` 设置为 __dummy （见 4.3.2）的地址,  `sp` 设置为 该线程申请的物理页的高地址
        task[i]->thread.ra = (uint64)__dummy;
        task[i]->thread.sp = (uint64)task[i]+PGSIZE;
    }

    printk("...proc_init done!\n");
}
```



#####  __dummy

```riscv
__dummy:
    la t0,dummy
    csrw sepc,t0
    sret
```

__dummy是context switch时调用的代码，负责将sepc设置为dummy的地址，在退出中断时执行dummy函数。



#####  实现线程切换

`void switch_to(struct task*next)`

```c++
void switch_to(struct task_struct* next) {
    if(current!=next)
    {
        #ifdef SJF
            printk("switch to [PID = %d COUNTER = %d]\n",next->pid,next->counter);
        #endif
        #ifdef PRIORITY
            printk("switch to [PID = %d PRIORITY = %d COUNTER = %d]\n",next->pid,next->priority,next->counter);
        #endif
        struct task_struct* prev = current;
        current = next;
        __switch_to(prev,next);
    }
    else return ;
}
```

调用`__switch_to`实现线程切换



`__switch_to`

```riscv
__switch_to:
    # save state to prev process
    addi t0,a0,40
    sd ra,0(t0)
    sd sp,8(t0)

    addi t0,a0,56
    sd s0,0(t0)
    sd s1,8(t0)
    sd s2,16(t0)
    sd s3,24(t0)
    sd s4,32(t0)
    sd s5,40(t0)
    sd s6,48(t0)
    sd s7,56(t0)
    sd s8,64(t0)
    sd s9,72(t0)
    sd s10,80(t0)
    sd s11,88(t0)

    # restore state from next process
    addi t0,a1,40
    ld ra,0(t0)
    ld sp,8(t0)

    addi t0,a1,56
    ld s0,0(t0)
    ld s1,8(t0)
    ld s2,16(t0)
    ld s3,24(t0)
    ld s4,32(t0)
    ld s5,40(t0)
    ld s6,48(t0)
    ld s7,56(t0)
    ld s8,64(t0)
    ld s9,72(t0)
    ld s10,80(t0)
    ld s11,88(t0)

    ret
```

保存当前进程与函数调用相关的寄存器并将下一个进程保存的相关寄存器的值load到寄存器中。



#####  实现调度入口函数

```c++
void do_timer()
{
    // 1. 如果当前线程是 idle 线程 直接进行调度
    // 2. 如果当前线程不是 idle 对当前线程的运行剩余时间减1 若剩余时间仍然大于0 则直接返回 否则进行调度
    if(current==idle)
    {
        schedule();
    }
    else
    {
        current->counter--;
        if(current->counter>0) return ;
        else schedule();
    }
}
```



#####  实现线程调度

```c++
void schedule(void)
{
    #ifdef SJF
        SJF_schedule();
    #endif
    #ifdef PRIORITY
        PRIORITY_schedule();
    #endif
}
```

######  短作业优先调度算法

```c++
void SJF_schedule()
{
    struct task_struct* next = current;
    uint64 flag = 0,min_counter = COUNTER_MAX + 1;
    for(int i=1;i<NR_TASKS;i++)
    {
        if(task[i]->counter!=0)
        {
            flag = 1;
            break;
        }
    }
    if(!flag)
    {
        printk("\n");
        for(int i=1;i<NR_TASKS;i++)
        {
            task[i]->counter=rand()%(COUNTER_MAX-COUNTER_MIN+1)+COUNTER_MIN;
            printk("SET [PID = %d COUNTER = %d]\n",task[i]->pid,task[i]->counter);
        }
        printk("\n");
    }

    for(int i=1;i<NR_TASKS;i++)
    {
        if(task[i]->state!=TASK_RUNNING||task[i]->counter==0)continue;
        if(task[i]->counter<min_counter)
        {
            min_counter = task[i]->counter;
            next = task[i];
        }
    }
    switch_to(next);
    
}
```



###### 优先级调度算法

```c++
void PRIORITY_schedule()
{
    struct task_struct* next = current;
    uint64 flag = 0,max_priority = PRIORITY_MIN - 1;
    for(int i=1;i<NR_TASKS;i++)
    {
        if(task[i]->counter!=0)
        {
            flag = 1;
            break;
        }
    }
    if(!flag)
    {
        printk("\n");
        for(int i=1;i<NR_TASKS;i++)
        {
            task[i]->counter=rand()%(COUNTER_MAX-COUNTER_MIN+1)+COUNTER_MIN;
            printk("SET [PID = %d PRIORITY = %d COUNTER = %d]\n",task[i]->pid,task[i]->priority,task[i]->counter);
        }
        printk("\n");
    }

    for(int i=1;i<NR_TASKS;i++)
    {
        if(task[i]->state!=TASK_RUNNING||task[i]->counter==0)continue;
        if(task[i]->priority>max_priority)
        {
            max_priority=task[i]->priority;
            next=task[i];
        }
    }
    switch_to(next);
    
}
```

两个算法思路类似，先遍历task->counter，如果全为0，则使用rand()函数对每一个进程的运行剩余时间重新赋值。此后从头开始遍历，寻找剩余时间最短的或优先级最高的进程，将其赋值给next，利用`switch_to()`进行进程切换。



#### 测试结果

为方便截图，在以下测试结果中将`NR_TASKS`改为$1+5$。

短作业优先调度算法

![短作业优先调度](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221116185740338.png)

优先级调度算法

![优先级调度](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221116190209083.png)



### 思考题

1. 中断的上下文需要保存所有的寄存器，`context_switch`属于函数调用，遵循函数调用的规则，所以只需要保存s0-11 12个callee saved register和ra，sp共14个寄存器。

2. 在刚进入`__switch_to`时，ra存放的为`switch_to()`函数中调用`__switch_to`的地址

   ![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221116191534001.png)

当执行完`ld ra,0(t0)`后，`next->thread.ra`被赋值给ra，ra存储的为`__dummy`的地址

