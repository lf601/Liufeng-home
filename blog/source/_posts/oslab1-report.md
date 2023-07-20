---
title: oslab1_report
date: 2023-07-08 14:45:19
tags: os实验报告
categories: 操作系统
banner_img: /img/os_report.jpg
---

### Lab1 实验报告

#### 代码

`head.S`

```assembly
.extern start_kernel

    .section .text.entry
    .globl _start
_start:
    
    la sp,boot_stack_top #存放栈指针
    call start_kernel #调用start_kernel

    .section .bss.stack
    .globl boot_stack
boot_stack:
    .space 0x1000000 # <-- change to your stack size

    .globl boot_stack_top
boot_stack_top:
```



`lib/Makefile`

```makefile
C_SRC       = $(sort $(wildcard *.c))
OBJ		    = $(patsubst %.c,%.o,$(C_SRC))

file = print.o
all:$(OBJ)
	
%.o:%.c
	${GCC} ${CFLAG} -c $<
clean:
	$(shell rm *.o 2>/dev/null)
```

其中前两行定义了2个变量，`C_SRC`为所有的.c文件，`OBJ`变量为把`C_SRC`中所有的.c替换成.o

`all:`指定了输出的伪目标为所有的.o文件

`%.o:%.c`为.o和对应的.c文件指定了依赖关系

`2>/dev/null`将生成的垃圾文件重定向到`/dev/null`目录下，执行`make clean`后将这些文件删除



`sbi.c`

```c
#include "types.h"
#include "sbi.h"


struct sbiret sbi_ecall(int ext, int fid, uint64 arg0,
			            uint64 arg1, uint64 arg2,
			            uint64 arg3, uint64 arg4,
			            uint64 arg5) 
{
    struct sbiret result;
	__asm__ volatile(
		"mv a0, %[arg0]\n"
		"mv a1, %[arg1]\n"
		"mv a2, %[arg2]\n"
		"mv a3, %[arg3]\n"
		"mv a4, %[arg4]\n"
		"mv a5, %[arg5]\n"
		"mv a6, %[arg6]\n"
		"mv a7, %[arg7]\n"
		"ecall\n"
		"mv %[error], a0\n"
		"mv %[value], a1"
		:[error] "=r" (result.error), [value] "=r" (result.value)
		:[arg0] "r" (arg0), [arg1] "r" (arg1), [arg2] "r" (arg2), [arg3] "r" (arg3), [arg4] "r" (arg4), [arg5] "r" (arg5), [arg6] "r" (fid), [arg7] "r" (ext)
		:"memory"
	);

	return result;

}
```

`mv`语句用于将寄存器与变量`arg0`-`arg7`绑定，然后调用`ecall`指令，最后将`a0`和`a1`寄存器的值存到`result`中输出



`puts()`

```c
void puts(char *s) {
    int i = 0;
    while(s[i]!='\0')
    {
        sbi_ecall(0x1,0x0,(int)s[i],0,0,0,0,0);
        i++;
    }
}
```

从头到尾依次调用`sbi_ecall`输出字符



`puti()`

```c
void puti(int x) {
    int a,b;

    if(x > 0)
    {
        a = x%10;
        b = x/10;
        if(b != 0)
            puti(b);
        sbi_ecall(0x1,0x0,0x30+a,0,0,0,0,0);
    }
    else if(x == 0)
        sbi_ecall(0x1,0x0,0x30,0,0,0,0,0);
    else if(x < 0)
    {
        x=-x;
        sbi_ecall(0x1,0x0,0x2d,0,0,0,0,0);
        if(x > 0)
        {
            a = x%10;
            b = x/10;
            if(b != 0)
                puti(b);
            sbi_ecall(0x1,0x0,0x30+a,0,0,0,0,0);
        }
    }

}
```

先判断`x`的正负性，分类讨论，采用递归的方式输出



`csr_read(csr)`

```c
#define csr_read(csr)                       \
({                                          \
    register uint64 __v;                    \
    asm volatile ("csrr " "%0, " #csr       \
                    : "=r" (__v) :          \
                    : "memory");            \
    __v;                                    \
})
```

模仿`csr_write(csr,val)`使用内联汇编进行宏的编写



实现效果

![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221012215909608.png)



#### 思考题

1. 函数调用分为6个阶段：

   * 将参数存到函数能访问到的位置
   * 跳转到函数开始位置
   * 获取函数需要的局部存储资源，按需要保存寄存器
   * 执行函数中的指令
   * 将返回值存储到调用者能访问到的位置，恢复寄存器，释放局部存储资源
   * 返回调用函数的位置

   为了获取良好的性能，变量应该存放在内存而不是寄存器中。在函数调用中不保留某些寄存器存储的值，称为临时寄存器(caller saved register)，另一些寄存器则称为保存寄存器(callee saved register)。caller saved register为caller需要主动保存的寄存器，callee可以直接对其进行更改，在函数调用前后可能发生改变，所以为调用者保存寄存器。同理callee saved register在函数调用前后不发生改变，为被调用者保存寄存器。

   

2. ![函数表](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221011203226755.png)

3. test.c

   ```c++
   puti(csr_read(sstatus));
   while(1);
   ```

   

   ![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221011215221310.png)

   其中sstatus寄存器各位含义如下

   ![sstatus寄存器布局](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221013083129129.png)

   | 字段名称 | bit  | 含义                                                         | 功能                                                         |
   | -------- | ---- | ------------------------------------------------------------ | ------------------------------------------------------------ |
   | SSP      | 8    | 0：表示之前的mode为U-mode                                    | 记录hart在进入s-mode前的执行模式。当执行SRET指令，如果SPP为0，设置为用户模式，如果SPP为1，设置为超级用户模式并将SPP置0。 |
   | SIE      | 1    | 0：禁止中断       1：开启中断                                | 表示在s-mode下禁止还是开启中断                               |
   | SPIE     | 5    | 表示在trap中禁止S-mode中断                                   | 表示在trap发生在s-mode之前，是否启用s-mode中断。当trap在s-mode时，SPIE设置到SIE中，并且把SIE设置为0。当执行SRET指令后，SIE设置到SPIE中，然后将SPIE设置为0。 |
   | UBE      | 6    | 全称：Endianness  Control                 0：小端字节序     1：大端字节序 | 控制从U-mode进行显式内存访问的字节序，对指令访问没有影响。   |

   

4. test.c

   ```c++
   csr_write(sscratch,111);
   puti(csr_read(sscratch));
   ```

   

   ![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221011215315041.png)

5. 安装aarch64交叉编译工具链![工具链安装](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221011235851303.png)

   使用默认配置![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221011235907925.png)

   编译直到获得sys.i文件![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221012000107365.png)

   使用vim查看文件![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221012000343915.png)

6. `ARM32`

   路径

   `linux\arch\arm\tools`

   ![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221012001733054.png)

   `x86(32bit)`

   路径

   `linux\arch\x86\entry\syscalls`

   ![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221012001433432.png)

   `x86(64bit)`

   路径

   `linux\arch\x86\entry\syscalls`

   ![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221012001454428.png)

   `riscv(64bit)`

   路径

   `linux\arch\riscv\kernel\syscalls`

   ![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221012002716441.png)

7. ELF文件是linux kernel编译出的带调试信息和符号表的可执行文件，在OS实验中为代码编译链接后生成的可供QEMU运行的RISCV64架构程序。ELF文件既可以参与程序的链接也可以用于执行。若用于链接，则将ELF文件视为`Section Headers Table`描述的`Section`的集合。若用于执行，则被视为`Program Headers Table`描述的段的集合。

   `readelf -a vmlinux`

   ![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221012112706516.png)

   打开2个terminal，其中一个make run整个repo

   另一个terminal输入`ps au`查看现在运行的进程

   ![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221012112921578.png)

   输入`cat /proc/PID/maps`

   ![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221012113157989.png)