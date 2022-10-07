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

## Frida脚本修改参数、主动调用

确认了Arith类的函数sub是最终计算器减法的真实执行函数，接下来通过Frida脚本进行进一步的利用。

Hook脚本如下：

```js
function main(){
    Java.perform(function(){
        var Arith = Java.use('com.example.junior.util.Arith')
        Arith.sub.implementation = function(str,str2){
            var result = this.sub(str,str2)
            console.log('str,str2,result =>',str,str2,result)
            //打印java调用栈
            console.log(Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()))
            return result
        }
    })
}
setImmediate(main)
```

打印Java调用栈的代码就是将Android开发中获取调用栈的函数Log.getStackTraceString(Throwable e)翻译为JavaScript语言。

在 使 用 Frida 将 代 码 注 入 App 前 ， 我 们 先 取 消 Objection 的 Hook，然后进行Frida脚本的注入。这里取消Objection对目标函数 的Hook是因为不能使用Objection和Frida对同一个函数进行Hook， 否则会报错。

```
frida -UF -l hook1.js
```

```
(base) ┌──(root㉿kali)-[/home/zhy/VscodeWorkspace/Junior_hook]
└─# frida -UF -l hook1.js
     ____
    / _  |   Frida 15.2.2 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to Nexus 5X (id=00e310eb64541e43)
Error: sub(): has more than one overload, use .overload(<signature>) to choose from:
        .overload('double', 'double')                                                    
        .overload('java.lang.String', 'java.lang.String')                                
    at X (frida/node_modules/frida-java-bridge/lib/class-factory.js:569)                 
    at K (frida/node_modules/frida-java-bridge/lib/class-factory.js:564)                 
    at set (frida/node_modules/frida-java-bridge/lib/class-factory.js:932)               
    at <anonymous> (/frida/repl-2.js:3)                                                  
    at <anonymous> (frida/node_modules/frida-java-bridge/lib/vm.js:12)                   
    at _performPendingVmOps (frida/node_modules/frida-java-bridge/index.js:250)          
    at <anonymous> (frida/node_modules/frida-java-bridge/index.js:225)                   
    at <anonymous> (frida/node_modules/frida-java-bridge/lib/vm.js:12)                   
    at _performPendingVmOpsWhenReady (frida/node_modules/frida-java-bridge/index.js:244) 
    at perform (frida/node_modules/frida-java-bridge/index.js:204)                       
    at main (/frida/repl-2.js:11)                                                        
    at apply (native)                                                                    
    at <anonymous> (frida/runtime/core.js:51)                                            
[Nexus 5X::junior ]->
```

从这里我们可以发现sub函数进行了重载，所以我们需要修改脚本加入对于重载的选择

```
(base) ┌──(root㉿kali)-[/home/zhy/VscodeWorkspace/Junior_hook]
└─# frida -UF -l hook1.js
     ____
    / _  |   Frida 15.2.2 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to Nexus 5X (id=00e310eb64541e43)
[Nexus 5X::junior ]-> str,str2,result => 7 2 -116
java.lang.Throwable
        at com.example.junior.util.Arith.sub(Native Method)
        at com.example.junior.CalculatorActivity.caculate(CalculatorActivity.java:169)
        at com.example.junior.CalculatorActivity.onClick(CalculatorActivity.java:94)
        at android.view.View.performClick(View.java:6294)
        at android.view.View$PerformClick.run(View.java:24770)
        at android.os.Handler.handleCallback(Handler.java:790)
        at android.os.Handler.dispatchMessage(Handler.java:99)
        at android.os.Looper.loop(Looper.java:164)
        at android.app.ActivityThread.main(ActivityThread.java:6494)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:438)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:807)



```

我们会发现1-7的结果，APP页面的最终结果变成了-122

这里的123直接传递实际上是有问题的，正确的传入代码如下：

```js
function main(){
    Java.perform(function(){
        var Arith = Java.use('com.example.junior.util.Arith')
        Arith.sub.overload("java.lang.String","java.lang.String").implementation = function(str,str2){
            //Arith.sub.implementation = function(str,str2){

            //var result = this.sub(str,"123")
            var JavaString = Java.use('java.lang.String')
            var result = this.sub(str,JavaString.$new('123'))
            console.log('str,str2,result =>',str,str2,result)
            //打印java调用栈
            console.log(Java.use("android.util.Log").getStackTraceString(Java.use("java.lang.Throwable").$new()))
            return result
        }
    })
}
setImmediate(main)
```

```
(base) ┌──(root㉿kali)-[/home/zhy/VscodeWorkspace/Junior_hook]
└─# frida -UF -l hook1.js
     ____
    / _  |   Frida 15.2.2 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to Nexus 5X (id=00e310eb64541e43)
[Nexus 5X::junior ]-> str,str2,result => 4 9 -119
java.lang.Throwable
        at com.example.junior.util.Arith.sub(Native Method)
        at com.example.junior.CalculatorActivity.caculate(CalculatorActivity.java:169)
        at com.example.junior.CalculatorActivity.onClick(CalculatorActivity.java:94)
        at android.view.View.performClick(View.java:6294)
        at android.view.View$PerformClick.run(View.java:24770)
        at android.os.Handler.handleCallback(Handler.java:790)
        at android.os.Handler.dispatchMessage(Handler.java:99)
        at android.os.Looper.loop(Looper.java:164)
        at android.app.ActivityThread.main(ActivityThread.java:6494)
        at java.lang.reflect.Method.invoke(Native Method)
        at com.android.internal.os.RuntimeInit$MethodAndArgsCaller.run(RuntimeInit.java:438)
        at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:807)



```

对sub()函数的调用，我们会发现在Java 中是直接通过Arith类对象来完成对sub()函数的调用。如果还是无法 确认，则可以通过将Objection注入到应用中，再使用如下命令打印 出Airth类的所有函数：

```
om.example.junior on (google: 8.1.0) [usb] # android hooking list class_methods com.exam
ple.junior.util.Arith
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

我们可以发现后面两条，然后编写函数调用的脚本

```js
function callSub(){
    Java.perform(function(){
        var Arith = Java.use('com.example.junior.util.Arith')
        var JavaString = Java.use('java.lang.String')
        var result = Arith.sub(JavaString.$new("123"),JavaString.$new("111"))
        console.log("123-111=",result)
    })
}
setImmediate(callSub)
```

结果如下：

```
(base) ┌──(root㉿kali)-[/home/zhy/VscodeWorkspace/Junior_hook]
└─# frida -UF -l hook2.js
     ____
    / _  |   Frida 15.2.2 - A world-class dynamic instrumentation toolkit
   | (_| |
    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to Nexus 5X (id=00e310eb64541e43)
123-111= 12
[Nexus 5X::junior ]->

```

## 规模化利用RPC

批量数据的调用，就需要修改原有的主动调用脚本call.js 的内容，将原本只调用一次的sub()函数修改为可以调用多次的格式， 并且需要将完成主动调用的函数修改为导出的rpc函数。具体脚本如下：

```
function callSub(){
    Java.perform(function(){
        var Arith = Java.use('com.example.junior.util.Arith')
        var JavaString = Java.use('java.lang.String')
        var result = Arith.sub(JavaString.$new(a),JavaString.$new(b))
        console.log(a,"-",b,"=",result)
    })
}
rpc.exports = {
    sub : callSub,
};

```

python脚本如下：

```
import frida,sys
def on_message(message,data):
    if message['type']=='send':
        print("[*]{0}".format(message['payload']))
    else:
        print(message)

device = frida.get_usb_device()

process = device.attach('com.example.junior')
with open('call.js')as f:
    jscode = f.read()
script = process.create_script(jscode)

script.on('message',on_message)
script.load()

for i in range(20,30):
    for j in range(0,10):
        script.exports.sub(str(i),str(j))
```


可以简单总结如下：第一步，使用Objection快速Hook 定位；第二步，通过Frida脚本进行关键函数的逻辑修改与主动调用；第三步，将Frida脚本的主动调用结合Python完成了对关键函数的大规模实例利用。
