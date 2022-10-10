# Xposed框架简述

## 简介

是一种比Frida出现更早的Hook框架，Xposed是一套开源的、在Android root模式下运行的框架，可以在不修改App源代码的情况下，通过Hook方式去影响程序的运行。与Frida一样，Xposed广泛用于 Android安全测试中，不管是针对特定API的监控、恶意代码的分析还 是对系统功能的自定义，Xposed框架都能够以对Java代码Hook的方 式游刃有余地完成。可惜的是，Xposed最后的发行版是v89，发行时 间是2017年12月18日。

查看相应的代码提交记录，这个版本适配的是Nougat代号的 Android，通过Android官网提供的“代号、标记和Build号”对照表 可知这个代号对应的Android版本为Android 7.1和7.0。

虽然Xposed框架已经很久远了，但是基于Xposed框架开发的其他框架依然活跃，就是稳定性值得商榷。

## 框架安装

安装雷电模拟器（Android 7.1版本），在雷电游戏中心下载xposed框架，然后打开，可以发现没有激活

然后下载一个压缩包framework-xposed-v89-sdk25-x86.zip

这个压缩包我放在了同仓库的资源文件夹下面，

下载成功后进行解压。

新建一个文件夹xposed，将上面的解压后的文件夹中的的文件复制到该文件夹中。接下来输入adb命令：

```
adb mount
adb push xposed /system 

adb shell
su
cd /system/xposed
mount -o remount -w /system
sh script.sh
```

然后重启一下雷电模拟器，打开xposed，就会发现已经激活了。

## Xposed插件安装脚本编写

Xposed模块从本质上来讲也是一个Android程序，因此编写Xposed模块也可以像编写Android程序一样使用Android Studio进行开发，要成功地使Xposed将Android Studio开发程序识别未Xposed模块，还需要特别配置一些文件。

Hook的目标程序的MainActivity代码如下：

```java
package com.example.reversedemo2;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.util.Log;

import java.util.Locale;

public class MainActivity extends AppCompatActivity {
    private String total = "hello";
    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        while(true){
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            fun(50,20);
            Log.d("reverseDemo.string", fun("LowerRcAse Me!!!"));
        }
    }
    void fun(int x,int y){
        Log.d("reverseDemo", String.valueOf(x+y));
    }
    String fun(String x){
        return x.toLowerCase();
    }
    void secret(){
        total+=" secretFunc";
        Log.d("reverseDemo.secret", "secret: this is secret func");
    }
    static void staticSecret(){
        Log.d("reverseDemo.Secret", "this is staticSecret func");
    }
}

```

选定的Hook的目标函数未String fun(String x)

**首先**重新新建一个Android项目，选在EmptyActivity作为模板，成功创建后切换到工程试图后，修改AndroidManifest.xml文件在`<activity Android:name=".MainActivity">`标签前添加如下代码：

```xml
<meta-data
            android:name="xposedmodule"
            android:value="true"/>
        <meta-data
            android:name="xposeddescription"
            android:value="这是一个Xposed插件"/>
        <meta-data
            android:name="xposedminversion"
            android:value="53"/>
```

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.example.xposeddemo">

    <application
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.XposedDemo"
        tools:targetApi="31">
        
        <meta-data
            android:name="xposedmodule"
            android:value="true"/>
        <meta-data
            android:name="xposeddescription"
            android:value="这是一个Xposed插件"/>
        <meta-data
            android:name="xposedminversion"
            android:value="53"/>
        <activity
            android:name=".MainActivity"
            android:exported="true">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>

</manifest>
```

name 为xposedmodule的meta-data标签所对应的value为true，即会将应用标记为Xposed模块；name为xposeddescription的meta-data 标签所对应的value为Xposed模块的描述。xposedminversion所对 应的是Xposed模块支持的最低版本。

将此项目运行后即可将该程序安装到手机上，这个时候可以在XposedInstaller应用中选择模块，即可看到这个Xposed模块。

然后我们编译安装的App已经可以被Xposed识别为Xposed模块。但是这时该Xposed模块还是一个空架子，实际内容需要我们去填充。

**第二步**在Android中引入第三方包含有Xposed的API的jar包XposedBridge.jar，使得我们编写的XposedDemo能够实现下一步的Hook操作。为了实现这一点，需要编辑build.gradle(Module)文件，添加如下代码：

```
repositories {
    jcenter()
}
```

然后再dependencies节点中添加如下代码

```java
compileOnly 'de.robv.android.xposed:api:82'
    compileOnly 'de.robv.android.xposed:api:82:sources'
```

> 记录一次错误，以上的方法有问题，jcenter已经被弃用，
>
> 需要用另一种方法加入该库

正确的方法：

将XposedBridgeApi-54.jar拷贝到libs目录下

然后再dependencies中添加

```java
    compileOnly fileTree(dir: 'libs',includes: ['*.jar'])

```

点击sync即可

**第三步、编写真正的Hook代码**

编写Hook脚本

```java
package com.example.xposeddemo;

import de.robv.android.xposed.IXposedHookLoadPackage;
import de.robv.android.xposed.XC_MethodHook;
import de.robv.android.xposed.XposedBridge;
import de.robv.android.xposed.XposedHelpers;
import de.robv.android.xposed.callbacks.XC_LoadPackage;

public class XposedHookDemo implements IXposedHookLoadPackage {

    @Override
    public void handleLoadPackage(XC_LoadPackage.LoadPackageParam loadPackageParam) throws Throwable {
        if (loadPackageParam.packageName.equals("com.example.reversedemo2")){
            XposedBridge.log(loadPackageParam.packageName+"has Hooked!");
            Class clazz = loadPackageParam.classLoader.loadClass("com.example.reversedemo2.MainActivity");
            XposedHelpers.findAndHookMethod(clazz, "fun", String.class, new XC_MethodHook() {
                @Override
                protected void beforeHookedMethod(MethodHookParam param) throws Throwable {
                    super.beforeHookedMethod(param);
                    XposedBridge.log("input:" +param.args[0]);
                }

                @Override
                protected void afterHookedMethod(MethodHookParam param) throws Throwable {
                    param.setResult("you has been hijacked");
                }
            });
        }
    }
}

```

> 在这段代码中，Xposed是通过IXposedHookLoadPackage接口 中的handleLoadPackage方法来实现Hook并篡改程序输出结果的。 在handleLoadPackage函数中，由于Xposed的Hook默认是针对整 个系统的，因此需要先通过loadPackageParam.packageName去过 滤 目 标 包 名 （ 目 标 App 为 第 2 章 中 的 demo02 程 序 ， 包 名 为 “com.roysue.demo02”）。 XposedBridge.log()是Xposed自己实现的打印日志log的函数。 接下来通过loadPackageParam.classLoader.loadClass()函数 实现对目标类对象的获取，等价于Frida脚本中所使用的Java.use()函 数 。 在 获 取 到 类 对 象 后 ， 通 过 XposedHelpers.findAndHookMethod()函数寻找并对指定函数进行 Hook。其中，第一个参数为之前所获取到的clazz类对象，第二个参 数为目标函数名，第二个参数之后的参数则是目标函数的参数列表。 由于Hook的目标函数只有一个String类型的参数，因此这里写的是 String.class。这个函数的最后一个参数是对目标函数的一个Hook回 调（callback），这个回调固定为XC_MethodHook。 在 这 个 XC_MethodHook 回 调 中 ， 需 要 实 现 beforeHookedMethod()和afterHookedMethod()两个回调函数，其 中beforeHookedMethod()函数可以通过参数param获取被Hook函 数的参数值，通常被用于在目标函数执行前获取和更改被Hook函数 的参数。在代码清单6-13中，只是演示获取了被Hook函数的参数值 并 使 用 XposedBridge.log() 把 参 数 值 打 印 出 来 。 afterHookedMethod()函数通常被用于获取和更改被Hook函数的返 回 值 ， 这 里 直 接 将 函 数 的 返 回 值 修 改 为 “You has been hijacked”。

最后在XposedDemo中添加Xposed模块的入口点，是的Xposed框架知道从哪个函数执行Hook。

右键点击main文件夹，再一次选择New->Folder->Asserts Floder，在弹出的窗口中直接单机Finish按钮完成assets文件夹的创建

在该文件夹下新建一个xposed_init文件

加入Hook类的路径

```
com/example/xposeddemo/XposedHookDemo
```

安装到测试机上，在XposedInstaller中管理模块的页面上勾选我们编写的XposedDemo。重启手机后就会生效，再次执行reverseDemo02就可以发现打印的日志内容变成了之前设置的“you have been  hijacked”.

与Frida脚本的相比的优点，Xposed本身使用Java语法开发，开发体验十分流畅，而Frida却打破了Java和js的壁垒，需要编写脚本将Java转化为JavaScript，实际操作过程存在一个翻译的过程。

与Frida脚本相比的缺点：热更新能力差，每次修改模块内容之后需要重启手机，而Frida可以及时更新，甚至不需要重启运行脚本。Xposed框架不支持Native函数的Hook，需要配合一些其他Hook框架实现，Frida本身支持对Native层进行Hook。