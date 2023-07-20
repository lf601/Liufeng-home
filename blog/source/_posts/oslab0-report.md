---
title: oslab0_report
date: 2023-07-08 14:45:05
tags: os实验报告
categories: 操作系统
banner_img: /img/os_report.jpg
---

## Lab 0 实验报告

#### 1搭建Docker环境

打开cmd，进入D盘`D:\oslab`目录

输入以下命令

`docker load < oslab.tar`生成镜像

![生成镜像](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20220930190728703.png)

输入`docker images`查看镜像

![查看镜像](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20220930190755741.png)

输入以下命令生成name为oslab的容器

`docker run --name oslab -it oslab:2021 bash`

提示符变为`#`说明成功进入容器

![进入容器](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20220930191111969.png)

输入`exit`退出容器，输入以下命令

`docker ps`

![查看运行中容器](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20220930191202365.png)

发现没有正在运行中的容器

再输入以下命令启动停止状态的容器

```bash
docker start oslab
docker ps
```

![查看启动后容器](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20220930191518070.png)

输入`docker exec -it oslab bash`从终端连入docker容器

输入

`docker run --name OSLAB -it -v C:\Users\LEGION:/have-fun-debugging oslab:2021 bash`

新建一个容器将本地文件夹映射到新容器中的`have-fun-debugging`文件夹

![与windows文件系统建立映射](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221001182108936.png)



#### 2获取Linux源码和已经编译好的文件系统

输入`git clone https://github.com/ZJU-SEC/os22fall-stu`

![下载源码](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002125433963.png)

![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002125500532.png)

#### 3编译Linux内核

![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002132043525.png)

![编译](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002132232029.png)

输入`make ARCH=riscv CROSS_COMPILE=riscv64-unknown-linux-gnu- -j8`编译内核



#### 4使用QEMU运行内核

在`linux`目录下输入

```bash
qemu-system-riscv64 -nographic -machine virt -kernel /debug/linux/arch/riscv/boot/Image -device virtio-blk-device,drive=hd0 -append "root=/dev/vda ro console=ttyS0" -bios default -drive file=/os22fall-stu/src/lab0/rootfs.img,format=raw,id=hd0
```

![运行](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002152532546.png)

![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002152602418.png)



#### 5使用GDB对内核进行调试

关闭前一步中打开的终端，在新的终端中使用QEMU运行内核

```bash
qemu-system-riscv64 -nographic -machine virt -kernel /debug/linux/arch/riscv/boot/Image -device virtio-blk-device,drive=hd0 -append "root=/dev/vda ro console=ttyS0" -bios default -drive file=/os22fall-stu/src/lab0/rootfs.img,format=raw,id=hd0 -S -s
```

![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002153246304.png)

因为添加了 `-S`所以暂停

打开一个新的终端进入`oslab`容器

输入`riscv64-unknown-linux-gnu-gdb /debug/linux/vmlinux`

![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002153554022.png)

连接QEMU

![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002154033053.png)

在0x80000000和0x80200000设置断点

![设置断点](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002154804151.png)

运行至0x80200000的断点

![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002154902119.png)

![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002154915297.png)

输入`step instruction`执行单步指令

![单步调试](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002155054592.png)

通过`info`指令查看寄存器的值

![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002155213087.png)

观察`addi gp,gp,-716`指令执行后`gp`寄存器值的变化

![](https://typora-lqy.oss-cn-hangzhou.aliyuncs.com/image/image-20221002155431103.png)

退出



#### 思考题

5.`vmlinux`和`Image`同为`linux`的镜像，其中，`vmlinux`是对`linux`源码编译得到的`elf`格式的文件，文件较大，`Image`是`vmlinux`经过`objcopy`处理后的二进制文件，只包含内核代码的二进制数据。