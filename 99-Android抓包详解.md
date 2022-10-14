# Android抓包详解

## 抓包介绍

在Android APP的逆向分析中，抓包通常是指通过一些手段获取APP与服务器之间传输的铭文网络数据信息，这些网络数据信息往往是逆向分析的切入点，通过抓包得到的信息可以快速定位关键接口函数的位置，为调试代码逻辑提供了便利。

抓包主要分为两种：

- Hook抓包：Hook抓包实际上是指通过对发包函数的Hook来达到抓包的作用。
- 中间人抓包：指将原来一段完整的客户端-服务器通信方式割裂成两段客户端-服务器的通信。中间人的抓包在OSI七层网体模型的结构通常会被分成以下两种情况
  - 应用层：Http(s)协议抓包
  - 会话层：Socket通信抓包

## 工具推荐

Burpsuite：应用层Http(s)协议数据抓包

Charles：简单抓包，用的舒服轻松。

不推荐Fidder（因为Fidder无法导入客户端整数（p12,Client SSL Certificates））

会话层抓包推荐：

Charles

tcpdump+WireShark

## Http(S)协议抓包配置

### Http抓包配置

首先将计算机和测试手机连接在同一个局域网中，并且确保手机和计算机可以相互访问。

由于测试主机是虚拟机，首先关闭虚拟机，然后再虚拟机设置中把网络适配器的网络连接方式设置为桥接模式，需要注意的是

[(73条消息) kali虚拟机如何使用桥接模式连接外网_Cold_L i的博客-CSDN博客_kali桥接](https://blog.csdn.net/qq_40317852/article/details/120381805)

由于按照上述配置桥接后会导致物理机连不上网

最后决定从windows搭建抓包环境

电脑打开移动热点，用手机连接电脑的移动热点。

通过电脑ping通手机后

[Charles手机抓包 - 简书 (jianshu.com)](https://www.jianshu.com/p/551711c121f0)

再proxy setting中设置好端口

如果要对HTTPS抓包的话，还需要设置Proxy -> SSL Proxying Settings -> SSL Proxying -> Add，添加所有的域名和端口

点击Help再打开SSL Proxying 然后点击install Charles Root Certificates on aMobile Device or Remote Browser可以看到手机端安装证书提示。

长按连接网络，根据提示设置手动代理，在手机浏览器中访问chls.pro/ssl，然后下载证书，安装好证书，就可以抓包了。

仅仅将证书安装为用户信任的证书还不够我们可以通过以下命令将证书修改成系统自带的证书

```sh
C:\Users\33551>adb shell
bullhead:/ $ su

bullhead:/ # cd data/misc/user/0/cacerts-added/
bullhead:/data/misc/user/0/cacerts-added # ls
068ea8b5.0
bullhead:/data/misc/user/0/cacerts-added # mount -o remount,rw /system

1|bullhead:/data/misc/user/0/cacerts-added # cp * /etc/security/cacerts/

bullhead:/data/misc/user/0/cacerts-added # chmod 777 /etc/security/cacerts/*
bullhead:/data/misc/user/0/cacerts-added # mount -o remount,ro /system
bullhead:/data/misc/user/0/cacerts-added # reboot
```

为了配合手机上的代理设置，我们还需要在Charles上进行代理 设置。在依次单击Proxy→Proxy Settings→Enable SOCKS proxy 后单击OK按钮，就完成了SOCKS5模式的代理配置。此时重新启动 Charles的抓包，会发现比原先HTTP模式抓的包更多

## 应用层抓包核心原理

HTTPS协议的整个通信过程主要分成发起请求、验证身份、协商密 钥、加密通信阶段

![](https://github.com/G-WS/Android-reverse/blob/main/image/Https%E9%80%9A%E8%AE%AF%E8%BF%87%E7%A8%8B.png?raw=true)

首先客户端向服务器发起访问请求，服务器接收到客户端的请求后，便将服务器使用的证书公钥发给客户端，客户端通过本地配置的信任证书与服务器端传输的公钥进行比对，其中证书文件由第三方可信任机构颁发，此时，如果客户端校验是爱则会提示公钥不被信任，即会提示“您的连接不是私密连接”，如果客户端校验成功则会将客户端只用的证书公钥使用服务器传输的公钥加密后发送给服务器。服务器在接收到加密的客户端公钥后，会使用自己的私钥解密数据获取客户端的公钥进行加密后传输给客户端作为一段会话的标志。这就是HTTPS通信协议的密钥协商过程。后面通过sessionkey对通信数据进行加密。

> HTTPS=HTTP+加密+认证+完整性保护



在HTTPS上的应用层抓包原理也是基于中间人攻击的方式。 HTTPS上的应用层抓包原理主要“攻破”的是HTTPS传输过程中验 证身份的步骤，我们在配置抓包环境时是将Charles证书加入到系统 本身信任的证书中，当应用进行通信时，如果没有进一步的安全保护 措施，那么客户端接收到的服务器证书即使是Charles证书也会继续通信，整个过程可以表示如下：

![](https://github.com/G-WS/Android-reverse/blob/main/image/HTTPS%E4%B8%AD%E9%97%B4%E4%BA%BA%E6%8A%93%E5%8C%85.png?raw=true)

应对通过手动给系统安装证书从而导致中间人攻击继续生效的风险，App 也对这类攻击推出了对抗手段，主要有以下两种方式：

-  SSL Pinning，又称证书绑定，可以说是客户端校验服务器的进 阶版：该种方式不仅校验服务器证书是否是系统中的可信凭证，在通信过程中甚至连系统内置的证书都不信任而只信任App 指定的证书。一旦发现服务器证书为非指定证书即停止通信， 最终导致即使将Charles证书安装到系统信任凭据中也无法生效。 
- 服务器校验客户端。这种方式发生在HTTPS验证身份阶段，服务 器在接收到客户端的公钥后，在发送session key之前先对客户端的公钥进行验证，如果不是信任的证书公钥，服务器就中止 和客户端的通信。

逆向开发和分析人员对这些App对抗抓包 的手段也有了一些绕过的方式，比如在应对客户端校验服务器的情况 时，考虑到对证书校验的代码是写于App内部的，自然而然地就可以 通过Hook修改校验服务器的代码，从而使得判断的机制失效。在这 方面Objection本身可以通过以下命令完成SSL Pinning Bypass的功能：

```sh
android sslpinning disable
```

开源的项目DroidSSLUnpinning中添加了一部分Objection框架中所没有的Bypass证书校验，项目地址如下：

[WooyunDota/DroidSSLUnpinning: Android certificate pinning disable tools (github.com)](https://github.com/WooyunDota/DroidSSLUnpinning)

> 由于SSL Pinning的功能是由开发者自定义的，因此并不存在一 个通用的解决方案，Objection和DroidSSLUnpinning也只是对常见 的App所使用的网络框架中对证书进行校验的代码逻辑进行了Hook修 改。一旦App中的代码被混淆或者使用了未知的框架，这些App的客 户端校验服务器的逻辑就需要安全人员自行分析，不过上述两种方案 已几乎可以覆盖目前已知的所有种类的证书绑定

在应对服务器校验客户端的对抗手段中，服务器并不掌握在分析 人员手中，因此在中间人的状态下与服务器进行通信的实际上已经变 成抓包软件，比如Charles。通常来说，我们所能做的对抗手段就是 将App中内置的证书导入Charles中，使得服务端认为自己仍旧是在 与其信任的客户端进行通信，最终达到欺骗服务器的作用。 具体在操作过程中需要完成两项工作：第一，找到证书文件和相 应的证书密码（这个过程是需要逆向分析的，后续会以案例进行详细 的介绍）；第二，在找到证书和密码后将其导入抓包软件中，比如导 入 Charles 。 打 开 Charles ， 依 次 单 击 Proxy→SSL Proxy Settings→Client Certificates→Add，添加新的证书，输入指定的域名IP以及端口并导入p12格式或者pem格式的这个证书，之后即可将Charles伪装成使用特定证书的客户 端，最终达到正常抓包的目的。
