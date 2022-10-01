# Frida环境搭建

## Frida环境搭建

Frida工作环境搭建十分简单

首先在计算机上的虚拟机kali中安装frida环境

```
pip install frida-tools
frida --version
```

如果需要在测试机上进行测试，还需要在测试机上安装和执行对应版本的server。

我们的测试机型是nexus 5X

首先执行getprop命令获取系统的架构。

```
getprop ro.product.cpu.abi
```

一般nexus 5X的架构是arm64-8a

我们只需要在Frida的GitHub主页 的release页面 `https://github.com/frida/frida/releases`下载和计算机版本相同的frida server。

下载完毕后，将frida-server通过adb工具推送到Android测试机上，在Android中，使用adb push命令推 送文件到data目录一般需要root权限，但是这里有一个例外，那就 是/data/local/tmp目录。所以，frida-server一般会被存放在测试 机的/data/local/tmp/目录下；在将frida-server存放到测试机目录 下后，使用chmod命令赋予frida-server充分的权限，这样frida-server就可以执行了。

```
frida-server-12.2.25-android-arm64.xz

#下载后的文件为该文件，先对该文件进行解压，解压后再采用adb push命令推送

adb push frida-server-12.2.25-android-arm64 /data/local/tmp/

#先用adb shell 命令进入手机调试模式

cd /data/local/tmp

#然后给frida-server文件设置可执行权限使其可以运行
chmod 755 frida-server-12.2.25-android-arm64

#运行该frida-server文件，注意需要通过root权限去运行
./frida-server-12.2.25-android-arm64

#然后再进行端口转发
adb forward tcp:27042 tcp:27042
adb forward tcp:27043 tcp:27043

#完成后就可以在kali中运行简单的frida命令测试是否安装成功
frida-ps -U
```

## Frida重启失败的问题

```
Unable to start: Could not listen on address 127.0.0.1, port 27042: Error binding to address 127.0.0.1:27042: Address already in use
```

通过 `netstat -tunlp`命令查询出端口进程id，然后通过`kill -9 [端口号]`杀死进程



## Frida 安装及 ERROR: Failed building wheel for frida 错误解决

```
# 方法一：
pip3 install frida-tools -i https://pypi.mirrors.ustc.edu.cn/simple/ 
# 方法二：
pip3 install frida -i https://pypi.mirrors.ustc.edu.cn/simple/ 

```

Frida 支持 python2 和 python 3，如果你使用的是较新版本的 Linux 发行版（例如 Ubuntu 18.0.4 / Kali 2019），由于自带 python 版本较高，可能会出现以下情况 ERROR: Failed building wheel for frida

```
ERROR: Command errored out with exit status 1:
     command: /usr/bin/python3 -u -c 'import sys, setuptools, tokenize; sys.argv[0] = '"'"'/tmp/pip-install-ritve202/frida/setup.py'"'"'; __file__='"'"'/tmp/pip-install-ritve202/frida/setup.py'"'"';f=getattr(tokenize, '"'"'open'"'"', open)(__file__);code=f.read().replace('"'"'\r\n'"'"', '"'"'\n'"'"');f.close();exec(compile(code, __file__, '"'"'exec'"'"'))' install --record /tmp/pip-record-44k2wbq5/install-record.txt --single-version-externally-managed --compile --install-headers /usr/local/include/python3.7/frida
         cwd: /tmp/pip-install-ritve202/frida/
    Complete output (13 lines):
    running install
    running build
    running build_py
    creating build
    creating build/lib.linux-x86_64-3.7
    creating build/lib.linux-x86_64-3.7/frida
    copying frida/__init__.py -> build/lib.linux-x86_64-3.7/frida
    copying frida/core.py -> build/lib.linux-x86_64-3.7/frida
    running build_ext
    looking for prebuilt extension in home directory, i.e. /root/frida-12.8.15-py3.7-linux-x86_64.egg
    prebuilt extension not found in home directory, will try downloading it
    querying pypi for available prebuilds
    error: <urlopen error [Errno 101] Network is unreachable>

```

关键错误代码：`i.e. /root/frida-12.8.15-py3.7-linux-x86_64.egg`，也就是说，极有可能是因为缺少相关文件。

在 `[frida · PyPI](https://pypi.org/project/frida/#files)`中下载对应的Linux版本`frida-12.8.15-py3.7-linux-x86_64.egg`

下载完成后，使用`easy-install`命令安装即可

```
python3 /usr/lib/python3/dist-packages/easy_install.py frida-12.8.16-py3.6-linux-x86_64.egg
```

## Frida IDE配置

在使 用Frida编写脚本时，有Frida的API智能提示功能是非常方便的。 Frida的作者非常体贴地提供了一个让VSCode、PyCharm等IDE支 持Frida的API智能提示的方式，具体配置方法如下：

```
curl -sL https://deb.nodesource.com/setup_18.x |bash -

apt-get install -y nodejs

node -v
npm -v

git clone https://github.com/oleavr/frida-agent-example.git

cd frida-agent-example/

npm install
```

最后在kali中安装Vscode

    sudo apt update
    sudo apt install curl gpg software-properties-common apt-transport-https
    
    导入Microsoft GPG密钥到您的Kali Linux:
    
    curl -sSL https://packages.microsoft.com/keys/microsoft.asc | sudo apt-key add -
    
    echo "deb [arch=amd64] https://packages.microsoft.com/repos/vscode stable main" | sudo tee /etc/apt/sources.list.d/vscode.list
    
    sudo apt update
    sudo apt install code
即可安装成功。