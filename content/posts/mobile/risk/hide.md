---
title: 隐藏 Magisk&Xposed
date: 2026-01-17 9:51:38
categories: Mobile
tags: 
    - 检测对抗
mathjax:
top:
---

## 前言

看到群里的大佬发了一个隐藏环境入门 PDF 文档，于是跟着复现一下。

## Magisk hide

- 在设置中找到并点击隐藏 Magisk 应用

<img src="/assets/image-20260123153918898.png" alt="image-20260123153918898" style="zoom:50%;" />

- 设置一个新的 Magisk 应用名称并确认：

<img src="/assets/image-20260123153947280.png" alt="image-20260123153947280" style="zoom:50%;" />

片刻后会生产一个随机包名的面具包，然后让我们添加一个快捷方式到桌面，我们可以通过该快捷方式打开面具。

- 在设置中开启 Zygisk：

<img src="/assets/image-20260123154154895.png" alt="image-20260123154154895" style="zoom:50%;" />

然后重启手机，再次打开面具。

- 在设置中开启遵守排除列表：

<img src="/assets/image-20260212173058834.png" alt="image-20260212173058834" style="zoom:50%;" />

- 在设置中点击配置排除列表，然后在排除列表中选择需要隐藏 ROOT 的对应应用。

例如 Duck Detector。

<img src="/assets/image-20260212173126432.png" alt="image-20260212173126432" style="zoom:50%;" />

注意将应用下的选项全部开启，否则可能会出现隐藏 ROOT 失败的问题。

<img src="/assets/image-20260212173208197.png" alt="image-20260212173208197" style="zoom:50%;" />

但是此时排除列表中的应用是无法使用虚拟框架和模块的。如果我想对某个排除列表中的应用使用虚拟框架和模块

就需要使用到 Shamiko 模块：[Releases · LSPosed/LSPosed.github.io](https://github.com/LSPosed/LSPosed.github.io/releases)

在面具 Magisk 的模块功能中使用本地安装刷入该模块，然后重启手机。再次打开面具 Magsik，在其模块功能中可以看到 Shamiko 模块已经成功刷入并加载。

此时 Shamiko 中有显示 Shamiko doesn't work since enforce denylist is enabled：

<img src="/assets/image-20260212174007490.png" alt="image-20260212174007490" style="zoom:50%;" />

- 成功隐藏：

<img src="/assets/image-20260212173929665.png" alt="image-20260212173929665" style="zoom:50%;" />

## Xposed hide

HMA（Hide My Applist）是一个用于隐藏应用或拒绝应用列表请求的 Xposed 模块。

但是这个模块的更新速度较慢，所以出现了它的好用的分支 HMA-OSS

下载：https://github.com/frknkrc44/HMA-OSS/releases

以下介绍它的使用步骤：

>重要提示：在 LSPosed 中仅勾选系统框架，绝对不要勾选其它应用。

- 打开 HMA-OSS 应用，点击应用管理。

<img src="/assets/image-20260117012503170.png" alt="image-20260117012503170" style="zoom:50%;" />

- 选择要隐藏 applist 的应用

这里选择 Duck Detector 应用来检测是否成功绕过 Root和 Xposed 检测。

<img src="/assets/image-20260117013751009.png" alt="image-20260117013751009" style="zoom:50%;" />

- 然后启用隐藏

<img src="/assets/image-20260117013824831.png" alt="image-20260117013824831" style="zoom:50%;" />

- 点击模板设置

<img src="/assets/image-20260117013915164.png" alt="image-20260117013915164" style="zoom:50%;" />

- 保持黑名单模式不变，打开应用预设

<img src="/assets/image-20260117013134561.png" alt="image-20260117013134561" style="zoom:50%;" />

- 勾选全部

<img src="/assets/image-20260117013201442.png" alt="image-20260117013201442" style="zoom:50%;" />

- 最后打开 Duck 发现顺利隐藏

<img src="/assets/image-20260117013620222.png" alt="image-20260117013620222" style="zoom:50%;" />