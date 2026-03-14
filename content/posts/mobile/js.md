---
title: 小程序逆向入门
date: 2025-11-30 23:30:38
categories: Mobile
tags: 
  - 小程序
  - JS
mathjax:
top:
---

## 前言

现如今微信小程序的使用范围越来越广，逐渐形成了一个生态。由此相应的小程序安全也成了人们关注的问题。所以与时俱进，我们也需要学习小程序的安全。

小程序安全的三板斧：抓包、解包、反编译。

## 基础知识

先来了解一下关于小程序的基础知识。

### 小程序架构分析

微信小程序采用了框架化开发，主要分为视图层、逻辑层、JSBridge。JS 负责业务逻辑层的实现，而视图层则由 WXML 和 WXSS 来共同实现，前者其实就是一种微信定义的模板语言，而后者类似 CSS。

从下面的微信小程序架构图上可以清晰的看出，小程序借助的是 JSBridge 实现了对底层 API 接口的调用，所以在小程序里面开发，开发者不用太多去考虑 IOS、安卓的实现差异的问题，安心在上层的视图层和逻辑层进行开发即可。

<img src="/assets/xxc.png" alt="xxc" style="zoom:50%;" />

- 视图层

框架的视图层由 WXML 与 WXSS 编写，由组件来进行展示，将逻辑层的数据反应成视图，同时将视图层的事件发送给逻辑层。

- 逻辑层

逻辑层将数据进行处理后发送给视图层，同时接受视图层的事件反馈。 

### 文件组成部分

一个小程序主体部分由三个文件组成，必须放在项目的根目录：

- APP.js：小程序逻辑；
- APP.json：小程序公共设置，决定页面文件的路径、窗口表现、设置网络超时时间等；
- APP.wxss：小程序公共样式表。

一个小程序页面由四个文件组成：

- JS：页面逻辑
- WXML：页面结构，框架设计的一套标签语言，结合基础组件、事件系统，可以构建出页面的结构；
- WXSS：：是一套样式语言，用于描述 WXML 的组件样式，用来决定 WXML 的 组件应该怎么显示；
- JSON：页面配置。

<img src="assets/image-20251001152836833.png" alt="image-20251001152836833" style="zoom: 50%;" />

从上述的架构图、文件组成部分来看，重点分析的就是小程序的逻辑层。而逻辑层主要的组成部分是由APP.js、APP.json、JS文件、JSON配置文件等组成，因此测试过程中主要分析的对象就是这一些。

### 小程序缓存目录

找到微信小程序的缓存目录，当然里面会存在很多的文件夹，可以都删除，然后打开一个小程序加载。

<img src="assets/Pasted%20image%2020250923192646.png" style="zoom: 50%;" />

- Windows缓存目录：`C:\Users\nanhang\AppData\Roaming\Tencent\xwechat\radium\Applet\packages\`

### 尝试加载小程序

这里我们只点击一个小程序加载，加载之后会在缓存目录下面生成一个后缀为 wxapkg 文件，这个文件就是小程序编译后的文件。

## 解包

微信小程序分发后的文件后缀为 pkg。在写这篇指南的时间为止，提取出来的 pkg 文件还是可以通过工具直接还原成可读的源代码，但不排除后续微信更新， 增加 pkg 文件提取、逆向还原的难度。

微信小程序在手机中存放的目录如下：

- **Android**： `/data/data/com.tencent.mm/MicroMsg/不同微信号的值不同 /appbrand/pkg `
- **IOS**：`/var/mobile/Containers/Data/Application/不同微信号的值不同 /WechatPrivate/c15d9cced65acecd30d2d6522df2f973/WeApp/LocalCache/release`

如果手机上打开了很多微信小程序，那么目录中就会存在多个小程序 pkg 文件。 第一种区分方法是按照修改时间来进行区分。第二种方法是在微信页面中删除 所有浏览过的小程序，重新打开需要进行测试的小程序，那么目录中只会存在 一个pkg文件。

通过 UnpackMiniApp 解密微信小程序

https://github.com/qwerty472123/wxappUnpacker 小程序逆向还原工具，依赖 nodejs 等，具体安装、使用说明可以参照链接中的内容。 

常用还原源码命令：

```shell
node wuWxapkg.js –o –d 目标文件.pkg
```

## 解密

## 反编译

关于小程序的分包反编译，目前没有特别好的免费工具来进行，好的工具都需要充值vip。

这里我们就尝试手动反编译吧，不过会存在问题哦，不一定是百分比能够成功的，毕竟像之前说的好工具是要钱的，尝试找破解版的，但是没用。

### 安装nodejs

这里我不知道我是什么时候安装的，可能安装的工具多了就曾经在某一刻是安装过的，这里我就不再安装了，给个连接自行安装吧。

安装后打开cmd输入`node -v`看到显示版本就证明安装成功了。

```shell
node -v
npm install esprima css-tree cssbeautify vm2 uglify-es js-beautify
```

### 反编译脚本

打开反编译脚本，将刚刚解密后的文件放入该目录下。

安装依赖：

这里就需要先在反编译脚本目录下安装依赖，这里挺麻烦的，但是没办法呀，穷买不起会员，无法一键反编译。

```
npm install esprima
npm install css-tree
npm install js-beautify
npm install uglify-es
npm install vm2
npm install cssbeautify
```

执行反编译：

如果上面一切顺利的话就可以执行反编译了。

```
node wuWxapkg.js wx6642bb131bfd374b.wxapkg
```

### 查看反编译后文件

这里我们就可以在当前目录下查看对应的反编译文件夹了,看到后就可以使用微信开发者工具打开了。

[微信开发者工具](https://developers.weixin.qq.com/miniprogram/dev/devtools/stable.html)

### 查看工程

到这里就可以将文件加入微信开发者工具的工程中了，不过这里需要申请一个测试账号，非常简单，无需什么麻烦的操作。

### 解密

在 UnpackMiniApp 文件夹中创建`wxpack`文件夹。

```powershell
mkdir wxpack
```

微信开发者工具下载：[下载](https://developers.weixin.qq.com/miniprogram/dev/devtools/download.html)

## 抓包

使用电脑版微信抓包。

### 配置微信代理

在微信的登陆页面配置代理，IP地址就是你主机的IP地址。

<img src="assets/Pasted%20image%2020250923151024.png" style="zoom:50%;" />

配置代理 IP 地址为 127.0.0.1，端口配置为 Proxying 显示的端口。

<img src="assets/Pasted%20image%2020250923151316.png" style="zoom:50%;" />


然后登陆微信之后，在登陆过程中就可以看到很多数据包。

我们可以使用 Reqable 的筛选功能筛选来自微信的数据包，点击左侧的微信图标即可进行筛选。

<img src="assets/Pasted%20image%2020250923151420.png" style="zoom: 50%;" />

对于废弃的数据包，我们可以邮件调试按钮进行清除。

在不抓取数据的时候关闭右上角的系统代理。

同样，我们使用 Yakit 抓小程序的包只需要将监听端口改为微信代理中设置的端口。

<img src="assets/Pasted%20image%2020251001203049.png" style="zoom:50%;" />

### 抓包小程序

现在就可以抓小程序的包了，将之前的数据包清空后，打开要抓取的小程序。比如这里我们选择瑞幸咖啡，然后就会获取到小程序的数据包：

<img src="assets/image-20250923201256949.png" alt="image-20250923201256949" style="zoom:50%;" />

## 开启Devtools

通过工具开启微信的 Devtools 进行 JS 调试。

下载工具：[下载](https://pan.baidu.com/share/init?surl=ow2FeAH1ucEWgYJcfzgUNQ&pwd=xjfe)

下载相应版本的微信：[下载微信](https://github.com/tom-snow/wechat-windows-versions/releases/download/v3.9.12.17/WeChatSetup-3.9.12.17.exe)

开启微信之前首先清除`C:\Users\用户\AppData\Roaming\Tencent\WeChat\XPlugin\Plugins\RadiumWMPF`下的所有目录，然后再打开微信，之后再启动工具。

之后点击小程序的图标就可以开启 Devtools。

<img src="/assets/image-20260102121753791.png" alt="image-20260102121753791" style="zoom: 67%;" />

开启后：

<img src="/assets/image-20260102121831997.png" alt="image-20260102121831997" style="zoom: 33%;" />

不过开启 Devtools 有封号风险，最好专门注册一个新账号来使用。

之后我们逆向小程序就可以通过 JS 逆向的方式来分析了，下面了解一些 JS 逆向的基本知识。

### 调用堆栈

调用堆栈是 JavaScript 引擎用于管理函数调用顺序的一种数据结构（遵循“后进先出”原则）。

工作原理：

- 当函数被调用时，引擎会为其创建一个“执行上下文”并压入栈顶。

- 函数执行完毕后，其执行上下文从栈顶弹出，控制权回到之前的函数。

- 栈顶始终是当前正在执行的函数。

示例：

```js
function a() {
  console.log("a开始");
  b();
  console.log("a结束");
}

function b() {
  console.log("b开始");
  c();
  console.log("b结束");
}

function c() {
  console.log("c执行");
}

a();
```

1.  执行 `a()` → `a` 入栈 → 栈：`[a]` 
2.  `a` 中调用 `b()` → `b` 入栈 → 栈：`[a, b]` 
3.  `b` 中调用 `c()` → `c` 入栈 → 栈：`[a, b, c]` 
4.  `c` 执行完毕 → `c` 出栈 → 栈：`[a, b]` 
5.  `b` 执行完毕 → `b` 出栈 → 栈：`[a]` 
6.  `a` 执行完毕 → `a` 出栈 → 栈空 

调用堆栈可以帮我们追踪函数执行顺序，调试时可通过堆栈信息定位代码执行路径（如报错时的“堆栈跟踪”）。

### 调试

使用 Chrome 浏览器调试分析，F12，然后打开源代码/来源：

<img src="/assets/image-20250925173253505.png" alt="image-20250925173253505" style="zoom:50%;" />

作用域、调用堆栈、XHR断点这三个功能需要重点认识：

<img src="/assets/image-20250925173341154.png" alt="image-20250925173341154" style="zoom:50%;" />

我们输入账号密码-> 浏览器接受到我们的账号密码-> 根据js进行加密-> 发送到对方的服务器（网页）

服务器接收数据-> 对我们的数据进行解密-> 判断我们的数据解密后是否正常-> 正常就返回正常结果（否则返回报错）

抓包与数据分析。

<img src="/assets/20250819144531-19ac4efa-7cc8-1.png" alt="20250819144531-19ac4efa-7cc8-1" style="zoom:50%;" />

- 按钮1：让代码继续执行，运行到下一个断点中断执行，如果没有设置断点会直接运行完代码
- 按钮2：跳过下一个函数调用。既不遇到函数时，执行下一步；遇到函数时，不进入函数直接执行；
- 按钮3：跳进下一个函数上下文。即不遇到函数时，执行下一步；遇到函数时，进入函数上下文，查看函数具体内容
- 按钮4：步出，跳出当前函数调用
- 按钮5：单步，当前断点的下一步
- 按钮6：停用/激活全部断点

右键 top 选择在所有文件中搜索。

<img src="/assets/image-20250925174952273.png" alt="image-20250925174952273" style="zoom:50%;" />

根据登陆信息，在所有文件中搜索登陆结果关键字，然后打断点：

<img src="/assets/image-20250925175423178.png" alt="image-20250925175423178" style="zoom:50%;" />

之后再次点击登陆按钮执行断点。

<img src="/assets/image-20250925175617808.png" alt="image-20250925175617808" style="zoom:50%;" />

在作用域的闭包变量名中找到了 logindata。

<img src="/assets/image-20250925180315899.png" alt="image-20250925180315899" style="zoom:50%;" />

这样就可以成功把运行的 js 代码给断下来了。

## 参考

> [微信小程序-某某牛仔城 | xiaoeryu](https://xiaoeeyu.github.io/2024/09/16/微信小程序-某某牛仔城/)
> [微信小程序devtool抓包+反编译wxapkg | oacia = oaciaのBbBlog~ = DEVIL or SWEET](https://oacia.dev/wechat-mini-program-with-devtool)
> [手把手js逆向断点调试&js逆向前端加密对抗&企业SRC实战分享-先知社区](https://xz.aliyun.com/news/18630)