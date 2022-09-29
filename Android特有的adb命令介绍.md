# Android特有的adb命令介绍

adb（Android Debug Bridge，Android调试桥）是一种功能 多样的命令行工具，可用于与设备进行通信。adb命令可用于执行各 种设备操作。

常用的adb命令主要有以下几种。 

（1）adb shell dumpsys activity top 

功能：查看当前处于前台的Activity。

```
adb shell dumpsys activity top
```

该命令经常与grep命令一同使用。

（2）adb shell dumpsys package  

功能：查看包信息，包括四大组件信息以及MIME等相关信息。

```
adb shell dumpsys d
```

（3）adb shell dbinfo  功能：用于查看App使用的数据库信息，包括执行操作的查询语 句等信息都会被打印出来。

```
adb shell dumpsys dbinfo org.sfjolbyvukzzlpp
```

与lsof命令打印出来的性质一致。

（4）adb shell screencap -p  

功能：用于执行截图操作，并保存到目录下。 

（5）adb shell input text  

功能：用于在屏幕选中的输入框内输入文字，可惜不能直接输入 中文。 

（6）adb shell pm命令 

功能：pm命令是Android中packageManager的命令行，是用 于管理package的命令，比如通过pm list packages命令可以列出所 有安装的APK包名。

```
adb shell pm list packages
```

pm install命令用于安装APK文件，只是这里的APK文件不是在主机目录下，而是Android手机目录下。

```
adb shell pm install /sdcard/network-debug.apk
```

pm命令还有很多其他用法，比如pm uninstall命令，用于卸载 Android上的应用。

（7）adb shell am命令 

功能：am命令是一个重要的调试工具，主要用于启动或停止服 务、发送广播、启动Activity等。在逆向过程中，往往在需要以 Debug模式启动App时会使用这个命令。 对应命令格式为adb shell am start-activity -D -N <包名>/<类 名>。

```
adb shell am start-activity -D -N com.example.network/.MainActivity
```

其实以上所有命令也可以通过执行adb shell进入Android的 shell中直接执行，只需要将开始的adb shell去掉就行。以Debug模 式启动App为例，其结果是一样的。

还有一些主机和Android交互相关的adb命令，比如：

**adb install** ：这个命令用于将主 机的apk安装到手机上，其中App路径是主机的目录。

```
adb install ./network-debug.apk
```

**adb push和adb pull**：这两个命令用于在主机和Android设备之 间交换文件，前者用于从主机推送文件到Android设备上，后者 用于从Android设备上获取文件到主机中。

记得携带获取后存放的路径。

**adb logcat**：用于查看Android中日志的输出

