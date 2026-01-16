---
title: VSCode调试Linux Kernel
date: 2025-11-30 23:30:38
categories: Linux
tags:
mathjax:
top:
---

## 前言

## 编译内核

- 安装工具

安装编译链工具：

```shell
sudo apt update
sudo apt install -y build-essential libncurses-dev flex bison libssl-dev libelf-dev bc
sudo apt install gcc-aarch64-linux-gnu g++-aarch64-linux-gnu
```

- 下载内核源码

```shell
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.15.167.tar.xz
tar -xJf linux-5.15.167.tar.xz
cd linux-5.15.167
```

- 编译

```shell
#配置编译工具链
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- defconfig

./scripts/config --enable CONFIG_VIRTUALIZATION
./scripts/config --enable CONFIG_KVM
./scripts/config --enable CONFIG_KVM_ARM_HOST
./scripts/config --enable CONFIG_VIRTIO
./scripts/config --enable CONFIG_VIRTIO_PCI
./scripts/config --enable CONFIG_VIRTIO_BLK
./scripts/config --enable CONFIG_VIRTIO_NET
./scripts/config --enable CONFIG_VIRTIO_CONSOLE

#更新配置
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- olddefconfig  

#编译
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)    
```

编译后的结果如下：

- `vmlinux`：原始内核文件
- `bzImage`：压缩内核镜像

## 构建文件系统

构建文件系统需要使用 busybox，busybox 是一个集成了大量 Unix 工具的精简版系统工具集，常用于嵌入式系统和 initramfs。

- 构建 busybox

```shell
#下载源码
wget https://busybox.net/downloads/busybox-1.37.0.tar.bz2

#解压
tar -jxvf busybox-1.37.0.tar.bz2

# 使用默认配置
make defconfig

# 使用脚本方式开启静态编译选项
sed -i 's/^# CONFIG_STATIC is not set/CONFIG_STATIC=y/' .config

#编译
make -j$(nproc)  
make install
```

编译完成后会生成一个 `_install` 目录，接下来我们将会用它来构建我们的文件系统。

- 初始化文件系统

```shell
cd _install  
mkdir -pv {bin,sbin,etc,proc,sys,dev,home/ctf,root,tmp,lib64,lib/x86_64-linux-gnu,usr/{bin,sbin}}  
touch etc/inittab  
mkdir etc/init.d  
touch etc/init.d/rcS  
chmod +x ./etc/init.d/rcS
```

- 配置初始化脚本

首先配置`etc/inttab` ，写入如下内容：

```shell
::sysinit:/etc/init.d/rcS  
::askfirst:/bin/ash  
::ctrlaltdel:/sbin/reboot  
::shutdown:/sbin/swapoff -a  
::shutdown:/bin/umount -a -r  
::restart:/sbin/init
```

在上面的文件中指定了系统初始化脚本，因此接下来配置`etc/init.d/rcS`写入如下内容，主要是挂载各种文件系统：

```shell
#!/bin/sh  
mount -t proc none /proc  
mount -t sysfs none /sys  
mount -t devtmpfs devtmpfs /dev  
mount -t tmpfs tmpfs /tmp  
mkdir /dev/pts  
mount -t devpts devpts /dev/pts  
  
echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n"  
setsid cttyhack setuidgid 1000 sh  
  
poweroff -d 0 -f
```

也可以在根目录下创建`init`文件，写入如下内容：

```shell
#!/bin/sh  
chown -R root:root /  
chmod 700 /root  
chown -R ctf:ctf /home/ctf  
  
mount -t proc none /proc  
mount -t sysfs none /sys  
mount -t tmpfs tmpfs /tmp  
mkdir /dev/pts  
mount -t devpts devpts /dev/pts  
  
echo 1 > /proc/sys/kernel/dmesg_restrict  
echo 1 > /proc/sys/kernel/kptr_restrict  
  
echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n"  
  
cd /home/ctf  
su ctf -c sh  
  
poweroff -d 0 -f
```

并且要添加可执行权限：

```
chmod +x ./init
```

- 配置用户组

```shell
echo "root:x:0:0:root:/root:/bin/sh" > etc/passwd
echo "ctf:x:1000:1000:ctf:/home/ctf:/bin/sh" >> etc/passwd

echo "root:x:0:" > etc/group
echo "ctf:x:1000:" >> etc/group

echo "none /dev/pts devpts gid=5,mode=620 0 0" > etc/fstab
```

- 配置glibc库

如果使用的是 glibc，而不是 busybox 内置的 libc，需确定动态链接库路径。

```
ldd /bin/busybox
```

- 打包文件系统为 cpio 镜像文件

打包文件系统为 cpio 格式

```shell
#将当前目录及其子目录下的所有文件打包成一个 cpio 格式的归档文件
find . | cpio -o -H newc > ./rootfs.cpio
```

## 模拟运行

制作好文件系统文件后，我们就可以通过 qemu 工具来模拟运行 Linux 内核。

- **环境搭建**

```shell
sudo apt install qemu-system
```

- **模拟内核**

模拟内核我们需要编写一个启动脚本`start.sh`来使用 qemu。

使用`cpio`文件系统：

```shell
#!/bin/sh
qemu-system-aarch64 \
    -machine virt \
    -cpu cortex-a57 \
    -m 256M \
    -kernel ./arch/arm64/boot/Image \
    -initrd ./rootfs.cpio \
    -append "root=/dev/ram rdinit=/sbin/init console=ttyAMA0 earlyprintk=serial init=/init" \
    -smp 2 \
    -nographic \
    -no-reboot \
    -monitor none \
    -fsdev local,id=fs1,path=/tmp,security_model=none \
    -device virtio-9p-pci,fsdev=fs1,mount_tag=hostshare \
    -s
```

## 配置vscode

- C/C++ 调试插件

首先在内核目录下创建一个`.vscode`目录，然后在目录中创建一下两个文件：

- `launch.json`

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "kernel-dbg",
      "type": "cppdbg",
      "request": "launch",

      "preLaunchTask": "start-kernel-env",

      "miDebuggerServerAddress": "127.0.0.1:1234",
      "program": "${workspaceFolder}/vmlinux",
      "args": [],
      "stopAtEntry": false,
      "cwd": "${workspaceFolder}",
      "environment": [],
      "externalConsole": false,
      "logging": {
        "engineLogging": true
      },
      "MIMode": "gdb",
      "setupCommands": [
        {
          "description": "启用漂亮打印",
          "text": "-enable-pretty-printing",
          "ignoreFailures": true
        },
        {
          "description": "将反汇编风格设置为 Intel",
          "text": "-gdb-set disassembly-flavor intel",
          "ignoreFailures": true
        }
      ]
    }
  ]
}

```

- `tasks.json`

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "start-kernel-env",
      "type": "shell",
      "command": "${workspaceFolder}/start.sh",
      "problemMatcher": [],
      "presentation": {
        "reveal": "always",
        "panel": "shared"
      }
    }
  ]
}
```

- `start.sh`

```shell
#!/bin/sh
qemu-system-aarch64 \
     -m 256M \
     -kernel ./arch/arm64/boot/bzImage \
     -initrd ./rootfs.cpio \
     -monitor /dev/null \
     -append "root=/dev/ram rdinit=/sbin/init console=ttyS0 oops=panic panic=1 loglevel=3 quiet kaslr" \
     -cpu kvm64,+smep \
     -smp cores=2,threads=1 \
     -nographic \
     -s
```

## 效果

在`execve`系统调用定义处打一个断点，然后 F5 开始调试，然后就可以看到内核断到了这里。



## 后言

通过 vscode 调试内核源码，我们就可以很清晰的分析内核源码学习内核。