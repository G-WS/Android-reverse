# Frida脚本入门

## 0x01 Frida脚本的基本概念

Frida脚本就是利用Frida动态插桩框架，使用Frida导出的API和方法对内存控件里的对象方法进行监视，修改或者替换的一段代码。

Frida的API是用JavaScript实现的，所以可以充分利用JavaScript的匿名函数优势以及大量的Hook（钩子函数）和回调函数API。



Frida版本的helloWorld

```js
setTimeout(
    function(){
        Java.perform(function(){
            console.log("hello world")
        })
    }
)
```

首先我们把一个匿名函数作 为参数传给了setTimeout()函数，在这个匿名函数体中调用了Frida 的API函数Java.perform()，这个API函数本身又接受了一个匿名函数 作为参数，该匿名函数最终调用console.log()函数来打印一个"Hello world！"字符串。这里需要调用setTimeout()方法是因为该方法将函 数注册到JavaScript运行库中，然后在JavaScript运行库中调用 Java.perform()方法将函数注册到App的Java运行库中并在其中执行 该函数。

在手机上将FridaServer运行起来

需要将手机调整至开发者模式，USB连接

同时为了确认是否建立连接。可以通过frida-ps -U命令查看移动设备上正在运行的进程。

```sh
abd shell
cd /data/local/tmp 
./frida-server-15.2.2-android-arm64 
```

在写好的脚本的目录中。通过以下命令可以以attach模式注入指定应用

```shell
frida -U -l hello_world.js android.process.media
```

> ps:手机上的Frida-server需要与计算机上的Frida版本保持一致，否则会报错。

## 0x02 Java层Hook基础

### 1 载入类

Java.use方法用于加载一个Java类，相当于Java中的`Class.forName()`。比如要加载一个String类：

```
var StringClass = Java.use("java.lang.String");
```

加载内部类：

```
var MyClass_InnerClass = Java.use("com.luoyesiqiu.MyClass$InnerClass");
```

其中InnerClass是MyClass的内部类

### 2 修改函数的实现

修改一个函数的实现是逆向调试中相当有用的。修改一个函数的实现后，如果这个函数被调用，我们的Javascript代码里的函数实现也会被调用。

#### 2.1 函数参数类型表示

1. 对于基本类型，直接用它在Java中的表示方法就可以了，不用改变，例如：

- int
- short
- char
- byte
- boolean
- float
- double
- long

1. 基本类型数组，用左中括号接上基本类型的缩写

基本类型缩写表示表：

| 基本类型 | 缩写 |
| -------- | ---- |
| boolean  | Z    |
| byte     | B    |
| char     | C    |
| double   | D    |
| float    | F    |
| int      | I    |
| long     | J    |
| short    | S    |

例如：`int[]`类型，在重载时要写成`[I`

1. 任意类，直接写完整类名即可

例如：`java.lang.String`

1. 对象数组，用左中括号接上完整类名再接上分号

例如：`[java.lang.String;`

#### 2.2 带参数的构造函数

修改参数为byte[]类型的构造函数的实现

```php
ClassName.$init.overload('[B').implementation=function(param){
    //do something
}
```

> 注：ClassName是使用Java.use定义的类;param是可以在函数体中访问的参数

修改多参数的构造函数的实现

```php
ClassName.$init.overload('[B','int','int').implementation=function(param1,param2,param3){
    //do something
}
```

#### 2.3 无参数构造函数

```php
ClassName.$init.overload().implementation=function(){
    //do something
}
```

调用原构造函数

```kotlin
ClassName.$init.overload().implementation=function(){
    //do something
    this.$init();
    //do something
}
```

> 注意：当构造函数(函数)有多种重载形式，比如一个类中有两个形式的func：`void func()`和`void func(int)`，要加上overload来对函数进行重载，否则可以省略overload

#### 2.4 一般函数

修改函数名为func，参数为byte[]类型的函数的实现

```javascript
ClassName.func.overload('[B').implementation=function(param){
    //do something
    //return ...
}
```

#### 2.5 无参数的函数

```delphi
ClassName.func.overload().implementation=function(){
    //do something
}
```

> 注： 在修改函数实现时，如果原函数有返回值，那么我们在实现时也要返回合适的值

```javascript
ClassName.func.overload().implementation=function(){
    //do something
    return this.func();
}
```

### 3. 调用函数

和Java一样，创建类实例就是调用构造函数，而在这里用`$new`表示一个构造函数。

```php
var ClassName=Java.use("com.luoye.test.ClassName");
var instance = ClassName.$new();
```

实例化以后调用其他函数

```go
var ClassName=Java.use("com.luoye.test.ClassName");
var instance = ClassName.$new();
instance.func();
```

### 4. 字段操作

字段赋值和读取要在字段名后加`.value`，假设有这样的一个类：

```java
package com.luoyesiqiu.app;
public class Person{
    private String name;
    private int age;
}
```

写个脚本操作Person类的name字段和age字段：

```php
var person_class = Java.use("com.luoyesiqiu.app.Person");
//实例化Person类
var person_class_instance = person_class.$new();
//给name字段赋值
person_class_instance.name.value = "luoyesiqiu";
//给age字段赋值
person_class_instance.age.value = 18;
//输出name字段和age字段的值
console.log("name = ",person_class_instance.name.value, "," ,"age = " ,person_class_instance.age.value);
```

输出：

```ini
name =  luoyesiqiu , age =  18
```

### 5. 类型转换

用`Java.cast`方法来对一个对象进行类型转换，如将`variable`转换成`java.lang.String`：

```php
var StringClass=Java.use("java.lang.String");
var NewTypeClass=Java.cast(variable,StringClass);
```

### 6. Java.available字段

这个字段标记Java虚拟机（例如： Dalvik 或者 ART）是否已加载, 操作Java任何东西之前，要确认这个值是否为true

### 7. Java.perform方法

Java.perform(fn)在Javascript代码成功被附加到目标进程时调用，我们核心的代码要在里面写。格式：

```javascript
Java.perform(function(){
//do something...
});
```



### 举例

首先编写一个简单的APP用于练习，APP的MainActivity代码如下：

> 注意：创建工程的路径在/root/AndroidStudioProjects/reverseDemo1

```java
package com.example.reversedemo;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.util.Log;

public class MainActivity extends AppCompatActivity {

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
        }
    }
    void fun(int x,int y){
        Log.d("reverseDemo", String.valueOf(x+y));
    }
}
```

该APP逻辑为每次间隔一年在logcat中打印出fun(50,30)函数的运行结果，最终结果显示在控制台中。

```
adb logcat |grep reverseDemo
```

以下是编写的Frida脚本，目标是编写Hook fun()函数并打印出fun()函数的参数值。Frida脚本的最终代码如下所示：

```js
function main(){
    console.log("Script loaded successfully")
    Java.perform(function(){
        console.log("Inside java perform function")
        var MainActivity = Java.use('com.example.reverseDemo1.MainActivity')
        MainActivity.fun.implementation = function(x,y){
            console.log("x=>",x,"y=>",y)
            var ret_value = this.fun(x,y)
            return ret_value
        }
    })
}

setImmediate(main)
```

通过如下命令使用Frida的CLI模式以attach模式注入APP

```
frida -U -l demo1.js reverseDemo1
```

该脚本使用function关键字定义了一个main()函数，用于存放Hook脚本，然后调用Frida的API函数Java.perform()将脚本中的内容注入到Java运行库，注意，这个API参数是一个匿名函数，函数内容是健康空和修改Java函数逻辑的主体内容。注意这里的Java.perform()函数非常重要，任何对APP中Java层的操作都必须包裹在这个函数中，否则Frida运行起来就会报错。

在Java.perform()函数包裹的匿名函数中，首先调用了Frida的 API函数Java.use()，这个函数的参数是Hook的函数所在类的类名， 参数的类型是一个字符串类型，比如Hook的fun()函数所在类的全名 为com.roysue.demo02.MainActivity，那么传递给这个函数的参数 就是"com.roysue.demo02.MainActivity"。这个函数的返回值动态 地为相应Java类获取一个JavaScript Wrapper，可以通俗地理解为 一个JavaScript对象。 在获取到对应的JavaScript对象后，通过“.”符号连接fun这个 对 应 的 函 数 名 ， 然 后 加 上 implementation 关 键 词 表 示 实 现 MainActivity对象的fun()函数，最后通过“=”这个符号连接一个匿 名函数，参数内容和原Java的内容一致。不同的是，JavaScript是一 个弱类型的语言，不需要指明参数类型。此时一个针对MainActivity 类的fun()函数的Hook框架就完成了。 这个匿名函数的内容取决于逆向开发和分析人员想修改这个被 Hook的函数的哪些运行逻辑。比如调用console.log()函数把参数内容 打印出来，通过this.fun()函数再次调用原函数，并把原本的参数传递 给这个fun()函数。简而言之，就是重新执行原函数的内容，最后将这 个函数的返回值直接通过return指令返回。 在Hook一个函数时，还有一个地方需要注意，那就是最好不要 修改被Hook的函数的返回值类型，否则可能会引起程序崩溃等问 题，比如直接通过调用原函数将原函数的返回值返回。 当然，也可以传递不同的参数，简单修改程序逻辑即可。

当我们对APP增加一些新功能，修改代码如下：

```java
package com.example.reversedemo1;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.util.Log;

import java.util.Locale;

public class MainActivity extends AppCompatActivity {

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
}

```

fun函数有了重载，当参数是两个int类型的情况下返回两个整数之和，当参数为String类型时，返回字符串的 小写形式。如果使用之前的脚本，会报错如下：

```
Error: fun(): has more than one overload, use .overload(<signature>) to choose from:
        .overload('java.lang.String')                                                                         
        .overload('int', 'int')                                                                               
    at X (frida/node_modules/frida-java-bridge/lib/class-factory.js:569)                                      
    at K (frida/node_modules/frida-java-bridge/lib/class-factory.js:564)                                      
    at set (frida/node_modules/frida-java-bridge/lib/class-factory.js:932)                                    
    at <anonymous> (/frida/repl-2.js:6)                                                                       
    at <anonymous> (frida/node_modules/frida-java-bridge/lib/vm.js:12)                                        
    at _performPendingVmOps (frida/node_modules/frida-java-bridge/index.js:250)                               
    at <anonymous> (frida/node_modules/frida-java-bridge/index.js:225)                                        
    at <anonymous> (frida/node_modules/frida-java-bridge/lib/vm.js:12)                                        
    at _performPendingVmOpsWhenReady (frida/node_modules/frida-java-bridge/index.js:244)                      
    at perform (frida/node_modules/frida-java-bridge/index.js:204)                                            
    at changeArgs (/frida/repl-2.js:12)                                                                       
    at apply (native)                                                                                         
    at <anonymous> (frida/runtime/core.js:51)[Nexus 5X::reverseDemo1 ]->

```

这是函数的重载导致Frida不知道具体应该Hook哪 个 函 数 而 出 现 的 问 题 。 其 实 Frida 已 经 提 供 了 解 决 方 案 （ use .overload() ） ， 就 是 指 定 函 数 签 名 ， 将 报 错 中 的.overload('java.lang.String')或者.overload('int', 'int')添加到要 Hook的函数名后、关键词implementation之前。当然，相应的参数 和具体函数逻辑也得修改

具体代码如下：

```js
function changeArgs(){
    console.log("Script loaded successfully")
    Java.perform(function(){
        console.log("Inside java perform function")
        var MainActivity = Java.use("com.example.reversedemo1.MainActivity")
        console.log("Java.Use.successfully")//定位类成功
        MainActivity.fun.overload('int','int').implementation= function(x,y){
            console.log("x=>",x,"y=>",y)
            var ret_value = this.fun(2,5)
            return ret_value
        }
    })
}

setImmediate(changeArgs)
```

## 0x03 Java层主动调用

### 1、主动调用与被动调用

主动调用就是强制调用一个函数去执行。相对地，被动调用是由 App按照正常逻辑去执行函数，函数的执行完全依靠与用户交互完成 程序逻辑进而间接调用到关键函数，而主动调用则可以直接调用关键 函数，主动性更强，甚至可以直接完成关键数据的“自吐”。在逆向 分析过程中，如果不想分析详细的算法逻辑，可以直接通过主动传递 参数来调用关键算法函数，忽略方法函数的实现过程直接得到密文或 者明文，可以说这是各种算法调用的“克星”。

### 2、类方法与实例方法的主动调用

在Java中，类中的函数可分为两种：类函数和实例方法。通俗地 讲，就是静态的方法和动态的方法。类函数使用关键字static修饰， 和对应类是绑定的，如果类函数还被public关键词修饰着，在外部就 可以直接通过类去调用；实例方法则没有关键字static修饰，在外部 只能通过创建对应类的实例再通过这个实例去调用。在Frida中主动 调用的类型会根据方法类型区分开。如果是类函数的主动调用，直接 使用Java.use()函数找到类进行调用即可；如果是实例方法的主动调 用，则需要在找到对应的实例后对方法进行调用。这里用到了Frida 中非常重要的一个API函数Java.choose()，这个函数可以在Java的堆 中寻找指定类的实例。

首先，修改之前的APP MainActivity中的内容

```java
package com.example.reversedemo1;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.util.Log;

import java.util.Locale;

public class MainActivity extends AppCompatActivity {

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
        Log.d("reverseDemo.secret", "secret: this is secret func");
    }
    static void staticSecret(){
        Log.d("reverseDemo.Secret", "this is staticSecret func");
    }
}

```

在这个App中，两个新加的函数并没有被调用。接下 来，我们完成两个隐藏函数的主动调用，js脚本代码如下：

```js
function demo3(){
    console.log("Script loaded successfully")
    Java.perform(function(){
        console.log("Inside java perform function")

        //静态函数主动调用
        var MainActivity = Java.use("com.example.reversedemo1.MainActivity")
        MainActivity.staticSecret()
        
        //动态方法主动调用
        Java.choose("com.example.reversedemo1.MainActivity",{
            onMatch: function(instance){
                console.log("instance found",instance)
                instance.secret()
            },
            onComplete:function(){
                console.log('search Complete')
            }
        })
        
    })
}

setImmediate(demo3)
```

可以发现静态的staticSecret()函数 和Hook时使用的方式大同小异，都是使用Java.use这个API去获取 MainActivity类，在获取对应的类对象后通过“.”连接符连接 staticSecret方法名，最终以和Java中一样的方式直接调用静态方法 staticSecret()函数；动态方法secret需要先通过Java.choose这个 API从内存中获取相应类的实例对象，然后才能通过这个实例对象去 调用动态的secret()函数。如果使用“MainActivity.secret();”方式 （和staticSecret()函数一样）调用，那么会报错。

当将脚本注入APP后，APP运行的日志中就可以发现两个secret函数都已经执行了。

## 0x04 RPC及其自动化

在Frida中，可以使用python完成JavaScript脚本对进程的注入以及相应的Hook。

首先修改之前的MainActivity类中的代码

```java
package com.example.reversedemo1;

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

这次脚本的目的时获取total这个实例变量的值。

Java中的变量也存在是否使用static修饰的区别。如果使用static修饰，可以直接通过类进行获取；实例变量，不适用static修饰，和特定的对象绑在一起。

```js
function CallSecretFunc(){
    console.log("Script loaded successfully")
    Java.perform(function(){
        console.log("Inside java perform function")
        
        //动态方法主动调用
        Java.choose("com.example.reversedemo1.MainActivity",{
            onMatch: function(instance){
                console.log("instance found",instance)
                instance.secret()
            },
            onComplete:function(){
                console.log('search Complete')
            }
        })
        
    })
}
function getTotalValue(){
    Java.perform(function(){
        var MainActivity = Java.use("com.example.reversedemo1.MainActivity")
        Java.choose("com.example.reversedemo1.MainActivity",{
            onMatch: function(instance){
                console.log("total value= ",instance.total.value)
                instance.secret()
            },
            onComplete:function(){
                console.log('search Complete')
            }
        })
        
    })
}

setImmediate(getTotalValue)
```

编写的python RPC脚本如下：

```python
import frida,sys
def on_message(message,data):
    if message['type']=='send':
        print("[*]{0}".format(message['payload']))
    else:
        print(message)

device=frida.get_usb_device()
process=device.attach("com.example.reversedemo1")
with open('callSecret.js') as f:
    jscode=f.read()
    script = process.create_script(jscode)
    script.on('message',on_message())
    script.load()

    command=""
    while 1==1:
        command=input("\nEnter command:\n1:Exit\n2:Call secret function\n3:Get Total Value\nchoice:")
        if command=="1":
            break
        elif command=="2":
            script.exports.callsecretfunc()
        elif command=="3":
            script.exports.gettotalvalue()
```

