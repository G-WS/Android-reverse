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
