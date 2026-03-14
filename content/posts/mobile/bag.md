---
title: Android APP抓包入门
date: 2025-11-30 23:30:38
categories: Mobile
tags:
    - 抓包
mathjax:
top:
---


## 简介

常用工具：

- Yakit：和 burpsuite 差不多的抓包方式。
- Reqable：支持standalone、VPN模式，可绕过系统代理拦截 Flutter。
- OkHttp：OkHttp 是Android 最流行的网络库，抓包工具通过 hook 接口来抓包。
- r0Capture：hook 抓包通杀脚本。
- eCapture：内核级 hook 抓包，无需证书捕获 TLS 明文。

为了能够拦截 TLS/SSL 通信，我们需要我们的代理工具的证书才能被设备信任。通过 android 设置，我们可以轻松地将证书安装到“用户”CA 存储中。

只有在以下情况下，应用才信任用户证书：

1. 该应用以 Android 6（API 级别 23）或更低版本为目标平台
2. 或者[网络安全配置](https://developer.android.com/privacy-and-security/security-config)专门包括“用户”证书

Example which trusts "user" certificates：`xml/network_security_config.xml`

```xml
<base-config cleartextTrafficPermitted="false"> 
	<trust-anchors> 
		<certificates src="system" /> 
		<certificates src="user" /> 
	</trust-anchors> 
</base-config>
```

当应用程序发送明文流量时，我们可以认为这是一个安全问题。但是，只能通过证书拦截的加密流量并不是安全问题。通过安装 CA，我们故意削弱设备的安全性，以允许我们解密流量。

安装证书：

由于默认的网络安全配置规则，大多数应用仅新人系统证书。面向 Android 9 及更高版本的应用默认配置如下：

```xml
<base-config cleartextTrafficPermitted="false"> 
	<trust-anchors> 
		<certificates src="system" /> 
	</trust-anchors> 
</base-config>
```

## 防抓包检测

### 代理检测

代理检测是用于检测设备是否设置了网络代理。检测的目的是识别设备是否通过代理服务器来转发网络流量，避免被分析 App 的数据包。

在以下代码中 App 会检查系统设置或网络配置，以确定是否有代理服务器被设置为转发流量。例如，它可能会检查系统属性或调用特定的网络信息 API 来获取当前的网络代理状态。

```java
System.getProperty("http.proxyHost"); 
System.getProperty("http.proxyPort");
// 如果 proxyHost 或 proxyPort 不为空，说明设备设置了代理
```

- 强制不走代理：

如果在 App 的开发中直接通过代码强制网络流量不走代理，那么就无法通过代理来抓包。

```java
// HttpURLConnection 强制不走代理
connection = (HttpURLConnection) url.openConnection(Proxy.NO_PROXY);

// OkHttp 强制不走代理
OkHttpClient client = new OkHttpClient.Builder()
    .proxy(Proxy.NO_PROXY)
    .build();
```

- frida hook 绕过代理检测

通过 frida hook 系统类，修改`System.getProperty`使其返回空绕过代理检测。以下 hook 适用于 Java 层的代理检测。

```java
function anti_proxy() {
    var GetProperty = Java.use("java.lang.System");
    GetProperty.getProperty.overload("java.lang.String").implementation = function(getprop) {
        if (getprop.indexOf("http.proxyHost") >= 0 || getprop.indexOf("http.proxyPort") >= 0) {
            return null;
        }
        return this.getProperty(getprop);
    }
}

function main(){
	Java.perform(function(){
		anti_proxy();
	});
}
setImmediate(main);
```

使用：

这里仍然通过 Yakit 抓包，在没有

```
frida -U com.zj.wuaipojie -l ./hook_ssl.js
```


**透明代理**

透明代理（Transparent Proxy）是一种特殊的代理服务类型，它可以在客户端（如浏览器或应用程序）不知道的情况下拦截、转发和处理网络请求。与传统的代理服务不同，透明代理不需要客户端进行任何配置就能工作。

代理检测是用于检测设备是否设置了网络代理。这种检测的目的是识别出设备是否尝试通过代理服务器来转发网络流量，从而可能截获和分析App的网络通信。

App会检查系统设备或网络配置，以确定是否有代理服务器被设置为转发流量。例如，它可能会检查系统属性或调用特定的网络信息API来获取当前的网络代码状态。

```java
return System.getProperty("http.proxyHost") == null && System.getProperty("http.proxyHost") == null
//Port跟设置有关，例如Charles默认是8888
```

强制不走代理：

```java
connection = (HttpURLConnection) url.openConnection(Proxy.NO_PROXY);

OkHttpClient.Builder().proxy(Proxy.NO_PROXY).build()
```

绕过脚本：

通过 frida 进行 hook 绕过。

```js
function anti_proxy() {
    var GetProperty = Java.use("java.lang.System");
    GetProperty.getProperty.overload("java.lang.String").implementation = function(getprop) {
        if (getprop.indexOf("http.proxyHost") >= 0 || getprop.indexOf("http.proxyPort") >= 0) {
            return null;
        }
        return this.getProperty(getprop);
    }
}
```

透明代理（Transparent Proxy）是一种特殊的代理服务类型，它可以在客户端（如浏览器或应用程序）不知道的情况下拦截、转发和处理网络请求。与传统的代理服务不同，透明代理不需要客户端进行任何配置就能工作。
[[Clash版\]安卓上基于透明代理实现热点抓包](https://blog.seeflower.dev/archives/296/)
[安卓上基于透明代理对特定APP抓包](https://blog.seeflower.dev/archives/207/)

### VPN检测

VPN检测是指应用程序或系统检查用户是否正在使用虚拟专用网络（Virtual Private Network, VPN）的一种技术。当用户使用 VPN 时，他们的网络流量会被加密并通过一个远程服务器路由，这可以隐藏用户的实际IP地址和位置信息，同时保护数据的安全性和隐私。

当客户端运行 VPN 虚拟隧道协议时，会在当前节点创建基于`eth`之上的`tun0`接口或`ppp0`接口。这些接口是用于建立虚拟网络连接的特殊网络接口。

VPN 检测要比代理检测更加的底层，如果代理抓不到包我们可以通过 VPN 检测抓到包。

根据 OSI 七层模型，二者分别支持的协议:

| VPN  | OpvenVPN、IPsec、IKEv2、PPTP、L2TP、WireGuard等 |
| :--- | :---------------------------------------------- |
| 代理 | HTTP、HTTPS、SOCKS、FTP、RTSP等                 |

VPN 协议大多是作用在 OSI 的第二层和第三层之间，由此可见 VPN 能抓到代理方式的所有的包。

实现代码：

```java
public final boolean Check_Vpn1() {
        try {
            Enumeration<NetworkInterface> networkInterfaces = NetworkInterface.getNetworkInterfaces();
            if (networkInterfaces == null) {
                return false;
            }
            Iterator it = Collections.list(networkInterfaces).iterator();
            while (it.hasNext()) {
                NetworkInterface networkInterface = (NetworkInterface) it.next();
                if (networkInterface.isUp() && !networkInterface.getInterfaceAddresses().isEmpty()) {
                    Log.d("zj595", "isVpn NetworkInterface Name: " + networkInterface.getName());
                    if (Intrinsics.areEqual(networkInterface.getName(), "tun0") || Intrinsics.areEqual(networkInterface.getName(), "ppp0") || Intrinsics.areEqual(networkInterface.getName(), "p2p0") || Intrinsics.areEqual(networkInterface.getName(), "ccmni0")) {
                        return true;
                    }
                }
            }
            return false;
        } catch (Throwable th) {
            th.printStackTrace();
            return false;
        }
    }

 public final boolean Check_Vpn2() {
        boolean z;
        String networkCapabilities;
        try {
            Object systemService = getApplicationContext().getSystemService("connectivity");
            Intrinsics.checkNotNull(systemService, "null cannot be cast to non-null type android.net.ConnectivityManager");
            ConnectivityManager connectivityManager = (ConnectivityManager) systemService;
            NetworkCapabilities networkCapabilities2 = connectivityManager.getNetworkCapabilities(connectivityManager.getActiveNetwork());
            Log.i("zj595", "networkCapabilities -> " + networkCapabilities2);
            boolean z2 = networkCapabilities2 != null && networkCapabilities2.hasTransport(4);
            // 检查网络能力是否包含 "WIFI|VPN" 
            if (networkCapabilities2 != null && (networkCapabilities = networkCapabilities2.toString()) != null) {
                if (StringsKt.contains$default((CharSequence) networkCapabilities, (CharSequence) "WIFI|VPN", false, 2, (Object) null)) {
                    z = true;
                    return !z || z2;
                }
            }
            z = false;
            if (z) {
            }
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
    }

```

绕过脚本：

```js
function hook_vpn() {
    Java.perform(function () {
        var NetworkInterface = Java.use("java.net.NetworkInterface");
        NetworkInterface.getName.implementation = function () {
            var name = this.getName();  //hook java层的getName方法
            console.log("name: " + name);
            if (name === "tun0" || name === "ppp0") {
                return "rmnet_data0";
            } else {
                return name;
            }
        }

        var NetworkCapabilities = Java.use("android.net.NetworkCapabilities");
        NetworkCapabilities.hasTransport.implementation = function () {
            return false;
        }

        NetworkCapabilities.appendStringRepresentationOfBitMaskToStringBuilder.implementation = function (sb, bitMask, nameFetcher, separator) {
            if (bitMask == 18) {
                console.log("bitMask", bitMask);
                sb.append("WIFI");
            } else {
                console.log(sb, bitMask);
                this.appendStringRepresentationOfBitMaskToStringBuilder(sb, bitMask, nameFetcher, separator);
            }
        }

    })
}
```

### pinning之指纹

SSL Pinning也称为证书锁定，是 Google 官方推荐的检验方式，意思是将服务器提供的 SSL/TLS 证书内置到移动客户端，当客户端发起请求的时候，通过对比内置的证书与服务器的证书是否一致，来确认这个连接的合法性。

> 这里还要提到一个概念:`单向校验`，本质上二者没区别，`SSL Pinning`可以理解为加强版的`单向校验

<img src="https://pic.rmb.bdstatic.com/bjh/240818/3e7d68353ecaa9c62720955d1d230ea61665.png" alt="图片" style="zoom: 50%;" />

`SSL Pinning`主流的三套方案:`公钥校验`、`证书校验`、`Host校验`
因为是客户端做的校验，所以可以在本地进行hook对抗，参考以下的两个项目:
[JustTrustMe](https://github.com/Fuzion24/JustTrustMe)、[sslunpining](https://github.com/ac-pm/SSLUnpinning_Xposed)

常见的网络框架：

- Retrofix
- OkHttp



### pinning之证书

### 双向认证

双向验证，就是客户端需要验证服务端证书的正确性，服务端也需要验证客户端的证书正确性。

<img src="https://pic.rmb.bdstatic.com/bjh/240825/3f61fa813672068bbdd42947d4d6d6907655.png" alt="图片" style="zoom:50%;" />

- 实现方案

首先借助 openssl 生成服务器证书。

```shell
# 生成CA私钥 
openssl genrsa -out ca.key 2048

# 生成CA自签名证书 
openssl req -x509 -new -nodes -key ca.key -sha256 -days 1024 -out ca.crt
```

生成服务端证书：

```shell
openssl genrsa -out server.key 2048
```

这个指令生成一个 2048 位的 RSA 私钥，并将其保存到名为 server.key 的文件中。

```shell
openssl req -new -key server.key -out server.csr -config server_cert.conf
```

这个指令基于第一步生成的私钥创建一个新的证书签名请求（CSR）。CSR包含了公钥和一些身份信息，这些信息在证书颁发过程中用于识别证书持有者。`-out server.csr`指定了CSR的输出文件名。
执行这个指令时，系统会提示你输入一些身份信息，如国家代码、组织名等，这些信息将被包含在CSR中。(我们这边测试直接全部按回车键默认即可)

| 字段名称                               | 描述                                                   | 默认值/示例值 | 是否必填 |
| :------------------------------------- | :----------------------------------------------------- | :------------ | :------- |
| Country Name (2 letter code)           | 国家代码，两位字母代码。                               | AU            | 否       |
| State or Province Name                 | 州或省份的全名。                                       |               | 否       |
| Locality Name (eg, city)               | 城市或地区名称。                                       |               | 否       |
| Organization Name                      | 组织名称，通常是公司或机构的名称。                     |               | 否       |
| Organizational Unit Name (eg, section) | 组织单位名称，可以是部门或团队的名称。                 |               | 否       |
| Common Name (CN)                       | 完全限定的域名（FQDN）或个人名称，用于标识证书持有者。 |               | 是       |
| Email Address                          | 与证书持有者关联的电子邮件地址。                       |               | 否       |
| Challenge Password                     | 挑战密码，用于CSR的额外安全措施。                      |               | 否       |
| Optional Company Name                  | 可选的公司名称字段。                                   |               | 否       |

`-config server_cert.conf`创建一个OpenSSL配置文件（如 `server_cert.conf`）并指定IP地址，具体的ip地址可以由ipconfig获取

```
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
CN = 192.168.199.108

[v3_req]
subjectAltName = @alt_names

[alt_names]
IP.1 = 192.168.199.108
```

```shell
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -days 365 -extfile server_cert.conf -extensions v3_req
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.cer
```

使用CA证书签发服务器证书。
生成cer证书供服务端验证。

客户端证书：

```shell
openssl genrsa -out client.key 2048
openssl req -new -out client.csr -key client.key
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out client.crt -days 500 -sha256
```

生成客户端带密码的p12证书（这步很重要，双向认证的话，浏览器访问时候要导入该证书才行；可能某些Android系统版本请求的时候需要把它转成bks来请求双向认证）：

```
openssl pkcs12 -export -out client.p12 -inkey client.key -in client.crt -certfile ca.crt
```

到这一步的时候，设置密码和验证密码光标不会显示，直接输入即可

- dump内置证书

```js
function hook_KeyStore_load() {
    Java.perform(function () {
        var ByteString = Java.use("com.android.okhttp.okio.ByteString");
        var myArray=new Array(1024);
        var i = 0
        for (i = 0; i < myArray.length; i++) {
            myArray[i]= 0x0;
         }
        var buffer = Java.array('byte',myArray);
        var StringClass = Java.use("java.lang.String");
        var KeyStore = Java.use("java.security.KeyStore");
        KeyStore.load.overload('java.security.KeyStore$LoadStoreParameter').implementation = function (arg0) {
            console.log(Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()));
            console.log("KeyStore.load1:", arg0);
            this.load(arg0);
        };
        KeyStore.load.overload('java.io.InputStream', '[C').implementation = function (arg0, arg1) {
            console.log(Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()));
            console.log("KeyStore.load2:", arg0, arg1 ? StringClass.$new(arg1) : null);
            if (arg0){
                var file =  Java.use("java.io.File").$new("/data/user/0/com.zj.wuaipojie/files/client"+".p12");
                var out = Java.use("java.io.FileOutputStream").$new(file);
                var r;
                while( (r = arg0.read(buffer)) > 0){
                    out.write(buffer,0,r)
                }
                console.log("证书保存成功!")
                out.close()
            }
            this.load(arg0, arg1);
        };
    });
}
```



## Yakit抓包

研究一下最近很火的 Yakit 怎么抓真机 App 的包。

这里电脑和真机处于同一无线网络下。

### 配置

点击 Wifi 网络的右上角的编辑。然后在代理中设置手动代理，设置为电脑的无线网卡地址。

<img src="/assets/Pasted%20image%2020250721213325.png" style="zoom: 50%;" />


安装 Yakit 证书，进入高级配置然后选择证书下载。

<img src="/assets/Pasted%20image%2020250721214047.png" style="zoom: 50%;" />

将证书导入到手机中，比如直接通过 USB 共享文件复制过去。

进入手机设置->安全和隐私->更多安全设置->加密与凭据->安装证书。

<img src="/assets/Pasted%20image%2020250721214533.png" style="zoom: 50%;" />


然后选择我们导入到手机的证书文件，点击然后选择证书安装程序。

<img src="/assets/Pasted%20image%2020250721214343.png" style="zoom: 50%;" />

### 验证

手机打开浏览器，进入百度随便一个网址。

<img src="/assets/Pasted%20image%2020250721223410.png" style="zoom: 50%;" />


可以看到 Yakit 已经抓到了数据包。

<img src="/assets/Pasted%20image%2020250721223443.png" style="zoom: 80%;" />


## Reqable抓包

Reqable 使用经典的中间人（MITM）技术分析 HTTPS 流量，当客户端与 Reqable 的代理服务器（下文简称中间人）进行通信时，中间人需要重签远程服务器的 SSL 证书。为了保证客户端与中间人成功进行 SSL 握手通信，需要将中间人的根证书（简称CA根证书）安装到客户端的证书管理中心。

Reqable 可以自动绕过对于代理的检测以及对于 VPN 的检测。

由于 Flutter 网络通信通常绕过系统代理，很多传统抓包工具无法抓取其通信，需要 Reqable VPN 模式。

### 桌面端配置

在桌面端启动 Reqable，选择证书->安装根证书到本机

![](/assets/Pasted%20image%2020250525233140.png)

安装成功

<img src="/assets/Pasted%20image%2020250525233222.png" style="zoom: 50%;" />

### 手机端配置

手机端同样也要安装证书，选择证书->安装根证书到Android设备，然后在左下角点击导出证书。其余安装流程和 Yakit 差不多，这里就不过多赘述了。

<img src="/assets/Pasted%20image%2020250724093101.png" style="zoom: 50%;" />

手机端进入协同模式

<img src="/assets/Pasted%20image%2020250525233832.png" style="zoom: 50%;" />

扫描桌面端的二维码。

<img src="/assets/Pasted%20image%2020250525233909.png" style="zoom: 50%;" />


点击 ip 旁的手机图标，唤出二维码，然后扫描。

<img src="/assets/Pasted%20image%2020250525233953.png" style="zoom: 50%;" />

点击右下角的按钮开始抓包。

![](/assets/Pasted%20image%2020250525234051.png)

允许。

<img src="/assets/Pasted%20image%2020250525234149.png" style="zoom: 50%;" />


<img src="/assets/Pasted%20image%2020250525234213.png" style="zoom: 50%;" />

### 抓包

然后就可以对目标 APP 进行抓包，我们可以选择特定 APP 进行筛选。
<img src="/assets/Pasted%20image%2020250724233401.png" style="zoom: 50%;" />

## Hook抓包

Hook 抓包是一种截取应用程序数据包的方法，通过 Hook 应用或系统函数来获取数据流。在应用层 Hook 时，通过查找触发请求的函数来抓包，优点是不受防抓包手段影响，缺点是抓包数据不便于我们分析和筛选。

可以绕过诸如代理检测、VPN检测、明文解密、可以抓 OkHttp、可以抓 Flutter。

### OkHttp

常见安卓网络开发框架：

| 框架名称               | 描述                                                                               |
| ------------------ | -------------------------------------------------------------------------------- |
| Volley             | 由 Google 开源的轻量级网络库，支持网络请求处理、小图片的异步加载和缓存等功能                                       |
| Android-async-http | 基于 Apache HttpClient 的一个异步网络请求处理库                                                |
| xUtils             | 类似于 Afinal，但被认为是 Afinal 的一个升级版，提供了 HTTP 请求的支持                                    |
| OkHttp             | 一个高性能的网络框架，已经被 Google 官方认可，在 Android 6.0 中底层源码已经使用了 OkHttp 来替代 HttpURLConnection |
| Retrofit           | 提供了一种类型安全的 HTTP 客户端接口，简化了 HTTP 请求的编写，通常与 OkHttp 配合使用                             |

拦截器是 OkHttp 提供的对 Http 请求和响应进行统一处理的强大机制，它可以实现网络监听、请求以及响应重写、请求失败充实等功能。  
OkHttp 中的 Interceptor 就是典型的责任链的实现，它可以设置任意数量的 Intercepter 来对网络请求及其响应做任何中间处理，比如设置缓存，Https证书认证，统一对请求加密/防篡改社会，打印log，过滤请求等等。  OkHttp 中的拦截器分为 Application Interceptor（应用拦截器） 和 NetWork Interceptor（网络拦截器）两种。

这里我们学习如何通过 OkHttpLogger-Frida 项目对 OkHttp 框架进行抓包。

函数：
- `find()`：
- `hold()`：

将`okhttpfind.dex`文件复制到`/data/local/tmp`目录下，并设置可执行权限。

```shell
adb push ./okhttpfind.dex /data/local/tmp
adb shell
chmod 777 /data/local/tmp/okhttpfind.dex
```

官方使用示例：

![](/assets/screen.gif)


基本使用：

```shell
frida -U woaipojie -l okhttp_poker.js 
hold()
```


可以看到成功抓到了数据包！

![](/assets/Pasted%20image%2020250724212929.png)


### r0Capture

如果 APP 不是使用 OkHttp 框架开发的，或者说它的混淆非常强导致你无法定位到它的发包函数。那么可以使用 r0Capture 这个工具。

r0Capture 是一个基于 Firda 的 Android 抓包框架脚本，可以通杀 Android 上绝大多数应用层流量协议，并绕过各种证书校验、加固保护、网络层劫持检测，用于动态分析和安全测试。

- 仅限安卓平台，测试安卓 7-14 可用；
- 无视所有证书校验或绑定；
- 通杀 TCP/IP 四层模型中的应用层中的全部协议；
- 通杀协议包括：Http,WebSocket,Ftp,Xmpp,Imap,Smtp,Protobuf等等、以及它们的SSL版本；
- 通杀所有应用层框架，包括HttpUrlConnection、Okhttp1/3/4、Retrofit/Volley等等；
- 无视加固，不管是整体壳还是二代壳或VMP，不用考虑加固的事情；但是对于部分大厂的自定义的 SSL 框架则不支持。

用法：

- Spawn 模式（启动 App 前 hook）

```shell
python3 r0capture.py -U -f com.coolapk.market -v
```

- Attach 模式（运行中注入，抓包保存为 pcap）

```shell
python3 r0capture.py -U 酷安 -v -p iqiyi.pcap
```

- `-U`：连接 USB 真机
- `-f`：指定包名（旧版 Frida）
- `酷安`：新版 Frida 用 App 名（通过 `frida-ps -U` 查看）
- `-p`：保存为 pcap 文件，便于后续 Wireshark 分析

我们可以将保存的 pcap 文件通过 Wireshark 进行详细分析。

r0Capture 虽然适配了大多数的环境，但是对于一些大厂自定义的框架仍然是没有支持的。

### eCapture

eCapture 是一个基于内核机制的抓包工具，利用了 eBPF 和 Hook 技术实现内核层的网络抓包。eBPF 是一个运行在 Linux 内核里面的虚拟组件，它可以无需改变内核代码或者加载内核模块的情况下，安全而又高效地扩展内核的功能。

<img src="/assets/Pasted%20image%2020250724142445.png" style="zoom: 80%;" />

eCapture 的工作原理涉及到用户态和内核态。用户态就是运行应用程序的地方，比如各种 App。在这个区域中，eCapture 通过一个共享的模块获取应用程序的网络数据。然后，它将这些数据传递给内核态的 eBPF 程序进行分析和处理。在内核空间，eCapture 通过 eBPF 插件捕捉网络层的数据流，比如数据包是从哪里来的、发到了哪里去。这一过程不需要修改应用程序本身，所以对系统性能影响很小。

安卓设备的内核版本只有在 5.10 版本上才可以进行无任何修改的开箱抓包操作。

通过以下命令查看系统内核版本：

```shell
adb shell uname -r
```

使用 eCapture 可以在不配置 CA 证书的情况下也能捕获到 https/tls 的通讯明文。

- 环境搭建

```shell
#下载文件
adb push ./ecapture /data/local/tmp
chmod +x ./ecapture
```

- 基本使用

捕获 TLS 明文（text模式）

```shell
./ecapture tls -m text
```

捕获 TLS 握手密钥（keylog 模式）

```shell
./ecapture tls -m keylog --keylogfile=openssl_keylog.log
```

该文件可配合 Wireshark 设置 TLS 解密，路径在：  `Edit > Preferences > Protocols > TLS > (Pre)-Master-Secret log filename`

保存为 pcap 文件（pcap 模式）

```shell
./ecapture tls -m pcapng -i eth0 --pcapfile=ecapture.pcapng
```

然后可用 Wireshark 打开 `.pcapng` 文件，看到**明文数据包**。

```shell
adb shell ps | findstr 应用包名（获取进程pid）
./ecapture tls -p pid -m text
```

通过 eCaptureQ 联合抓包：

```shell
./ecapture tls --ecaptureq ws://0.0.0.0:28257
```

设置手机的 ip 地址：

<img src="/assets/image-20251122195024038.png" alt="image-20251122195024038" style="zoom:50%;" />

然后重启下之后，在手机中打开要抓包的 APP 即可抓到包。

<img src="/assets/image-20251122195654110.png" alt="image-20251122195654110" style="zoom:50%;" />


## 参考

>[Android HTTPS认证的N种方式和对抗方法总结](https://ch3nye.top/Android-HTTPS%E8%AE%A4%E8%AF%81%E7%9A%84N%E7%A7%8D%E6%96%B9%E5%BC%8F%E5%92%8C%E5%AF%B9%E6%8A%97%E6%96%B9%E6%B3%95%E6%80%BB%E7%BB%93/)
>[吾爱破解安卓逆向入门教程《安卓逆向这档事》二十二课、一饭三吃(抓包版)](https://www.bilibili.com/video/BV17AUsYZE8b/?share_source=copy_web&vd_source=a1e0ec33522dd06cf97338beeab421c6)
>[httptoolkit/frida-interception-and-unpinning: Frida scripts to directly MitM all HTTPS traffic from a target mobile application](https://github.com/httptoolkit/frida-interception-and-unpinning)
>[gojue/ecapture：使用 eBPF 捕获没有 CA 证书的 SSL/TLS 明文。在 amd64/arm64 的 Linux/Android 内核上受支持。](https://github.com/gojue/ecapture?tab=readme-ov-file)