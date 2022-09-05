# AndroidFrida逆向与抓包实践

## 1.1虚拟机环境准备

使用Kali Linux：

[(72条消息) 超详细的最新版 2022.2 kali 安装步骤及拍摄快照的方法_小窦。的博客-CSDN博客_kali最新版](https://blog.csdn.net/weixin_51178129/article/details/126033729)

由于官网下载很慢，可以去镜像网站下载

[Index of /kali-images/kali-2022.3/](http://old.kali.org/kali-images/kali-2022.3/)

因为虚拟机本身时区不是东八区，需要重新设置时区。

启动虚拟机后，打开terminal，使用如下命令设置时区：

```sh
dpkg-reconfigure tzdata
在弹出的窗口中选择Asia-Shanghai
```

另外，kali Linux系统中默认不带中文，当打开中文网页或者抓包时，若数据包的数据中有中文，则无法解析，故需要配置kali使其支持中文，执行如下命令：

```
apt update

apt install xfonts-intl-chinese

apt install ttf-wqy-microhei
```

