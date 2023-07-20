---
title: oslab4_report
date: 2023-07-08 14:45:33
tags: os实验报告
categories: 操作系统
banner_img: /img/os_report.jpg
---

### lab4实验报告

#### 开启虚拟内存映射

在lab4中，开启虚拟内存映射被分为了2步，分别为`setup_vm`和`setup_vm_final`，具体实现如下

##### `setup_vm`的实现

将 0x80000000 开始的 1GB 区域进行两次映射，其中一次是等值映射 ( PA == VA ) ，另一次是将其映射至高地址 ( PA + PV2VA_OFFSET == VA )

###### 相关宏定义

```c++
#define VPN2(va) ( (va>>30) & 0x1ff )
#define VPN1(va) ( (va>>21) & 0x1ff )
#define VPN0(va) ( (va>>12) & 0x1ff )
```



`linear_mapping`函数

`linear_mapping`仿照`create_mapping`函数创建，用于创建单级页表线性映射关系，其中由于在`setup_vm`中权限位都为15，所以只需要传递3个参数。

观察SV39模式物理地址的布局

![SV39Mode物理地址布局](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221207121050653.png)

1G为$2^{30}$，所以使用PPN[2]作为物理地址的页表好，后30位作为偏移量

```c++
void linear_mapping( unsigned long *pgtbl, unsigned long va, unsigned long pa )
{
    int index = VPN2(va);
    unsigned long PPN = (pa>>30) & 0x3ffffff;
    pgtbl[index] = (15) | PPN<<28;

    return ;
}
```



`setup_vm`函数

```c++
void setup_vm(void) {
    /* 
    1. 由于是进行 1GB 的映射 这里不需要使用多级页表 
    2. 将 va 的 64bit 作为如下划分： | high bit | 9 bit | 30 bit |
        high bit 可以忽略
        中间9 bit 作为 early_pgtbl 的 index
        低 30 bit 作为 页内偏移 这里注意到 30 = 9 + 9 + 12， 即我们只使用根页表， 根页表的每个 entry 都对应 1GB 的区域。 
    3. Page Table Entry 的权限 V | R | W | X 位设置为 1
    */
   unsigned long pa = PHY_START;
   unsigned long va = PHY_START;
   int index;
   unsigned long PPN;
   memset(early_pgtbl,0x0,PGSIZE);

   linear_mapping(early_pgtbl,va,pa);

   va = VM_START;
   linear_mapping(early_pgtbl,va,pa);
   
   printk("...setup_vm done!\n");
}
```



在完成`setup_vm`函数之后，需要对`head.S`进行修改

在汇编代码起始处调用`setup_vm`函数和`relocate`，其中`relote`汇编代码段执行对`satp`寄存器的修改，具体实现如下

```riscv
relocate:

    # set ra = ra + PA2VA_OFFSET
    # set sp = sp + PA2VA_OFFSET (If you have set the sp before)
    li t0, PA2VA_OFFSET
    add ra, ra, t0
    add sp, sp, t0

    # set satp with early_pgtbl
    # satp[PPN] <= early_pgtbl / 2^12 (根页表地址/4KB 即为根页表在物理页*上的页号)
    la t0, early_pgtbl
    srli t0, t0, 12
    # satp[ASID] <= 0
    li t1, 0x00000fffffffffff
    and t0, t0, t1
    # satp[MODE] <= 8 意为Sv39模式
    li t1, 0x8000000000000000
    or t0, t0, t1
    csrw satp, t0

    # flush tlb
    sfence.vma zero, zero

    # flush icache
    fence.i

    ret
```



自此已经完成了虚拟地址的开启

#####  `setup_vm_final`的实现

由于`setup_vm_final`中需要申请页面的接口， 应该在其之前完成内存管理初始化，需要修改 mm.c 中的代码，mm.c 中初始化的函数接收的起始结束地址需要调整为虚拟地址``

```c++
void mm_init(void) {
    kfreerange(_ekernel, (char *)PHY_END+PA2VA_OFFSET);
    printk("...mm_init done!\n");
}
```



`create_mapping()`函数用于创建多级页表映射关系，传递了4个参数，分别为根页表地址，待映射的物理地址和虚拟地址，权限。

在函数开始，声明了3个函数中将会使用的数组，如下

* `unsigned int vpn[3];`*存储三、二、一级页表中所求PTE的偏移量*
* `unsigned long* v_pgtbl[3];`*存储三、二、一级页表的虚拟地址*
* `unsigned long pte[3];`*存储三、二、一级页表中所求PTE的值*

`create_mapping()`函数中，每次循环映射一页4KB的物理地址到虚拟地址，具体流程如下

1. 将根页表的基地址作为`v_pgtbl[2]`，根据`v_pgtbl[2]`和`vpn[2]`得到 PTE 的内容 `pte[2]`
2. 若 PTE 的有效位 V 为 0，则申请一块内存空间作为新的二级页表，并将新建的二级页表的地址存放到 PTE 中，并将有效位 V 置 1
3. 根据 PTE 的内容求出二级页表的虚拟地址，在二级页表中用同样的方法新建一个一级页表或求得一级页表的地址
4. 在一级页表中求得 PTE 的地址，将物理地址存入 PTE，将有效位 V 置 1，根据`perm`改写 RWX 权限位

```c++
/* 创建多级页表映射关系 */
void create_mapping(uint64 *pgtbl, uint64 va, uint64 pa, uint64 sz, int perm) 
{
/*
    pgtbl 为根⻚表的基地址
    va, pa 为需要映射的段的开头的虚拟地址、物理地址
    sz 为需要映射的段的⼤⼩
    perm 为映射的读写权限，可设置不同section所在⻚的属性，完成对不同section的保护
    创建多级⻚表的时候可以使⽤ kalloc() 来获取⼀⻚作为⻚表⽬录
    可以使⽤ V bit 来判断⻚表项是否存在
*/
    unsigned int vpn[3]; // 存储三、二、一级页表中所求PTE的偏移量
    unsigned long* v_pgtbl[3]; // 存储三、二、一级页表的虚拟地址
    unsigned long pte[3]; // 存储三、二、一级页表中所求PTE的值
    unsigned long* page;

    unsigned long end = va + sz;

    while( va < end )
    {
        v_pgtbl[2] = pgtbl;
        vpn[2] = VPN2(va);
        pte[2] = v_pgtbl[2][vpn[2]];
        if(!(pte[2]&1))
        {
            page = (unsigned long*)kalloc();
            pte[2] = ((((unsigned long)page-PA2VA_OFFSET) >> 12) << 10) | 1;
            v_pgtbl[2][vpn[2]] = pte[2];
        }

        v_pgtbl[1] = (unsigned long*)(((pte[2] >> 10) << 12)+PA2VA_OFFSET);
        vpn[1] = VPN1(va);
        pte[1] = v_pgtbl[1][vpn[1]];
        if(!(pte[1]&1))
        {
            page = (unsigned long*)kalloc();
            pte[1] = ((((unsigned long)page-PA2VA_OFFSET) >> 12) << 10) | 1;
            v_pgtbl[1][vpn[1]] = pte[1];
        }

        v_pgtbl[0] = (unsigned long*)(((pte[1] >> 10) << 12)+PA2VA_OFFSET);
        vpn[0] = VPN0(va);
        pte[0] = (perm & 15) | ((pa >> 12) << 10);
        v_pgtbl[0][vpn[0]] = pte[0];

        va += PGSIZE;
        pa += PGSIZE;
    }
    return ;
}
```



`setup_vm_final`函数

在`setup_vm_final()`中调用`create_mapping()`函数，对128MB物理内存进行映射，其中`.text`段的权限设置为 X | - | R | V，`.rodata`段的权限设置为 - | - | R | V，其余内存的权限设置为 - | W | R | V。为了确定各段的起始地址，从`vmlinux.lds`引入如下外部变量

```c++
extern char _stext[];
extern char _srodata[];
extern char _sdata[];
```

```c++
void setup_vm_final(void) {
    memset(swapper_pg_dir, 0x0, PGSIZE);

    unsigned long va = VM_START+OPENSBI_SIZE;
    unsigned long pa = PHY_START+OPENSBI_SIZE;
    // No OpenSBI mapping required

    // mapping kernel text X|-|R|V
    create_mapping(swapper_pg_dir,va,pa,(unsigned long)_srodata-(unsigned long)_stext,11);

    // mapping kernel rodata -|-|R|V
    va += (unsigned long)_srodata-(unsigned long)_stext;
    pa += (unsigned long)_srodata-(unsigned long)_stext;
    create_mapping(swapper_pg_dir,va,pa,(unsigned long)_sdata-(unsigned long)_srodata,3);

    // mapping other memory -|W|R|V
    va += (unsigned long)_sdata-(unsigned long)_srodata;
    pa += (unsigned long)_sdata-(unsigned long)_srodata;
    create_mapping(swapper_pg_dir,va,pa,0x8000000-((unsigned long)_sdata-(unsigned long)_stext),7);

    // set satp with swapper_pg_dir
    unsigned long PG_DIR = (unsigned long)swapper_pg_dir-PA2VA_OFFSET;
    asm volatile (
        "li t0, 8\n"
        "slli t0, t0, 60\n"
        "mv t1, %[PG_DIR]\n"
        "srli t1, t1, 12\n"
        "add t0, t0, t1\n"
        "csrw satp, t0"
        :
        :[PG_DIR]"r"(PG_DIR)
        :"memory"
    );

    // flush TLB
    asm volatile("sfence.vma zero, zero");

    // flush icache
    asm volatile("fence.i");
    return;
}
```



实现完成后，在`head.S`中适当的位置调用`setup_vm_final`，`head.S`补全后的逻辑如下(只显示_start段)

```riscv
_start:

    la sp,boot_stack_top

    call setup_vm
    call relocate

    call mm_init

    call setup_vm_final

    call task_init
    
    # 以下省略设置时间中断部分
    
    call start_kernel
```



同时，为了输出效果与github上一致，修改了部分lab3中代码，在此不做呈现



#### 编译与测试

![编译测试结果](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221207124554542.png)



###思考题

1. 验证 `.text`, `.rodata` 段的属性是否成功设置，给出截图。

   程序能正常运行，说明`.text`的X位设置成功

   在`main.c`中添加代码尝试对段进行读操作

   ```c++
   extern char _stext[];
   extern char _srodata[];
   
   printk("_stext = %ld\n", *_stext);
   printk("_srodata = %ld\n", *_srodata);
   ```

   运行结果如下

   ![验证读操作权限](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221207125428762.png)

   说明两段的R位设置成功

   在`main.c`中尝试对其进行修改

   ```c++
   *_stext = 0;
   printk("_stext = %ld\n", *_stext);
   *_srodata = 0;
   printk("_srodata = %ld\n", *_srodata);
   ```

   运行结果如下

   ![验证写操作权限](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221207125854680.png)

   可以发现写操作后面的输出语句并没有执行，说明没有写操作权限

2. 为什么我们在 `setup_vm` 中需要做等值映射?

   因为汇编代码在读取当前指令的同时会直接将pc指向下一条指令，在`csrw satp`之后，地址已经全部变为虚拟地址，而pc此时在还未执行时已经取到了下一条指令，即下一条指令的物理地址，但是执行下一条指令时，操作系统会认为我们运行在虚拟内存上，会通过映射去寻找虚拟内存对应的物理内存，但是pc此时实际上存储的就是物理内存，此时去做映射无法找到对应的物理地址，会发生缺页中断，而等值映射相当于是在代码的物理地址处也开辟了 一块虚拟内存，在这一段地址建立了物理地址和虚拟地址的一一对应关系，这样就可以通过映射找到物理地址了

3. 在 Linux 中，是不需要做等值映射的。请探索一下不在 `setup_vm` 中做等值映射的方法。

   不做等值映射的话会发生缺页异常。有一个思路是利用缺页异常，在发生缺页异常时，进入用户自己设定的异常处理程序，与lab2处理时间中断类似，在中断处理程序中重新设置sepc的值，将其设置为`pc+PA2VA_OFFSET`，这样退出中断后程序就会读取虚拟地址上的汇编代码段。

    