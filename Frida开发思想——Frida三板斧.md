# Frida开发思想——Frida三板斧

## Objection辅助定位

Frida 需要每次手动编写代码去Hook从静态分析到的函数，进而观察其参 数和返回值是否与需求相符，Objection将常用的一些功能集成在一 起，使得逆向开发和分析人员在分析过程中不需要浪费精力在编写代 码上。

下面以 Junior.apk为例，样 本 来 自 于 《 Android Studio开发实战：从零基础到App上线（第2版）》一书中的Junior 样例，源代码在https://github.com/aqi00/android2上。

安装遍历APP的所有Activity

安装过程中报错INSTALL_FAILED_TEST_ONLY: installPackageLI

通过AndroidKiller将AndroidManifest.xml中的android:testOnly="true"改成false或者直接去掉，然后重新编译打包，再次安装即可

先打开手机的fridaserver，然后命令如下：

```
objection -g com.example.junior explore
Using USB device `Nexus 5X`
Agent injected and responds ok!

     _   _         _   _
 ___| |_|_|___ ___| |_|_|___ ___
| . | . | | -_|  _|  _| | . |   |
|___|___| |___|___|_| |_|___|_|_|
      |___|(object)inject(ion) v1.11.0

     Runtime Mobile Exploration
        by: @leonjza from @sensepost

[tab] for command suggestions
com.example.junior on (google: 8.1.0) [usb] # 

```

```
com.example.junior on (google: 8.1.0) [usb] # ls
Type  Last Modified  Read  Write  Hidden  Size  Name
----  -------------  ----  -----  ------  ----  ----

Readable: True  Writable: True
com.example.junior on (google: 8.1.0) [usb] # android hooking list activities
com.example.junior.BbsActivity
com.example.junior.CalculatorActivity
com.example.junior.CaptureActivity
com.example.junior.ClickActivity
com.example.junior.ColorActivity
com.example.junior.GravityActivity
com.example.junior.IconActivity
com.example.junior.MainActivity
com.example.junior.MarginActivity
com.example.junior.MarqueeActivity
com.example.junior.NineActivity
com.example.junior.PxActivity
com.example.junior.ScaleActivity
com.example.junior.ScreenActivity
com.example.junior.ScrollActivity
com.example.junior.ShapeActivity
com.example.junior.StateActivity

Found 17 classes
com.example.junior on (google: 8.1.0) [usb] # 

```

我们发现整个App总共有 17个activity，这里以com.example.junior.CalculatorActivity为例进行分析

```
com.example.junior on (google: 8.1.0) [usb] # android intent launch_activity com.example.
junior.CalculatorActivity
(agent) Starting activity com.example.junior.CalculatorActivity...
(agent) Activity successfully asked to start.

```

观察测试机，我们可以发现activity被启动

每次点击等号后按钮的 计算结果都会被打印出来，我们找到了对应等号的按钮id为btn_equal,并且根据这个id找到了对饮的源码

```
else if (resid == R.id.btn_equal) {
            if (this.operator.length() == 0 || this.operator.equals("＝")) {
                Toast.makeText(this, "请输入运算符", 0).show();
            } else if (this.nextNum.length() <= 0) {
                Toast.makeText(this, "请输入数字", 0).show();
            } else if (caculate()) {
                this.operator = inputText;
                this.showText += "=" + this.result;
                this.tv_result.setText(this.showText);
            }
```

我们发现主要的代码点击后在caculate()方法中，接下来我们需要验证我们的想法

通过以下命令以及其执行结果如下：

```
com.example.junior on (google: 8.1.0) [usb] # android hooking list class_methods com.exam
ple.junior.CalculatorActivity
private boolean com.example.junior.CalculatorActivity.caculate()
private void com.example.junior.CalculatorActivity.clear(java.lang.String)
protected void com.example.junior.CalculatorActivity.onCreate(android.os.Bundle)
public void com.example.junior.CalculatorActivity.onClick(android.view.View)

Found 4 method(s)

```

使用如下命令Hook这个函数来确认在点击 “等号”按钮后这个函数被调用了。在Hook上后，任意输入一个表 达式并点击“等号”按钮，会发现这个函数在点击“等号”按钮后被 调用，Hook结果如下

```
com.example.junior on (google: 8.1.0) [usb] # android hooking watch class_method com.exam
ple.junior.CalculatorActivity.caculate --dump-args --dump-backtrace --dump-return
(agent) Attempting to watch class com.example.junior.CalculatorActivity and method caculate.
(agent) Hooking com.example.junior.CalculatorActivity.caculate()
(agent) Registering job 113166. Type: watch-method for: com.example.junior.CalculatorActivity.caculate                                                                            
com.example.junior on (google: 8.1.0) [usb] # (agent) [113166] Called com.example.junior.CalculatorActivity.caculate()
(agent) [113166] Backtrace:
        com.example.junior.CalculatorActivity.caculate(Native Method)
        com.example.junior.CalculatorActivity.onClick(CalculatorActivity.java:94)
        android.view.View.performClick(View.java:6294)
        android.view.View$PerformClick.run(View.java:24770)
        android.os.Handler.handleCallback(Handler.java:790)
        android.os.Handler.dispatchMessage(Handler.java:99)
        android.os.Looper.loop(Looper.java:164)
        android.app.ActivityThread.main(ActivityThread.java:6494)
        java.lang.reflect.Method.invoke(Native Method)
        com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:438)
        com.android.internal.os.ZygoteInit.main(ZygoteInit.java:807)

(agent) [113166] Return Value: true
```

caculate方法在jadx中显示如下：

```java
private boolean caculate() {
        if (this.operator.equals("＋")) {
            this.result = String.valueOf(Arith.add(this.firstNum, this.nextNum));
        } else if (this.operator.equals("－")) {
            this.result = String.valueOf(Arith.sub(this.firstNum, this.nextNum));
        } else if (this.operator.equals("×")) {
            this.result = String.valueOf(Arith.mul(this.firstNum, this.nextNum));
        } else if (this.operator.equals("÷")) {
            if (Double.parseDouble(this.nextNum) == 0.0d) {
                Toast.makeText(this, "被除数不能为零", 0).show();
                return false;
            }
            this.result = String.valueOf(Arith.div(this.firstNum, this.nextNum));
        }
        Log.d(TAG, "result=" + this.result);
        this.firstNum = this.result;
        this.nextNum = BuildConfig.FLAVOR;
        return true;
    }
```

对减法的处理是通过调用Arith类中的sub()函数 来实现的。为了验证Arith类在内存中是真实存在的，我们通常使用以 下Objection命令来获取一个应用在内存中的所有类。

```
@android hooking list classes
```

在运行这行命令后会列出很多类，甚至会超过整个 Terminal缓存空间，这时会出现一些类被缓存冲刷掉的情况，如果只 是简单地在终端窗口里查找，那么不一定能找到。其实Objection本 身有一个log文件，用于记录objection运行时产生的所有数据。这个 日志数据存放在~/.objection目录下的objection.log文件中

解 决 方 法 ： 在 运 行 objection 注 入 App 之 前 ， 首 先 切 换 到 ~/.objection目录下，将之前的objection.log文件删除或者改名

```
(base) ┌──(root㉿kali)-[/]
└─# cd ~/.objection/
                                                                                           
(base) ┌──(root㉿kali)-[~/.objection]
└─# ls
objection_history  objection.log  version_info
                                                                                           
(base) ┌──(root㉿kali)-[~/.objection]
└─# mv objection.log objection_old.log
                                                                                           
(base) ┌──(root㉿kali)-[~/.objection]
└─# cat objection.log
cat: objection.log: 没有那个文件或目录

```

```
(base) ┌──(root㉿kali)-[~/.objection]
└─# objection -g com.example.junior explore
Using USB device `Nexus 5X`
Agent injected and responds ok!

     _   _         _   _
 ___| |_|_|___ ___| |_|_|___ ___
| . | . | | -_|  _|  _| | . |   |
|___|___| |___|___|_| |_|___|_|_|
      |___|(object)inject(ion) v1.11.0

     Runtime Mobile Exploration
        by: @leonjza from @sensepost

[tab] for command suggestions
com.example.junior on (google: 8.1.0) [usb] # android hooking list classes
[B
[C
[D
[F
[I
[J
[Landroid.animation.Animator;
[Landroid.animation.Keyframe$FloatKeyframe;
[Landroid.animation.PropertyValuesHolder;
[Landroid.app.LoaderManagerImpl;
[Landroid.arch.lifecycle.Lifecycle$Event;
[Landroid.arch.lifecycle.Lifecycle$State;
[Landroid.content.pm.ActivityInfo;
[Landroid.content.pm.ConfigurationInfo;
[Landroid.content.pm.FeatureGroupInfo;
[Landroid.content.pm.FeatureInfo;
[Landroid.content.pm.InstrumentationInfo;
[Landroid.content.pm.PathPermission;
[Landroid.content.pm.PermissionInfo;
[Landroid.content.pm.ProviderInfo;
[Landroid.content.pm.ServiceInfo;
[Landroid.content.pm.Signature;
[Landroid.content.res.Configuration;
[Landroid.content.res.StringBlock;
[Landroid.content.res.XmlBlock;
[Landroid.database.sqlite.SQLiteConnectionPool$AcquiredConnectionStatus;
[Landroid.graphics.Bitmap$Config;
......
......
sun.util.locale.LanguageTag
sun.util.locale.LocaleObjectCache
sun.util.locale.LocaleObjectCache$CacheEntry
sun.util.locale.LocaleSyntaxException
sun.util.locale.LocaleUtils
sun.util.locale.ParseStatus
sun.util.locale.StringTokenIterator
sun.util.logging.LoggingProxy
sun.util.logging.LoggingSupport
sun.util.logging.LoggingSupport$1
sun.util.logging.PlatformLogger
sun.util.logging.PlatformLogger$1
sun.util.logging.PlatformLogger$Level
void

Found 5233 classes
```

在遍历完成后退出Objection注入模式以确保log文件刷新成功， 并重新通过cat命令查看这个objection.log文件，由于log文件过大， 因此还需要配合grep命令过滤文本，从而通过观察结果是否有输出来 判定内存中是否存在目标类Arith

在判定内存中确实存在Arith类后，我们进一步通过Objection命 令判断Arith类是否存在sub()函数

```
(base) ┌──(root㉿kali)-[~/.objection]
└─# cat objection.log | grep com.example.junior.util.Arith

```

```
com.example.junior on (google: 8.1.0) [usb] # android hooking list class_methods com.exampl
e.junior.util.Arith
public static java.lang.String com.example.junior.util.Arith.add(double,double)
public static java.lang.String com.example.junior.util.Arith.add(java.lang.String,java.lang.String)
public static java.lang.String com.example.junior.util.Arith.div(double,double)
public static java.lang.String com.example.junior.util.Arith.div(double,double,int)
public static java.lang.String com.example.junior.util.Arith.div(java.lang.String,java.lang.String)
public static java.lang.String com.example.junior.util.Arith.div(java.lang.String,java.lang.String,int)
public static java.lang.String com.example.junior.util.Arith.mul(double,double)
public static java.lang.String com.example.junior.util.Arith.mul(java.lang.String,java.lang.String)
public static java.lang.String com.example.junior.util.Arith.round(double,int)
public static java.lang.String com.example.junior.util.Arith.sub(double,double)
public static java.lang.String com.example.junior.util.Arith.sub(java.lang.String,java.lang.String)

Found 11 method(s)

```

在内存中确定这个函数存在后，便可以使用如下命令对这个函数 进行Hook了

```
com.example.junior on (google: 8.1.0) [usb] # android hooking watch class_method com.exampl
e.junior.util.Arith.sub --dump-args --dump-backtrace --dump-return
(agent) Attempting to watch class com.example.junior.util.Arith and method sub.
(agent) Hooking com.example.junior.util.Arith.sub(double, double)
(agent) Hooking com.example.junior.util.Arith.sub(java.lang.String, java.lang.String)
(agent) Registering job 304689. Type: watch-method for: com.example.junior.util.Arith.sub
com.example.junior on (google: 8.1.0) [usb] # (agent) [304689] Called com.example.junior.util.Arith.sub(java.lang.String, java.lang.String)
(agent) [304689] Backtrace:
        com.example.junior.util.Arith.sub(Native Method)
        com.example.junior.CalculatorActivity.caculate(CalculatorActivity.java:169)
        com.example.junior.CalculatorActivity.onClick(CalculatorActivity.java:94)
        android.view.View.performClick(View.java:6294)
        android.view.View$PerformClick.run(View.java:24770)
        android.os.Handler.handleCallback(Handler.java:790)
        android.os.Handler.dispatchMessage(Handler.java:99)
        android.os.Looper.loop(Looper.java:164)
        android.app.ActivityThread.main(ActivityThread.java:6494)
        java.lang.reflect.Method.invoke(Native Method)
        com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:438)
        com.android.internal.os.ZygoteInit.main(ZygoteInit.java:807)

(agent) [304689] Arguments com.example.junior.util.Arith.sub(7, 9)
(agent) [304689] Return Value: -2

```

