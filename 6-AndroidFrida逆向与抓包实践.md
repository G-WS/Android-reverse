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

首先，配置好adb环境

然后配置python的隔离，这里选用miniconda，因为pyenv工具的最后一步解决依赖包的问题一直很棘手，甚至可能会导致系统桌面环境崩溃

```
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh 
#下载安装脚本

#赋予安装脚本可执行的权限
chmod +x Miniconda3-latest-Linux-x86_64.sh

#运行安装脚本
sh Miniconda3-latest-Linux-x86_64.sh
```

运行最后一条命令后会先要求阅读Lisense阅读完毕后输入yes

在提示需要手动执行一次conda init命令时输入yes，才能真正将conda安装成功。

安装完后重启一次terminal，执行conda create -n py380 python=3.8.0命令安装指定版本的python，其中py380为安装时conda激活Python3.8.0之后的代称，若想使用特定的python版本，可以使用

```
conda activate py380
```

来激活对应的版本，在不需要使用该特定版本时，则可执行

```
conda deactivate
```

命令来退出

配置安装htop工具，它是加强版的top工具，htop可以动态查看当前活跃的，系统占用率高的进程

uptime是开机时间，load average是平均载荷

通过

```
apt-get install htop
```

命令安装

在终端中输入htop即可使用。

系统网络负载工具jnettop

通过

```
apt-get install jnettop
```

命令下载

该测试使用的是Nexus 5X作为测试机，首先打开测试机的设置，连续点击测试机的版本号，进入开发者模式，打开测试机的USB调试。

通过在terminal中的`adb devices`命令测试是否连接成功，成功与不成功的截图如下：

![](.\\image\adbdevices.png)
若图片查看不成功请转至该仓库的Image目录查看adbdevices.png图片

打开Bootloader锁:

打开OEM解锁

下载镜像包https://developers.google.cn/android/images?h1=zh%3Dcn#bullhead

然后刷入Android8.0的系统和TWRP获取root权限，由于magisic是一个假root，所以更推荐获取SuperSU的压缩文件通过twrp刷入，之后下载kali-nethunter的压缩包，通过twrp刷入系统，通过adb shell确认是否获取root权限，可以通过Nethunter的终端运行各种Android原本不支持的Linux命令。

如果因为手机太小，可以通过配置SSH，使用xshell去连接具体参看书本《安卓Frida逆向与抓包实战（陈佳林）》
