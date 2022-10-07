# Android常用Linux命令介绍

1、cat命令

功能：该命令用于在Shell中方便的查看文本文件的内容。

2、echo和touch命令 

功能：touch命令可以创建一个空文件；echo命令通过配合 “>”或者“>>”对文件进行写操作，其中“>”为覆盖写操作、 “>>”为扩展写操作。

```
echo "123" >> 2.txt
```

3、grep命令

功能：该命令用于在shell中过滤出符合条件的输出。

```
cat 3.txt |grep 123
```

4、ps命令 

功能：该命令可输出当前设备正在运行的进程。在Android 8之 后，ps命令只能打印出当前进程，需要加上-e参数才能打印出全部的 进程。

```
ps 
ps -e
```

5、netstat命令 

功能：该命令输出App连接的IP、端口、协议等网络相关信息， 通常使用的参数组合为-alpe。netstat -alpe用于查看所有sockets连 接的IP和端口以及相应的进程名和pid，配合grep往往有奇效。

```
netstat -alpe | grep org.sfjoldyvukzzlpp
```

6、lsof命令 

功能：该命令可以用于查看对应进程打开的文件

```
lsof -p 21312 -l |grep db
```

7、top命令 

功能：该命令用于查看当前系统运行负载以及对应进程名和一些 其他的信息，和之前讲的htop作用一样，只是相对来说htop更加人性 化



