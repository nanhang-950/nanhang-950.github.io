---
title: Android APP脱壳入门
date: 2025-11-30 23:30:38
categories: Android
tags: 
  - 脱壳
mathjax:
top:
---

## 加固简介

Android 应用加固（壳）是为了防止逆向分析、DEX dump、调试和修改，保护代码和资源的完整性与安全。随着技术发展，加固经历了四代演进：

### 第一代：整体 Dex 加密

**核心特性：**

- 整个 dex 文件加密，启动时由壳解密加载。
- 分为“落地壳”（解密后写磁盘）和“不落地壳”（内存解密）。
- 简单反调试和字符串加密。

**对抗方法：**

- 内存 dump：扫描 dex 魔数或 hook DexFile 加载函数。
- 缓存脱壳：获取 dalvik-cache / odex 文件。
- 动态调试：断点 dvmDexFileOpenPartial 等。
- Hook / Frida：hook Dex 加载函数。
- 定制系统：修改系统源码或 ROM。

**特点总结：**

- 粒度为 dex 级别。
- 一旦运行时解密被捕获，保护完全失效。

### 第二代：指令抽取

**核心特性：**

- 将关键方法体抽取，运行时按需解密回填，执行后可销毁。
- 只保护核心逻辑，避免完整 dex 长期存在。
- 部分加固厂商会结合 So 加密。

**对抗方法：**

- 类主动加载：模拟类加载触发回填（如 DexHunter、FDex）。
- 方法主动调用：在方法入口 hook 或插桩捕获 CodeItem（如 FART）。
- 定制 ROM / 系统：修改系统实现以触发回填。

**特点总结：**

- 防止传统 dex dump。
- 回填时机复杂，可通过主动调用覆盖全代码。

### 第三代：DEX 虚拟化与 Native 化

**核心特性：**

- **虚拟化**：将 Java 字节码或方法指令映射为自定义指令集，由自带解释器运行。
- **Native 化**：关键逻辑转 C/C++ 或 JNI，结合 OLLVM 混淆、控制流平坦化、虚假控制流。
- Dex 文件不再以标准格式存在，opcode 被替换或映射。

**对抗方法：**

- Hook / 逆向解释器：分析 dispatch 表及 opcode 映射。
- 动态调试 native 层（gdb / lldb / Frida）。
- Dump 运行时内存 / 回填后的 CodeItem。

**特点总结：**

- 改变执行语义或脱离 Dalvik/ART，增加逆向难度。
- Dex 已不再标准化，静态分析困难。


### 第四代：LLVM / 全原生化保护

**核心特性：**

- 整个业务逻辑或关键模块编译为 native (.so / wasm)，Java 仅做启动入口。
- 使用 LLVM/OLLVM 进行控制流平坦化、虚假控制流、指令混淆等。
- 可动态生成代码，彻底绕过 ART/Dalvik。

**对抗方法：**

- Native 动态调试：gdb / lldb / Frida。
- QEMU 跟踪动态生成结构。
- 内存 dump 并结合 LLVM 插桩分析。

**特点总结：**

- 逻辑完全原生化，逆向成本高。
- 是目前市面上强度最高的加固方案，结合虚拟化和混淆技术。


## 脱壳工具

frida-dexdump 是一个基于 Frida 的开源工具，用来在运行时在内存中查找并 dump 出应用的 dex 数据，主要用于支持逆向分析人员分析被保护、打包或混淆的 APK

- 安装：

```shell
pip3 install frida-dexdump
```

- 脱壳命令：

```shell
frida-dexdump -U -f com.xxx.xxx
```


## 脱壳实战

我在网上找了两道很经典的 CTF 题，都是二代壳。

下载链接：https://pan.baidu.com/s/1pttNHOCRwUp-tZlZ7O6U2g?pwd=3uu5

### 梆梆加固

首先查壳：

发现为梆梆加固免费版。
<img src="/assets/Pasted%20image%2020250821135948.png" width="60%">

获取包名：`com.example.how_debug`


脱壳命令：

```shell
frida-dexdump -U -f com.example.how_debug
```

最后出现以下字样则表示脱壳成功：

```
INFO:frida-dexdump:[*] All done...
```

将脱壳后 class.dex 文件丢进 jeb 中便在`Main`函数中发现 flag。

<img src="/assets/Pasted%20image%2020250821135711.png" width="60%">

### 360加固

查壳：

发现存在 360 加固。

<img src="/assets/Pasted%20image%2020250821133817.png" width="60%">

获取包名：`ctf.crack.vulcrack`

使用 frida-dexdump 脱壳：

```shell
frida-dexdump -U -f ctf.crack.vulcrack
```

将脱壳后的 class.dex 文件丢进 jeb 分析，发现 Flag 类：

<img src="/assets/Pasted%20image%2020250821140143.png" width="60%">

上面 Flag 类中定义了两个方法来解密两段 base64 编码，猜测这就是加密的 flag。方法将 base64 编码解密后交给了`comm`方法处理，`comm`方法将字符串转换为字符数组后，再遍历每个字节然后根据下标`i`做减法。

所以我们编写解密脚本时需要自己实现`comm`方法逻辑。

```python
import base64 

keyFirst = "Zm1jan85NztBN0c0NjJIOzJGLzc8STk0OTZFSDE=" 
keySecond = "QTpISTlFNEkxRTY8fQ==" 

def comm(key, num): 
	ns = [] 
	for i in range(len(key)): 
		ns.append(key[i] - (i % num)) 
	print(bytes(ns)) 

comm(base64.b64decode(keyFirst), 8) 
comm(base64.b64decode(keySecond), 4)
```

运行脚本后获取到解密的两段 flag：

```
b'flag{414A6E12-B42E-48D3-95CE-'
b'A9FF9D2F1D49}'
```

拼接在一起即可。