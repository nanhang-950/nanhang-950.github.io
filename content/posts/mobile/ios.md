---
title: IOS逆向入门
date: 2025-11-30 23:30:38
categories: Mobile
tags: 
mathjax:
top:
---

## 越狱

我的设备：iPhone X，IOS 16.1.1

越狱（Jailbreak）是对 iOS 系统安全/限制的一种突破，目的是获取更高权限（如 root）和安装非 App Store 的软件（侧载）。越狱常用于安全研究、定制系统或安装调试工具（例如 Frida）。

越狱可以分为有根越狱和无根越狱：

- 有根越狱：越狱后以 root 身份长期存在，重启后仍保持越狱状态（或能自动恢复）。多见于较老 iOS 版本与设备。
- 无根越狱：越狱状态随系统重启丢失（重启需要重新越狱），安装的插件/应用仍在但不可使用，待再次越狱后恢复可用。

### 越狱流程

设备参数：
- 无 ID 锁（必须要求）
- IOS 版本：16.1.1

[全网最详细的苹果ios设备一键越狱教程](https://www.bilibili.com/video/BV1yAtyeoETX/?share_source=copy_web&vd_source=a1e0ec33522dd06cf97338beeab421c6)

通过爱思助手实现一键越狱，我这里跟进系统版本使用的是 dopamine。

越狱中间需要登陆 Apple ID，可以自己去注册一个。然后还需要做一个设置，在通用->VPN与设备管理中选择信任。

在越狱中会过程中会让我们在手机端设置 root 密码，后面 ssh 连接 iPhone 的时候会需要使用。

### IOS App安装机制

- AppStore 下载
- 用描述文件安装
- 证书签名安装：根据苹果的设定，所有的 app 源文件，打包成 ipa 后缀的文件，都是需要证书签名，然后才能安装在苹果设备上的，上面的描述文件模式，也有做签名操作，只是在签名的时候，苹果允许以描述文件的形式导出（我自己的理解，不知道对不对了）这个证书可以是 apple ID 账号，也可以是 udid，也可以是企业证书  
- testflight 测试版安装：这个有的正能量的 app 也会选择的，不过有设备数量限制，只能多少设备安装
- 巨魔安装：这个是利用了苹果的漏洞，实现直接用 ipa 文件安装，且不需要用证书签名，这个操作又叫做侧载，巨魔的安装条件也很苛刻，只有部分设备，部分 ios 版本才可以的  

### 越狱后安装

- openssh

通过 Slieo 搜索安装，通过 mobile 用户 ssh 登陆，这里的密码就是越狱的时候设置的密码。

然后搜索安装。

- 巨魔商店

首先下载 TrollInstallerX：[下载巨魔](https://release-assets.githubusercontent.com/github-production-release-asset/766661694/6e63b308-47d2-4d44-8973-d5aa577671f3?sp=r&sv=2018-11-09&sr=b&spr=https&se=2025-10-28T03%3A09%3A52Z&rscd=attachment%3B+filename%3DTrollInstallerX.ipa&rsct=application%2Foctet-stream&skoid=96c2d410-5711-43a1-aedd-ab1947aa7ab0&sktid=398a6654-997b-47e9-b12b-9515b896b4de&skt=2025-10-28T02%3A09%3A06Z&ske=2025-10-28T03%3A09%3A52Z&sks=b&skv=2018-11-09&sig=GbeGYpBMwo1%2B9lZGEblDQvAEBM%2FlmJZVD0kLBWzWpK0%3D&jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmVsZWFzZS1hc3NldHMuZ2l0aHVidXNlcmNvbnRlbnQuY29tIiwia2V5Ijoia2V5MSIsImV4cCI6MTc2MTYxOTI4MywibmJmIjoxNzYxNjE4OTgzLCJwYXRoIjoicmVsZWFzZWFzc2V0cHJvZHVjdGlvbi5ibG9iLmNvcmUud2luZG93cy5uZXQifQ.xtLHB2-_MhGh-IlMXY59qJ1SUqVdwS4eZuce2i03_fk&response-content-disposition=attachment%3B%20filename%3DTrollInstallerX.ipa&response-content-type=application%2Foctet-stream)

然后在 Slieo 中添加源：https://repo.6ziz.com

然后在 Slieo 中搜索安装 AppSync 补丁，之后打开 TrollInstallerX 安装巨魔。

- frida

这里我选择的是 16.7.19，在 github 上下载 frida iPhone 版。
<img src="/assets/image-20251017170203316.png" style="zoom:50%;" />
安装：

```shell
scp .\frida_16.7.19_iphoneos-arm64.deb mobile@192.168.6.171:/var/jb/var/mobile
dpkg -i frida_16.7.19_iphoneos-arm64.deb
# 运行服务端
frida-server -l 0.0.0.0:6666
```

测试：

<img src="/assets/Pasted%20image%2020251028110849.png" width="50%">


## 基础知识

IOS 源自 MacOS，而 MacOS 又基于 Unix 系统内核，因此其目录结构基本与 Unix 系统相同，实际上 IOS 系统包含两类目录，一类是保留的 Unix 传统目录，另一类是 IOS/MacOS 特有的目录。

### 沙盒结构

沙盒也叫沙箱，其原理是把应用程序生成和修改的文件重定向到自身文件夹中。在沙盒机制下，每个程序之间的文件夹不能互相访问。

为了保证系统安全，IOS 采用了这种机制。IOS 应用程序在安装时会创建属于自己的沙盒文件，沙盒的根目录有三个文件夹，分别是 ：

- `Documents`：用于存储应用程序的数据，改路径可通过配置来实现 ITunes 文件共享，可被 ITunes 备份。
- `Library`：目录下有两个子目录
  - `Preferences`：保存应用程序的偏好设置文件，ITunes 同步设备时会备份该目录。
  - `Caches`：主要存放缓存文件，用来保存应用程序运行时产生的需要持久化的数据，这些数据一般存储体积比较大，但也不是特别重要，需要用户负责删除，ITunes 同步设备时不会备份该目录。
- `tmp`：保存应用程序运行时所需的临时数据，使用完毕后再将相应的文件从改目录删除。应用程序没有运行时，系统也可能会清除该目录下的文件。ITunes 同步设备时不会备份该目录。

### 应用结构

使用 IOS 系统，普通用户只有从 App Store 下载 App，而对于安全人员来说，还可以从越狱商店下载 App。

它们具有以下差别：

- Applications：存放的是 IOS 系统自身的 App 以及从越狱商店安装的 App。
- App Store 下载的 App 则存放在`/var/containers/Bundle/Application`目录。

安装包的差别：

- 越狱商店的 App 安装包格式通常为 deb。
- 而 App Store 的 App 安装包格式通常是 ipa。

文件权限的差别：

- 越狱商店的 App 能够以 root 权限运行。
- App Store 的 App 只能以 mobile 权限运行。

IPA 文件和 APK 文件一样都是 ZIP 文件，一个 IPA 包中必然存在一个 Payload 文件夹，里面存放的`*.app`文件夹才是真正的包内容。

- `Info.plist`文件是应用的主要配置文件。
- `*.lproj`文件夹：存放各种本地化的字符串。
- 资源文件：

### 文件权限

**iOS 是一个多用户操作系统**，每个用户扮演着不同的角色，对系统的控制权也各不相同。虽然表面上 iOS 设备只有一个“用户”，但实际上系统内部有多个账户与权限层级，用于实现安全隔离和访问控制。

iOS 基于 **Darwin 内核**（即 macOS 的底层），同样使用 **UNIX 权限模型**（UID、GID、rwx 权限）。
 在 iOS 中，主要存在以下几类用户：

| 用户                                        | UID                                | 说明                                                |
| ------------------------------------------- | ---------------------------------- | --------------------------------------------------- |
| `root`                                      | 0                                  | 超级用户，拥有系统全部控制权                        |
| `mobile`                                    | 501                                | 普通 App 运行的主要用户（SpringBoard、App Sandbox） |
| `_installd` / `_networkd` / `_locationd` 等 | 各种系统服务专用用户，权限受限     |                                                     |
| `_securityd`                                | 安全相关进程（密钥管理、签名验证） |                                                     |
| `_analyticsd`                               | 分析与日志进程                     |                                                     |

> 非越狱设备中，**App 永远以 `mobile` 用户身份运行**，并被严格限制在自己的沙盒中。

iOS 文件权限遵循标准的 UNIX 模型：

```
-rwxr-xr--   1 root  wheel   12345 Jan 17 12:00 AppBinary
```

含义如下：

| 符号                 | 含义                                       |
| -------------------- | ------------------------------------------ |
| `r`                  | 读权限 (read)                              |
| `w`                  | 写权限 (write)                             |
| `x`                  | 执行权限 (execute)                         |
| `owner/group/others` | 分别对应文件拥有者、所属组、其他用户的权限 |

不过，iOS 在此基础上引入了多层额外机制以强化安全性。

### Mach-O文件

Mach-O（Mach Object file format） 是 MacOS、iOS 等 Apple 系统中可执行文件、动态库（dylib）、内核扩展（kext）和调试符号（dSYM）的底层文件格式。

它是 Apple 自定义的可执行格式，功能类似于 Windows 的 PE 和 Linux 的 ELF。常见的有可执行文件和动态链接库。可执行文件和 ELF 文件一样无扩展名，动态链接库以 dylib 为扩展名。

## 砸壳

IOS 端 App 在上线前会由苹果商店进行 FairPlayDRM 数字版权加密保护，要对应用进行分析就必须先解密，从而得到原始未加密的二进制文件。

砸壳工具：frida-ios-dump

frida-ios-dump 是一款基于 frida 的 ios 脱壳工具。

```shell
wget https://github.com/AloneMonkey/frida-ios-dump
```

安装依赖：

```shell
pip3 install -r requirements.txt --upgrade
```

脚本使用很简单，修改 dump.py 配置一下你的 ssh ip 和端口、账号密码，它设置的默认端口为 2222 ，我们可以直接改成 22。

运行查看：

```shell
python3 dump.py -l
```

这里我通过 AI 对脚本做了一些修改，主要是指定 frida 服务端的地址和端口。

<img src="/assets/Pasted%20image%2020251029221642.png" style="zoom:50%;" />

运行命令

<img src="/assets/Pasted%20image%2020251029222934.png" style="zoom:50%;" />

然后就可以使用 zip 解压缩 计算器.ipa 文件。

```
unzip ./计算器.ipa
```

## 反编译

反编译分析的目标是 Mach-O 文件，一般都是应用程序的名字。

我们也可以通过`file`命令来判断目标：

```shell
file * | grep -i mach
```

IOS APP 都是使用 OC 或者是 Swift 开发的，我们可以使用 IDA 进行反编译分析：

<img src="/assets/Pasted%20image%2020251029223917.png" style="zoom:50%;" />


## 抓包

Reqable 是一个移动端非常优秀的抓包工具，我们可以通过它来抓取 IOS APP 的数据包。

Reqable 可以直接在 App Store 中安装 。

然后我们需要在手机端安装证书，这里跟着应用上操作就行。

然后我们就可以配置移动端和电脑端的协同，这里和 Android 一样都是扫码连接。

然后在电脑端我们可以直接选定左边手机端的 IP，这样我们就指挥接收它的数据包。

<img src="/assets/Pasted%20image%2020251029230200.png" style="zoom:50%;" />

## 参考

>[57《iOS逆向教程之快速入门》_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV1xWQaYMEhs/?spm_id_from=333.788.videopod.episodes&vd_source=edca928f1ab30d19924c36939a858238)
>[从iPhone越狱到打造完美手机 | LOTUS BLOG](https://blog.lotusssb.com/posts/from_iphone_jailbreaking_to_the_perfect_phone/)
>[iOS逆向 - 应用脱壳iOS端APP在上线之前，会经过苹果商店进行FairPlayDRM数字版本加密保护，俗称加壳](https://juejin.cn/post/7066318008758566948)
>[iOS逆向分析工具使用汇总逆向App总体思路 UI分析 Cycript 、Reveal; 代码分析 代码在Mach-O](https://juejin.cn/post/7051805569174208542#heading-0)
>[安装Frida插件 · iOS逆向开发：越狱包管理器](https://book.crifan.org/books/ios_re_package_manager/website/sileo/install_tweak/from_repo/install_frida_from_repo.html)