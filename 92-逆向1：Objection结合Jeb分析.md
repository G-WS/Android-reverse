# 逆向1：Objection结合Jeb分析

分析目标为之前的一个2022羊城杯的ctf的APP：BBButton.apk

软件我会放到APK文件夹下。

使用Jadx将App拖入分析，在Jadx初步分析完毕后，使用Jadx 查看AndroidManifest.xml内容。AndroidManifest.xml在资源文件目录下。

这个文件中会记载APP相关的属性，包括APP的包名、版本号、所申请的权限信息、四大组件信息以及入口类等信息。文件如下：

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android" android:versionCode="1" android:versionName="1.0" android:compileSdkVersion="30" android:compileSdkVersionCodename="11" package="com.crackme.bbbbutton" platformBuildVersionCode="30" platformBuildVersionName="11">
    <uses-sdk android:minSdkVersion="16" android:targetSdkVersion="30"/>
    <application android:theme="@style/Theme.BBBButton" android:label="@string/app_name" android:icon="@mipmap/ic_launcher" android:allowBackup="true" android:supportsRtl="true" android:roundIcon="@mipmap/ic_launcher_round" android:appComponentFactory="androidx.core.app.CoreComponentFactory">
        <activity android:name="com.crackme.bbbbutton.MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>
</manifest>
```

APP 的包名为com.crackme.bbbbutton，这是进程唯一的标志。

根据activity、service等标签可以确定APP的四大组件信息。

> 该APP只有一个activity，因此这个MainActivity就是入口类，否则需要通过intent-filter标签中的action属性信息和category属性去标志入口类。
>
> 入口类的action属性信息为android.intent.action.MAIN,而相应的category属性名则为android.intent.category.LAUNCHER。

查看入口类的代码，在Jadx界面左部选中并双击MainActivity 后，Jadx会自动显示翻译出的MainActivity类的内容

```java
package com.crackme.bbbbutton;

import android.annotation.SuppressLint;
import android.app.Activity;
import android.os.Bundle;
import android.view.View;
import android.widget.Button;
import android.widget.LinearLayout;
import android.widget.TextView;
import android.widget.Toast;
import java.util.Arrays;

/* loaded from: classes.dex */
public class MainActivity extends Activity implements View.OnClickListener {

    /* renamed from: c  reason: collision with root package name */
    public int f872c;
    public Button e;
    public TextView f;

    /* renamed from: b  reason: collision with root package name */
    public byte[] f871b = new byte[100];
    public int d = 16;

    public static byte[] getBytes(byte[] bArr) {
        byte[] bArr2 = new byte[bArr.length / 4];
        int i = 0;
        int i2 = 0;
        while (i < bArr.length / 4) {
            int i3 = i * 4;
            bArr2[i2] = (byte) ((bArr[i3] << 6) + (bArr[i3 + 1] << 4) + (bArr[i3 + 2] << 2) + bArr[i3 + 3]);
            i++;
            i2++;
        }
        return bArr2;
    }

    public final boolean a() {
        if (this.f872c == this.f871b.length) {
            Toast.makeText(this, "too many!\nplz reset!", 0).show();
            return false;
        }
        return true;
    }

    public final void b(View view) {
        if (a()) {
            Toast.makeText(this, "button1", 0).show();
            byte[] bArr = this.f871b;
            int i = this.f872c;
            this.f872c = i + 1;
            bArr[i] = 0;
        }
    }

    public final void c(View view) {
        if (a()) {
            Toast.makeText(this, "button2", 0).show();
            byte[] bArr = this.f871b;
            int i = this.f872c;
            this.f872c = i + 1;
            bArr[i] = 1;
        }
    }

    public final void d(View view) {
        if (a()) {
            Toast.makeText(this, "button3", 0).show();
            byte[] bArr = this.f871b;
            int i = this.f872c;
            this.f872c = i + 1;
            bArr[i] = 2;
        }
    }

    public final void e(View view) {
        if (a()) {
            Toast.makeText(this, "button4", 0).show();
            byte[] bArr = this.f871b;
            int i = this.f872c;
            this.f872c = i + 1;
            bArr[i] = 3;
        }
    }

    public final void f(View view) {
        String str;
        if (this.f872c >= 16) {
            byte[] copyOfRange = Arrays.copyOfRange(this.f871b, 0, this.d);
            byte[] copyOfRange2 = Arrays.copyOfRange(this.f871b, this.d, this.f872c);
            Toast.makeText(this, "checking...", 0).show();
            if (JniCheck.check1(copyOfRange) && JniCheck.check2(copyOfRange, copyOfRange2)) {
                str = "DASCTF{" + new String(getBytes(copyOfRange2)) + "}";
                h(str);
            }
        }
        str = "failed!";
        h(str);
    }

    public final void g(View view) {
        Toast.makeText(this, "reset", 0).show();
        Arrays.fill(this.f871b, (byte) 0);
        this.f872c = 0;
    }

    public final void h(String str) {
        TextView textView = new TextView(this);
        this.f = textView;
        textView.setGravity(1);
        this.f.setText(str);
        this.f.setTextColor(-7667573);
        this.f.setTextSize(20.0f);
        ((LinearLayout) findViewById(R.id.layout)).addView(this.f);
        this.e.setEnabled(false);
    }

    @Override // android.view.View.OnClickListener
    @SuppressLint({"NonConstantResourceId"})
    public void onClick(View view) {
        switch (view.getId()) {
            case R.id.button1 /* 2131230792 */:
                b(view);
                return;
            case R.id.button2 /* 2131230793 */:
                c(view);
                return;
            case R.id.button3 /* 2131230794 */:
                d(view);
                return;
            case R.id.button4 /* 2131230795 */:
                e(view);
                return;
            case R.id.buttonPanel /* 2131230796 */:
            default:
                return;
            case R.id.button_check /* 2131230797 */:
                f(view);
                return;
            case R.id.button_reset /* 2131230798 */:
                g(view);
                return;
        }
    }

    @Override // android.app.Activity
    public void onCreate(Bundle bundle) {
        super.onCreate(bundle);
        setContentView(R.layout.activity_main);
        this.e = (Button) findViewById(R.id.button_check);
        ((Button) findViewById(R.id.button1)).setOnClickListener(this);
        ((Button) findViewById(R.id.button2)).setOnClickListener(this);
        ((Button) findViewById(R.id.button3)).setOnClickListener(this);
        ((Button) findViewById(R.id.button4)).setOnClickListener(this);
        ((Button) findViewById(R.id.button_reset)).setOnClickListener(this);
        this.e.setOnClickListener(this);
    }
}
```

静态分析了以下代码逻辑，onCreate方法中对控件进行了绑定，并且设置了监听器1

监听器方法用于监听按钮是否被按下，为onClick方法，分别对button1~4和button_check和button_reset六个按钮控件设置了监听方法。

该入口类中设置了五个全局变量，分别是f872c(int),e(Button),f(TextView),

f871b(byte[100]),d(int)=16

首先分析a方法，a方法中进行一个判断，当f872c的值与f871b数组的长度相同的时候提示按钮点击过多。

经过观察分析b,c,d,e四个方法，这四个方法分别对应着Button1~4.每个方法开始都调用a方法进行判断，b方法为当button1按钮被点击后，首先f872c+1，然后将0存入f871b数组中，c、d、e三个方法和b相似，只不过每次存入数组的数字为1，2，3。

然后设置两个数组分别是按钮日志的前16个和从16开始后计数器个，然后进行jniCheck里面的check1方法和check2方法，这两个方法都封装在libJniCheck.so库文件中，需要IDA的动态调试来分析功能。**待后面学习动调后继续更新。**

> 根据上面的分析，我们可以知道f872c就是一个计数器，而f871b数组就是一个用来记录按钮控件点击记录的数组。所以简而言之，a方法就是用来判断点击次数是否超过100次。但是我也产生了一个疑惑，为什么一定要使用byte数组呢，在Java中哪怕是byte数组也是按照正常存储的，并不存在什么10进制转二进制的陷阱。那么就接着继续分析吧！



## ps：若软件为恶意软件

个恶意的App打开应用后，adb连接会自动断开，并 且无法通过USB连接上去。 再次使用Jadx查看App的包结构会发现有一个USBLock的相关 类

查看其中内容会发现，类中执行了一个setprop persist.sys.usb.config none 的 命 令 。 setprop 命 令 与 之 前 讲 的 getprop是相对的，setprop用于Android系统的某个属性。通过搜索 persist.sys.usb.config关键词会发现这个属性是与设置USB调试相 关的，当这个属性被设置为none时，就会导致adb调试被禁用，也就 达成了断开连接的作用。

如果无法使用USB连接，就无法通过adb启动frida-server，也 就无法使用Objection进行连接，推荐使用Termux软件。Termux是一个安卓手机的 Linux模拟器，可以在手机上模拟Linux环境。它提供一个命令行界 面，让用户与系统互动。其实它就是一个普通的手机App，可以从应 用商店下载安装。Termux的界面和一个普通的终端一样。

模拟器默认root过，此时只需要下载对应模拟器架构的fridaserver并用push推送到模拟器的对应目录下，这样不通过adb就可以 启动frida-server。

Objection网络模式连接

Objection默认使用USB模式连 接，但是可以通过-N参数使用网络模式，使用时只需要通过-h参数指 定IP地址、-p参数指定端口去实现网络模式连接。

通过输入“./frida-server -l 0.0.0.0:8888”并按回车键以网络模式运行frida-server，从而监听 任何对手机8888端口的网络连接。此时，通过以下命令可以查看 frida-server监听的端口。

```
可以通过
netstat -tulp | grep frida来验证
```

objection可以如此连接：

```
objection -N -h 192.168.31.52 -p 8888 -g
com.shimeng.qq2693533893 explore
```

