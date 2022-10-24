# Frida native层Hook

针对App native层的基本Hook模板如下：

```js
Interceptor.attach(addr,{
	onEnter(args){
		/* do someting with args */
	},
	onLeave(retval){
		/* do something with retval */
	}
})
```

在Frida脚本中实现native层Hook的API函数 是Interceptor.attach()函数，它的第一个参数是要Hook的函数地 址，第二个参数是一个callbacks回调。在callbacks回调中存在两个 函数：onEnter()函数是在函数调用前产生的回调，在这个函数中可 以处理函数参数的相关内容，被Hook的函数参数内容以数组的方式 存储在onEnter()函数的参数args中；onLeave()函数则是在被Hook 的目标函数执行完成后执行的函数，被Hook的函数返回值用 onLeave()函数中的retval变量来表示。

如 果 是 导 出 函 数 ， 那 么 可 以 通 过 API 函 数 Module.getExportByName(moduleName|null,exportName)或者 Module.findExportByName(moduleName|null, exportName)获 得相应函数的首地址。这两个API函数的第一个参数是模块名或null 值，第二个参数是目标函数的导出符号名。如果第一个参数为null， 那么API函数在执行时会在内存加载的所有模块中搜索导出符号名， 否则会在指定模块中搜索相应的导出符号。这两个API函数都是用于 寻找导出函数在内存的地址，不过还是存在一定的差别：以get开头的 函数无法寻找到相应导出符号名时会抛出一个异常，以find开头的函 数无法寻找到相应导出符号名时会直接返回一个null值。

下面是Hook的脚本：

```js
function hook_native(){
    var addr = Module.getExportByName("libjnitest.so","Java_com_example_jnitest_MainActivity_stringFromJNI");
    Interceptor.attach(addr,{
        onEnter:function(args){
            console.log("jinenv pointer=>",args[0])
            console.log("jobj pointer=>",args[1])
        },onLeave:function(retval){
            console.log("retval is =>",Java.vm.getEnv().getStringUtfChars(retval,null).readCString())
            console.log("================")
        }
    })
}
function main(){
    hook_native()
}

setImmediate(main)
```

下面是Hook的demo源代码：

为了能够保证stringFromJNI()函数的执行在完成Hook后执行， 这 里 先 在 MainActivity 类 的 onCreate() 函 数 中 加 上 一 个 对 stringFromJNI()函数的循环调用，修改后的onCreate()函数内容如

```java
package com.example.jnitest;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.util.Log;
import android.widget.TextView;

import com.example.jnitest.databinding.ActivityMainBinding;

public class MainActivity extends AppCompatActivity {

    // Used to load the 'jnitest' library on application startup.
    static {
        System.loadLibrary("jnitest");
    }

    private ActivityMainBinding binding;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        binding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());

        // Example of a call to a native method
        TextView tv = binding.sampleText;
        tv.setText(stringFromJNI());
        
        while(true) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Log.i("zhy", stringFromJNI());
        }
    }

    /**
     * A native method that is implemented by the 'jnitest' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();
//    public native String stringFromJNI2();
}
```

```c
#include <jni.h>
#include <string>

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_jnitest_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}

//extern "C" JNIEXPORT jstring JNICALL
//Java_com_example_jnitest_MainActivity_stringFromJNI2(JNIEnv *env, jobject thiz) {
//        std::string hello = "Hello from C++ stringFrom JNI2";
//        return env->NewStringUTF(hello.c_str());
//}
```

保证手机上运行Frida-server

通过attach的方式将脚本注入手机中：

```sh
(base) ┌──(root㉿kali)-[/home/zhy/桌面/NativeHook]
└─# frida -UF -l hookNative.js                            
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
[Nexus 5X::JniTest ]-> jinenv pointer=> 0x72e9acc1c0
jobj pointer=> 0x7ff5cdf774
retval is => Hello from C++
================
jinenv pointer=> 0x72e9acc1c0
jobj pointer=> 0x7ff5cdf774
retval is => Hello from C++
================
jinenv pointer=> 0x72e9acc1c0
jobj pointer=> 0x7ff5cdf774
retval is => Hello from C++

```

每次App重新运行后native函数加载的绝对 地址是会变化的，唯一不变的是函数相对于所在模块基地址的偏移， 因此我们可以在获取模块的基地址后加上固定的偏移地址获取相应函 数 的 地 址 ， Frida 中 也 正 好 提 供 了 这 样 的 方 式 ： 先 通 过 Module.findBaseAddress(name) 或 者 Module.getBaseAddress(name)两个API函数获取对应模块的基地 址，然后通过add(offset)函数传入固定的偏移offset获取最后的函数 绝对地址。

首先介绍一下动态注册 的JNI函数。区别于静态注册的JNI函数可以通过一定命名方式从 Java层找到对应native层函数名，动态注册的函数在native层实现的 函数名称不定，并且不一定要求相应函数是导出类型，这样的函数在 安全性上是一定会高于静态注册函数的，而这也正成为许多开发者选 择动态注册函数的理由。动态注册的函数实现方式十分简单，只需要 调用RegisterNatives()函数即可

```java
jint RegisterNatives(jclass clazz,const JNINativeMethod* methods,jintnMethods)
```

在RegisterNatives()函数中，第一个参数clazz是native函数所在 的类，可通过FindClass这个JNI函数获取（将类名的“.”符号换成 “/”）；methods参数是一个数组，其中包含函数的一些签名信息 以及对应在native层的函数指针，nMethods参数是methods数组的 数量。

以之前的demo工程为例，添加一个native函数stringFromJNI3()，具体实现如下：

```c
jstring JNICALL sI3(JNIEnv* env,jobject /*this*/){
    std::string hello = "Hello from C++ stringFromJNI3 zhy";
    return env->NewStringUTF(hello.c_str());
}

jint JNI_OnLoad(JavaVM* vm,void* reserved){
    JNIEnv* env;
    vm->GetEnv((void**)&env,JNI_VERSION_1_6);
    JNINativeMethod  method[] = {
            {"stringFromJNI3","()Ljava/lang/String;",(void*)sI3},
    };
    env->RegisterNatives(env->FindClass("com/example/jnitest/MainActivity"),method,1);
    return JNI_VERSION_1_6;
}
```

对MainActivity进行如下修改：

```java
package com.example.jnitest;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
import android.util.Log;
import android.widget.TextView;

import com.example.jnitest.databinding.ActivityMainBinding;

public class MainActivity extends AppCompatActivity {

    // Used to load the 'jnitest' library on application startup.
    static {
        System.loadLibrary("jnitest");
    }

    private ActivityMainBinding binding;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        binding = ActivityMainBinding.inflate(getLayoutInflater());
        setContentView(binding.getRoot());

        // Example of a call to a native method
        TextView tv = binding.sampleText;
        tv.setText(stringFromJNI());

        while(true) {
            try {
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            Log.i("zhy", stringFromJNI3());
        }
    }

    /**
     * A native method that is implemented by the 'jnitest' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();
    public native String stringFromJNI3();
//    public native String stringFromJNI2();
}
```

再次使用Objection遍历libjnitest.so模块中的导出函数，会发现无法找到stringFromJNI3()字符串相关的函数。

```sh
(base) ┌──(root㉿kali)-[/home/zhy/桌面/NativeHook]
└─# objection -g JniTest explore       
Checking for a newer version of objection...
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
com.example.jnitest on (google: 8.1.0) [usb] # memory list exports libjnitest.so
Save the output by adding `--json exports.json` to this command
Type      Name                                                                                                             Address
--------  ---------------------------------------------------------------------------------------------------------------  ------------
variable  _ZTSN10__cxxabiv117__array_type_infoE                                                                            0x72d3cb4a90
function  __cxa_call_unexpected                                                                                            0x72d3c9a75c
function  _ZNSt12length_errorD1Ev                                                                                          0x72d3c9afa0
variable  _ZTVN10__cxxabiv123__fundamental_type_infoE                                                                      0x72d3cbdf28
variable  _ZTIPDn                                                                                                          0x72d3cbdfd8
function  __cxa_free_exception                                                                                             0x72d3c9967c
variable  _ZTIN10__cxxabiv117__array_type_infoE                                                                            0x72d3cbe780
function  _ZN10__cxxabiv121__isOurExceptionClassEPK17_Unwind_Exception                                                     0x72d3c99610
function  _ZNSt11logic_errorC2EPKc                                                                                         0x72d3c99348
variable  _ZTIPDs                                                                                                          0x72d3cbe6b8
variable  _ZTVN10__cxxabiv129__pointer_to_member_type_infoE                                                                0x72d3cbe9b0
variable  _ZTIPDu                                                                                                          0x72d3cbe668
function  _ZnamSt11align_val_tRKSt9nothrow_t                                                                               0x72d3c99270
function  _ZNSt10bad_typeidD2Ev                                                                                            0x72d3caee58
variable  __cxa_new_handler                                                                                                0x72d3cbf038
function  _ZNSt13bad_exceptionD1Ev                                                                                         0x72d3c9af34
variable  _ZTVSt15underflow_error                                                                                          0x72d3cbc050
function  _ZdlPvm                                                                                                          0x72d3c99184
variable  _ZTVSt11range_error                                                                                              0x72d3cbbfb8
function  _ZNSt6__ndk117_DeallocateCaller27__do_deallocate_handle_sizeEPvm                                                 0x72d3c98750
function  _ZdaPvSt11align_val_t                                                                                            0x72d3c992ac
function  _ZNSt11logic_errorC1ERKS_                                                                                        0x72d3c993c8
variable  _ZTSN10__cxxabiv123__fundamental_type_infoE                                                                      0x72d3cb4978
function  _ZNSt13runtime_errorC1EPKc                                                                                       0x72d3c994e4
function  __gxx_personality_v0                                                                                             0x72d3c99f74
function  __cxa_throw                                                                                                      0x72d3c996d0
variable  _ZTISt12length_error                                                                                             0x72d3cbbf60
function  _ZNSt9bad_allocC2Ev                                                                                              0x72d3c9af58
function  _ZNSt12domain_errorD0Ev                                                                                          0x72d3c9b110
variable  _ZTSSt16invalid_argument                                                                                         0x72d3cb2a14
variable  _ZTSN10__cxxabiv116__enum_type_infoE                                                                             0x72d3cb4ab2
function  _ZNSt6__ndk111char_traitsIcE6assignERcRKc                                                                        0x72d3c98ce8
variable  _ZTSSt12out_of_range                                                                                             0x72d3cb2a3a
function  _ZdlPvmSt11align_val_t                                                                                           0x72d3c992a4
function  _ZNSt12length_errorD2Ev                                                                                          0x72d3c9afa0
variable  _ZTISt8bad_cast                                                                                                  0x72d3cbea68
variable  _ZTVN10__cxxabiv121__vmi_class_type_infoE                                                                        0x72d3cbe8d8
variable  _ZTVSt9bad_alloc                                                                                                 0x72d3cbbd58
function  _ZSt14set_unexpectedPFvvE                                                                                        0x72d3c9b484
function  _ZNKSt13bad_exception4whatEv                                                                                     0x72d3c9af4c
function  _ZdlPv                                                                                                           0x72d3c9917c
function  _ZNSt13runtime_errorC2ERKS_                                                                                      0x72d3c99564
variable  _ZTISt16invalid_argument                                                                                         0x72d3cbbf20
function  __cxa_uncaught_exceptions                                                                                        0x72d3c99d24
function  _ZNSt11logic_erroraSERKS_                                                                                        0x72d3c993f8
function  _ZNSt13runtime_errorC1ERKNSt6__ndk112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE                     0x72d3c99458
function  _ZdlPvSt11align_val_tRKSt9nothrow_t                                                                              0x72d3c992a0
function  __cxa_current_exception_type                                                                                     0x72d3c999b0
function  _ZNSt13bad_exceptionD2Ev                                                                                         0x72d3c9af34
function  _ZN7_JNIEnv12NewStringUTFEPKc                                                                                    0x72d3c9826c
variable  _ZTIa                                                                                                            0x72d3cbe158
function  _ZSt10unexpectedv                                                                                                0x72d3c99ee0
variable  _ZTIb                                                                                                            0x72d3cbe018
variable  _ZTSN10__cxxabiv121__vmi_class_type_infoE                                                                        0x72d3cb4af8
function  _ZNKSt6__ndk121__basic_string_commonILb1EE20__throw_length_errorEv                                               0x72d3c98a28
variable  _ZTIc                                                                                                            0x72d3cbe0b8
variable  _ZTId                                                                                                            0x72d3cbe568
variable  _ZTIe                                                                                                            0x72d3cbe5b8
variable  _ZTIf                                                                                                            0x72d3cbe518
variable  _ZTIg                                                                                                            0x72d3cbe608
variable  _ZTSN10__cxxabiv129__pointer_to_member_type_infoE                                                                0x72d3cb4932
variable  _ZTIh                                                                                                            0x72d3cbe108
variable  _ZTIi                                                                                                            0x72d3cbe248
function  __cxa_rethrow_primary_exception                                                                                  0x72d3c99b70
function  _ZNSt9type_infoD0Ev                                                                                              0x72d3caedf8
variable  _ZTIj                                                                                                            0x72d3cbe298
variable  _ZTSSt15underflow_error                                                                                          0x72d3cb2a80
variable  _ZTIl                                                                                                            0x72d3cbe2e8
variable  _ZTVN10__cxxabiv120__si_class_type_infoE                                                                         0x72d3cbe870
variable  _ZTIm                                                                                                            0x72d3cbe338
function  _ZNSt12domain_errorD1Ev                                                                                          0x72d3c9afa0
function  _ZnwmRKSt9nothrow_t                                                                                              0x72d3c99120
variable  _ZTIn                                                                                                            0x72d3cbe428
function  _ZNSt6__ndk112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE10__align_itILm16EEEmm                         0x72d3c98eb8
variable  _ZTVSt8bad_cast                                                                                                  0x72d3cbe9e8
variable  _ZTISt10bad_typeid                                                                                               0x72d3cbea80
function  _ZNSt8bad_castD0Ev                                                                                               0x72d3caee14
variable  _ZTIo                                                                                                            0x72d3cbe478
function  _ZNSt9exceptionD0Ev                                                                                              0x72d3c9af38
function  _ZNSt10bad_typeidC1Ev                                                                                            0x72d3caee44
variable  _ZTSPKa                                                                                                          0x72d3cb49de
variable  _ZTSSt10bad_typeid                                                                                               0x72d3cb4b55
variable  _ZTSPKb                                                                                                          0x72d3cb49ba
variable  _ZTIs                                                                                                            0x72d3cbe1a8
variable  _ZTSPKc                                                                                                          0x72d3cb49cc
variable  _ZTIt                                                                                                            0x72d3cbe1f8
function  _ZSt13set_terminatePFvvE                                                                                         0x72d3c9b4ac
function  _ZN7_JavaVM6GetEnvEPPvi                                                                                          0x72d3c984ac
variable  _ZTSPKd                                                                                                          0x72d3cb4a56
function  _ZNSt6__ndk112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEEC2IDnEEPKc                                     0x72d3c98214
variable  _ZTSPKe                                                                                                          0x72d3cb4a5f
function  _ZSt17__throw_bad_allocv                                                                                         0x72d3c99088
variable  _ZTIv                                                                                                            0x72d3cbdf78
function  _ZNSt13runtime_errorC2EPKc                                                                                       0x72d3c994e4
variable  _ZTSPKf                                                                                                          0x72d3cb4a4d
variable  _ZTSPKg                                                                                                          0x72d3cb4a68
variable  _ZTIw                                                                                                            0x72d3cbe068
variable  _ZTSPa                                                                                                           0x72d3cb49db
variable  _ZTSSt20bad_array_new_length                                                                                     0x72d3cb29da
function  _ZnamRKSt9nothrow_t                                                                                              0x72d3c99150
variable  _ZTVSt16invalid_argument                                                                                         0x72d3cbbef8
variable  _ZTSPKh                                                                                                          0x72d3cb49d5
variable  _ZTVSt12length_error                                                                                             0x72d3cbbf38
variable  _ZTSPb                                                                                                           0x72d3cb49b7
variable  _ZTIx                                                                                                            0x72d3cbe388
variable  _ZTIy                                                                                                            0x72d3cbe3d8
variable  _ZTSPc                                                                                                           0x72d3cb49c9
variable  _ZTSPKi                                                                                                          0x72d3cb49f9
variable  _ZTSPKj                                                                                                          0x72d3cb4a02
variable  _ZTSPd                                                                                                           0x72d3cb4a53
variable  _ZTIN10__cxxabiv121__vmi_class_type_infoE                                                                        0x72d3cbe928
variable  _ZTSPe                                                                                                           0x72d3cb4a5c
variable  _ZTIN10__cxxabiv123__fundamental_type_infoE                                                                      0x72d3cbdf60
function  _ZNKSt10bad_typeid4whatEv                                                                                        0x72d3caee80
variable  _ZTSPKl                                                                                                          0x72d3cb4a0b
variable  _ZTSPf                                                                                                           0x72d3cb4a4a
function  _ZNKSt8bad_cast4whatEv                                                                                           0x72d3caee38
function  _ZNSt20bad_array_new_lengthD0Ev                                                                                  0x72d3c9af90
variable  _ZTSPKm                                                                                                          0x72d3cb4a14
variable  _ZTSPg                                                                                                           0x72d3cb4a65
variable  _ZTVN10__cxxabiv117__class_type_infoE                                                                            0x72d3cbe820
function  _ZNSt14overflow_errorD0Ev                                                                                        0x72d3c9b2dc
function  _ZSt9terminatev                                                                                                  0x72d3c99e88
variable  _ZTSPKn                                                                                                          0x72d3cb4a2f
variable  _ZTSPh                                                                                                           0x72d3cb49d2
variable  _ZTSN10__cxxabiv117__pbase_type_infoE                                                                            0x72d3cb48c7
variable  _ZTISt13bad_exception                                                                                            0x72d3cbbe08
variable  _ZTSPi                                                                                                           0x72d3cb49f6
variable  _ZTSSt9type_info                                                                                                 0x72d3cb4b3c
variable  _ZTSPKo                                                                                                          0x72d3cb4a38
variable  _ZTSPj                                                                                                           0x72d3cb49ff
variable  _ZTVN10__cxxabiv120__function_type_infoE                                                                         0x72d3cbe798
variable  _ZTSPl                                                                                                           0x72d3cb4a08
variable  _ZTIN10__cxxabiv117__pbase_type_infoE                                                                            0x72d3cbde90
variable  _ZTSPm                                                                                                           0x72d3cb4a11
function  _ZnwmSt11align_val_t                                                                                             0x72d3c99194
variable  _ZTIN10__cxxabiv116__shim_type_infoE                                                                             0x72d3cbde60
variable  _ZTSPKs                                                                                                          0x72d3cb49e7
variable  _ZTSPn                                                                                                           0x72d3cb4a2c
variable  _ZTSPKt                                                                                                          0x72d3cb49f0
variable  _ZTSPo                                                                                                           0x72d3cb4a35
function  _ZSt14get_unexpectedv                                                                                            0x72d3c99e78
variable  _ZTSPKv                                                                                                          0x72d3cb49a5
variable  _ZTSSt9exception                                                                                                 0x72d3cb29ae
variable  _ZTSPKw                                                                                                          0x72d3cb49c3
variable  _ZTSPKx                                                                                                          0x72d3cb4a1d
variable  _ZTSPs                                                                                                           0x72d3cb49e4
function  _ZNKSt20bad_array_new_length4whatEv                                                                              0x72d3c9af94
variable  _ZTSPKy                                                                                                          0x72d3cb4a26
variable  _ZTSPt                                                                                                           0x72d3cb49ed
function  __cxa_increment_exception_refcount                                                                               0x72d3c99ae0
variable  _ZTSPv                                                                                                           0x72d3cb49a2
variable  _ZTSPw                                                                                                           0x72d3cb49c0
function  _ZNSt6__ndk111char_traitsIcE4copyEPcPKcm                                                                         0x72d3c98c0c
variable  _ZTSPx                                                                                                           0x72d3cb4a1a
function  _ZNSt9type_infoD1Ev                                                                                              0x72d3caedf4
variable  _ZTSPy                                                                                                           0x72d3cb4a23
function  _ZNSt6__ndk112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE6__initEPKcm                                   0x72d3c98840
function  _ZNSt12domain_errorD2Ev                                                                                          0x72d3c9afa0
variable  _ZTSSt13bad_exception                                                                                            0x72d3cb29bb
function  _ZNSt8bad_castD1Ev                                                                                               0x72d3caee10
function  _ZNSt13runtime_errorC1ERKS_                                                                                      0x72d3c99564
function  _ZNSt9exceptionD1Ev                                                                                              0x72d3c9af34
variable  _ZTSSt12length_error                                                                                             0x72d3cb2a29
function  _ZNSt10bad_typeidC2Ev                                                                                            0x72d3caee44
variable  _ZTIDh                                                                                                           0x72d3cbe4c8
variable  _ZTIDi                                                                                                           0x72d3cbe6f8
variable  _ZTSPDh                                                                                                          0x72d3cb4a3f
variable  _ZTSPDi                                                                                                          0x72d3cb4a87
function  __cxa_uncaught_exception                                                                                         0x72d3c99cfc
function  _ZNSt13runtime_errorD0Ev                                                                                         0x72d3c9b0ac
variable  _ZTVN10__cxxabiv116__shim_type_infoE                                                                             0x72d3cbdef0
variable  _ZTIDn                                                                                                           0x72d3cbdfc8
variable  _ZTSPDn                                                                                                          0x72d3cb49ac
function  _ZdaPvmSt11align_val_t                                                                                           0x72d3c992b4
function  _ZNSt6__ndk117_DeallocateCaller9__do_callEPv                                                                     0x72d3c98778
variable  _ZTIDs                                                                                                           0x72d3cbe6a8
function  _ZNSt20bad_array_new_lengthD1Ev                                                                                  0x72d3c9af34
variable  _ZTVSt10bad_typeid                                                                                               0x72d3cbea10
function  _ZdaPvRKSt9nothrow_t                                                                                             0x72d3c9918c
function  _ZNSt14overflow_errorD1Ev                                                                                        0x72d3c9b058
variable  _ZTIDu                                                                                                           0x72d3cbe658
variable  _ZTSPDs                                                                                                          0x72d3cb4a7b
variable  _ZTSa                                                                                                            0x72d3cb49d9
variable  _ZTIN10__cxxabiv119__pointer_type_infoE                                                                          0x72d3cbdea8
function  _ZNSt15underflow_errorD0Ev                                                                                       0x72d3c9b338
variable  _ZTSPDu                                                                                                          0x72d3cb4a6f
variable  _ZTVSt14overflow_error                                                                                           0x72d3cbc010
variable  _ZTSb                                                                                                            0x72d3cb49b5
function  _ZN7_JNIEnv9FindClassEPKc                                                                                        0x72d3c98534
variable  _ZTSc                                                                                                            0x72d3cb49c7
variable  _ZTSd                                                                                                            0x72d3cb4a51
function  _ZNSt9bad_allocD0Ev                                                                                              0x72d3c9af6c
variable  _ZTSe                                                                                                            0x72d3cb4a5a
variable  _ZTSf                                                                                                            0x72d3cb4a48
variable  _ZTSg                                                                                                            0x72d3cb4a63
variable  _ZTSSt13runtime_error                                                                                            0x72d3cb2a5b
variable  _ZTSh                                                                                                            0x72d3cb49d0
variable  _ZTSi                                                                                                            0x72d3cb49f4
variable  _ZTSN10__cxxabiv119__pointer_type_infoE                                                                          0x72d3cb48e9
variable  _ZTSj                                                                                                            0x72d3cb49fd
function  __cxa_rethrow                                                                                                    0x72d3c999fc
variable  _ZTISt12domain_error                                                                                             0x72d3cbbee0
variable  _ZTSl                                                                                                            0x72d3cb4a06
variable  _ZTSm                                                                                                            0x72d3cb4a0f
variable  _ZTISt9type_info                                                                                                 0x72d3cbea58
function  __cxa_end_catch                                                                                                  0x72d3c99884
variable  _ZTIN10__cxxabiv120__si_class_type_infoE                                                                         0x72d3cbe8c0
variable  _ZTSn                                                                                                            0x72d3cb4a2a
function  _ZNSt6__ndk111char_traitsIcE6lengthEPKc                                                                          0x72d3c98974
variable  _ZTSo                                                                                                            0x72d3cb4a33
function  _ZNSt13runtime_erroraSERKS_                                                                                      0x72d3c99594
function  _ZSt13get_terminatev                                                                                             0x72d3c99ef8
function  _ZNSt9type_infoD2Ev                                                                                              0x72d3caedf4
variable  __cxa_unexpected_handler                                                                                         0x72d3cbf010
variable  _ZTSSt9bad_alloc                                                                                                 0x72d3cb29cd
function  _ZSt15set_new_handlerPFvvE                                                                                       0x72d3c99f48
variable  _ZTSs                                                                                                            0x72d3cb49e2
variable  _ZTSt                                                                                                            0x72d3cb49eb
function  _ZNSt11logic_errorD0Ev                                                                                           0x72d3c9aff4
variable  _ZTISt9exception                                                                                                 0x72d3cbbdd0
variable  _ZTISt13runtime_error                                                                                            0x72d3cbbfe0
function  _ZNSt8bad_castD2Ev                                                                                               0x72d3caee10
variable  _ZTSv                                                                                                            0x72d3cb49a0
function  _ZNSt9exceptionD2Ev                                                                                              0x72d3c9af34
function  _ZNSt11logic_errorC2ERKNSt6__ndk112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE                       0x72d3c992bc
variable  _ZTSw                                                                                                            0x72d3cb49be
variable  _ZTVN10__cxxabiv117__array_type_infoE                                                                            0x72d3cbe748
function  _ZNSt11range_errorD0Ev                                                                                           0x72d3c9b280
function  __dynamic_cast                                                                                                   0x72d3cae0bc
function  _ZNSt12out_of_rangeD0Ev                                                                                          0x72d3c9b224
variable  _ZTSx                                                                                                            0x72d3cb4a18
variable  _ZTSy                                                                                                            0x72d3cb4a21
variable  _ZTSSt8bad_cast                                                                                                  0x72d3cb4b49
function  _ZNSt13runtime_errorD1Ev                                                                                         0x72d3c9b058
variable  __cxa_terminate_handler                                                                                          0x72d3cbf008
variable  _ZTIN10__cxxabiv120__function_type_infoE                                                                         0x72d3cbdec0
function  Java_com_example_jnitest_MainActivity_stringFromJNI                                                              0x72d3c9815c
function  _ZNSt20bad_array_new_lengthD2Ev                                                                                  0x72d3c9af34
function  _ZNSt14overflow_errorD2Ev                                                                                        0x72d3c9b058
function  _ZNSt16invalid_argumentD0Ev                                                                                      0x72d3c9b16c
function  __cxa_allocate_exception                                                                                         0x72d3c99630
function  _ZNSt15underflow_errorD1Ev                                                                                       0x72d3c9b058
variable  _ZTSN10__cxxabiv120__si_class_type_infoE                                                                         0x72d3cb4ad3
function  _ZNSt9bad_allocD1Ev                                                                                              0x72d3c9af34
variable  _ZTIN10__cxxabiv116__enum_type_infoE                                                                             0x72d3cbe808
function  _ZdaPv                                                                                                           0x72d3c99188
function  _ZNKSt11logic_error4whatEv                                                                                       0x72d3c9b050
variable  _ZTSN10__cxxabiv117__class_type_infoE                                                                            0x72d3cb48a5
function  _ZN10__cxxabiv119__getExceptionClassEPK17_Unwind_Exception                                                       0x72d3c99608
function  __cxa_get_globals                                                                                                0x72d3c99d44
function  _ZNSt6__ndk117__compressed_pairINS_12basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE5__repES5_EC2ILb1EvEEv  0x72d3c9880c
function  __cxa_get_exception_ptr                                                                                          0x72d3c997e8
variable  _ZTISt14overflow_error                                                                                           0x72d3cbc038
function  __cxa_pure_virtual                                                                                               0x72d3caee8c
function  _ZNSt11logic_errorD1Ev                                                                                           0x72d3c9afa0
variable  _ZTIPKDh                                                                                                         0x72d3cbe4f8
function  _Z3sI3P7_JNIEnvP8_jobject                                                                                        0x72d3c9833c
variable  _ZTIPKDi                                                                                                         0x72d3cbe728
function  _Znwm                                                                                                            0x72d3c990bc
variable  _ZTSN10__cxxabiv120__function_type_infoE                                                                         0x72d3cb490d
function  _ZNSt11range_errorD1Ev                                                                                           0x72d3c9b058
function  _ZNSt12out_of_rangeD1Ev                                                                                          0x72d3c9afa0
variable  _ZTSSt11logic_error                                                                                              0x72d3cb2a04
variable  _ZTVSt12domain_error                                                                                             0x72d3cbbea0
variable  _ZTIPKDn                                                                                                         0x72d3cbdff8
variable  _ZTVN10__cxxabiv119__pointer_type_infoE                                                                          0x72d3cbe978
function  _ZNSt13runtime_errorD2Ev                                                                                         0x72d3c9b058
variable  _ZTISt15underflow_error                                                                                          0x72d3cbc078
function  __cxa_decrement_exception_refcount                                                                               0x72d3c99960
variable  _ZTVN10__cxxabiv116__enum_type_infoE                                                                             0x72d3cbe7d0
variable  _ZTISt9bad_alloc                                                                                                 0x72d3cbbe20
function  __cxa_allocate_dependent_exception                                                                               0x72d3c99698
variable  _ZTIPKDs                                                                                                         0x72d3cbe6d8
function  _ZnamSt11align_val_t                                                                                             0x72d3c9926c
variable  _ZTIPKDu                                                                                                         0x72d3cbe688
variable  _ZTISt20bad_array_new_length                                                                                     0x72d3cbbe38
function  __cxa_demangle                                                                                                   0x72d3c9b610
variable  _ZTSPKDh                                                                                                         0x72d3cb4a43
variable  _ZTSPKDi                                                                                                         0x72d3cb4a8b
function  _ZNKSt9exception4whatEv                                                                                          0x72d3c9af3c
function  _ZNSt16invalid_argumentD1Ev                                                                                      0x72d3c9afa0
function  _ZNSt15underflow_errorD2Ev                                                                                       0x72d3c9b058
function  _ZSt15get_new_handlerv                                                                                           0x72d3c99f64
function  _ZNSt8bad_castC1Ev                                                                                               0x72d3caedfc
function  _ZNSt9bad_allocD2Ev                                                                                              0x72d3c9af34
variable  _ZTSPKDn                                                                                                         0x72d3cb49b0
function  _ZdlPvRKSt9nothrow_t                                                                                             0x72d3c99180
function  __cxa_deleted_virtual                                                                                            0x72d3caeea0
variable  _ZTIN10__cxxabiv129__pointer_to_member_type_infoE                                                                0x72d3cbded8
function  _ZNSt10bad_typeidD0Ev                                                                                            0x72d3caee5c
variable  _ZTSSt11range_error                                                                                              0x72d3cb2a4b
variable  _ZTSPKDs                                                                                                         0x72d3cb4a7f
variable  _ZTVSt13bad_exception                                                                                            0x72d3cbbde0
variable  _ZTSPKDu                                                                                                         0x72d3cb4a73
variable  _ZTIN10__cxxabiv117__class_type_infoE                                                                            0x72d3cbde78
function  _ZNKSt9bad_alloc4whatEv                                                                                          0x72d3c9af70
function  _ZNSt20bad_array_new_lengthC1Ev                                                                                  0x72d3c9af7c
variable  _ZTSDh                                                                                                           0x72d3cb4a3c
function  _ZNSt11logic_errorD2Ev                                                                                           0x72d3c9afa0
variable  _ZTSDi                                                                                                           0x72d3cb4a84
function  _ZNSt11logic_errorC1EPKc                                                                                         0x72d3c99348
function  _ZNSt11range_errorD2Ev                                                                                           0x72d3c9b058
function  _ZNSt12out_of_rangeD2Ev                                                                                          0x72d3c9afa0
variable  _ZTSDn                                                                                                           0x72d3cb49a9
function  __cxa_free_dependent_exception                                                                                   0x72d3c996cc
variable  _ZTSN10__cxxabiv116__shim_type_infoE                                                                             0x72d3cb4884
function  _ZNSt11logic_errorC2ERKS_                                                                                        0x72d3c993c8
variable  _ZTIPKa                                                                                                          0x72d3cbe188
variable  _ZTIPKb                                                                                                          0x72d3cbe048
variable  _ZTISt12out_of_range                                                                                             0x72d3cbbfa0
variable  _ZTIPKc                                                                                                          0x72d3cbe0e8
variable  _ZTIPKd                                                                                                          0x72d3cbe598
function  _ZdaPvSt11align_val_tRKSt9nothrow_t                                                                              0x72d3c992b0
variable  _ZTSDs                                                                                                           0x72d3cb4a78
variable  _ZTIPKe                                                                                                          0x72d3cbe5e8
variable  _ZTIPKf                                                                                                          0x72d3cbe548
variable  _ZTSDu                                                                                                           0x72d3cb4a6c
variable  _ZTVN10__cxxabiv117__pbase_type_infoE                                                                            0x72d3cbe940
function  _ZnwmSt11align_val_tRKSt9nothrow_t                                                                               0x72d3c99240
function  _ZNKSt13runtime_error4whatEv                                                                                     0x72d3c9b108
variable  _ZTIPKg                                                                                                          0x72d3cbe638
variable  _ZSt7nothrow                                                                                                     0x72d3cb271a
variable  _ZTIPKh                                                                                                          0x72d3cbe138
function  __cxa_current_primary_exception                                                                                  0x72d3c99afc
function  _ZNSt12length_errorD0Ev                                                                                          0x72d3c9b1c8
variable  _ZTISt11logic_error                                                                                              0x72d3cbbec8
variable  _ZTIPKi                                                                                                          0x72d3cbe278
variable  _ZTVSt20bad_array_new_length                                                                                     0x72d3cbbd80
function  _ZNSt13runtime_errorC2ERKNSt6__ndk112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE                     0x72d3c99458
variable  _ZTIPKj                                                                                                          0x72d3cbe2c8
function  _ZNSt16invalid_argumentD2Ev                                                                                      0x72d3c9afa0
variable  _ZTIPa                                                                                                           0x72d3cbe168
variable  _ZTIPKl                                                                                                          0x72d3cbe318
variable  _ZTIPb                                                                                                           0x72d3cbe028
variable  _ZTIPKm                                                                                                          0x72d3cbe368
variable  _ZTSSt14overflow_error                                                                                           0x72d3cb2a6d
variable  _ZTIPKn                                                                                                          0x72d3cbe458
variable  _ZTIPc                                                                                                           0x72d3cbe0c8
variable  _ZTIPKo                                                                                                          0x72d3cbe4a8
variable  _ZTIPd                                                                                                           0x72d3cbe578
function  _ZNSt8bad_castC2Ev                                                                                               0x72d3caedfc
variable  _ZTIPe                                                                                                           0x72d3cbe5c8
variable  _ZTIPf                                                                                                           0x72d3cbe528
function  _ZNSt10bad_typeidD1Ev                                                                                            0x72d3caee58
variable  _ZTIPg                                                                                                           0x72d3cbe618
variable  _ZTIPKs                                                                                                          0x72d3cbe1d8
variable  _ZTVSt12out_of_range                                                                                             0x72d3cbbf78
variable  _ZTIPh                                                                                                           0x72d3cbe118
variable  _ZTIPKt                                                                                                          0x72d3cbe228
function  _ZNSt13bad_exceptionD0Ev                                                                                         0x72d3c9af48
variable  _ZTIPi                                                                                                           0x72d3cbe258
variable  _ZTIPj                                                                                                           0x72d3cbe2a8
variable  _ZTIPKv                                                                                                          0x72d3cbdfa8
function  _Znam                                                                                                            0x72d3c9914c
variable  _ZTIPKw                                                                                                          0x72d3cbe098
variable  _ZTIPl                                                                                                           0x72d3cbe2f8
function  _ZN7_JNIEnv15RegisterNativesEP7_jclassPK15JNINativeMethodi                                                       0x72d3c984ec
variable  _ZTIPKx                                                                                                          0x72d3cbe3b8
function  _ZdlPvSt11align_val_t                                                                                            0x72d3c9929c
variable  _ZTIPm                                                                                                           0x72d3cbe348
variable  _ZTIPKy                                                                                                          0x72d3cbe408
variable  _ZTSSt12domain_error                                                                                             0x72d3cb29f3
variable  _ZTIPn                                                                                                           0x72d3cbe438
function  _ZNSt6__ndk112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEED2Ev                                           0x72d3c982c8
variable  _ZTIPo                                                                                                           0x72d3cbe488
function  _ZNSt20bad_array_new_lengthC2Ev                                                                                  0x72d3c9af7c
variable  _ZTIPs                                                                                                           0x72d3cbe1b8
variable  _ZTISt11range_error                                                                                              0x72d3cbbff8
variable  _ZTVSt9type_info                                                                                                 0x72d3cbea38
variable  _ZTIPt                                                                                                           0x72d3cbe208
function  JNI_OnLoad                                                                                                       0x72d3c983f4
function  __cxa_begin_catch                                                                                                0x72d3c997f0
variable  _ZTIPv                                                                                                           0x72d3cbdf88
variable  _ZTIPw                                                                                                           0x72d3cbe078
function  __cxa_get_globals_fast                                                                                           0x72d3c99dd4
function  _ZNSt9bad_allocC1Ev                                                                                              0x72d3c9af58
variable  _ZTIPx                                                                                                           0x72d3cbe398
variable  _ZTIPy                                                                                                           0x72d3cbe3e8
variable  _ZTVSt11logic_error                                                                                              0x72d3cbbe50
function  _ZN10__cxxabiv119__setExceptionClassEP17_Unwind_Exceptionm                                                       0x72d3c995f4
variable  _ZTVSt9exception                                                                                                 0x72d3cbbda8
function  _ZNSt11logic_errorC1ERKNSt6__ndk112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE                       0x72d3c992bc
function  _ZdaPvm                                                                                                          0x72d3c99190
variable  _ZTIPDh                                                                                                          0x72d3cbe4d8
variable  _ZTIPDi                                                                                                          0x72d3cbe708
variable  _ZTVSt13runtime_error                                                                                            0x72d3cbbe78

```

这是如果需要进行Hook需要通过第二种查找函数地址的方式，先找到模块基地址，再通过相对地址找到函数的地址。

通过Hook实现动态注册的函数RegisterNatives()来获取动 态注册后的sI3()函数地址。

一个用得到的项目 `https://github.com/lasting-yang/frida_hook_libart`

```
(base) ┌──(root㉿kali)-[/home/zhy/桌面/frida_hook_libart-master]
└─# frida -U -f com.example.jnitest -l hook_RegisterNatives.js --no-pause
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
Spawning `com.example.jnitest`...                                       
RegisterNatives is at  0x72e97414d8 _ZN3art3JNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi
Spawned `com.example.jnitest`. Resuming main thread!                    
[Nexus 5X::com.example.jnitest ]-> [RegisterNatives] method_count: 0x1
[RegisterNatives] java_class: com.example.jnitest.MainActivity name: stringFromJNI3 sig: ()Ljava/lang/String; fnPtr: 0x72d225133c  fnOffset: 0x72d225133c libjnitest.so!_Z3sI3P7_JNIEnvP8_jobject  callee: 0x72e94c69c0 libart.so!_ZN3art8CheckJNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi+0x2a4

```

获取到函数偏移后便可以利用上述第二种查找函数地址的方式 获取相应函数地址并对函数进行Hook，最终的Hook脚本内容如下：

但是从官网下载下来的脚本有变化，所以需要修改成一下的脚本

```js
var ishook_libart = false;

function hook_libart() {
    if (ishook_libart === true) {
        return;
    }
    var symbols = Module.enumerateSymbolsSync("libart.so");
    var addrGetStringUTFChars = null;
    var addrNewStringUTF = null;
    var addrFindClass = null;
    var addrGetMethodID = null;
    var addrGetStaticMethodID = null;
    var addrGetFieldID = null;
    var addrGetStaticFieldID = null;
    var addrRegisterNatives = null;
    var addrAllocObject = null;
    var addrCallObjectMethod = null;
    var addrGetObjectClass = null;
    var addrReleaseStringUTFChars = null;
    for (var i = 0; i < symbols.length; i++) {
        var symbol = symbols[i];
        if (symbol.name == "_ZN3art3JNI17GetStringUTFCharsEP7_JNIEnvP8_jstringPh") {
            addrGetStringUTFChars = symbol.address;
            console.log("GetStringUTFChars is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI12NewStringUTFEP7_JNIEnvPKc") {
            addrNewStringUTF = symbol.address;
            console.log("NewStringUTF is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI9FindClassEP7_JNIEnvPKc") {
            addrFindClass = symbol.address;
            console.log("FindClass is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI11GetMethodIDEP7_JNIEnvP7_jclassPKcS6_") {
            addrGetMethodID = symbol.address;
            console.log("GetMethodID is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI17GetStaticMethodIDEP7_JNIEnvP7_jclassPKcS6_") {
            addrGetStaticMethodID = symbol.address;
            console.log("GetStaticMethodID is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI10GetFieldIDEP7_JNIEnvP7_jclassPKcS6_") {
            addrGetFieldID = symbol.address;
            console.log("GetFieldID is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI16GetStaticFieldIDEP7_JNIEnvP7_jclassPKcS6_") {
            addrGetStaticFieldID = symbol.address;
            console.log("GetStaticFieldID is at ", symbol.address, symbol.name);
        } else if (symbol.name == "_ZN3art3JNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi") {
            addrRegisterNatives = symbol.address;
            console.log("RegisterNatives is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI11AllocObjectEP7_JNIEnvP7_jclass") >= 0) {
            addrAllocObject = symbol.address;
            console.log("AllocObject is at ", symbol.address, symbol.name);
        }  else if (symbol.name.indexOf("_ZN3art3JNI16CallObjectMethodEP7_JNIEnvP8_jobjectP10_jmethodIDz") >= 0) {
            addrCallObjectMethod = symbol.address;
            console.log("CallObjectMethod is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI14GetObjectClassEP7_JNIEnvP8_jobject") >= 0) {
            addrGetObjectClass = symbol.address;
            console.log("GetObjectClass is at ", symbol.address, symbol.name);
        } else if (symbol.name.indexOf("_ZN3art3JNI21ReleaseStringUTFCharsEP7_JNIEnvP8_jstringPKc") >= 0) {
            addrReleaseStringUTFChars = symbol.address;
            console.log("ReleaseStringUTFChars is at ", symbol.address, symbol.name);
        }
    }

    if (addrRegisterNatives != null) {
        Interceptor.attach(addrRegisterNatives, {
            onEnter: function (args) {
                console.log("[RegisterNatives] method_count:", args[3]);
                var env = args[0];
                var java_class = args[1];
                
                var funcAllocObject = new NativeFunction(addrAllocObject, "pointer", ["pointer", "pointer"]);
                var funcGetMethodID = new NativeFunction(addrGetMethodID, "pointer", ["pointer", "pointer", "pointer", "pointer"]);
                var funcCallObjectMethod = new NativeFunction(addrCallObjectMethod, "pointer", ["pointer", "pointer", "pointer"]);
                var funcGetObjectClass = new NativeFunction(addrGetObjectClass, "pointer", ["pointer", "pointer"]);
                var funcGetStringUTFChars = new NativeFunction(addrGetStringUTFChars, "pointer", ["pointer", "pointer", "pointer"]);
                var funcReleaseStringUTFChars = new NativeFunction(addrReleaseStringUTFChars, "void", ["pointer", "pointer", "pointer"]);

                var clz_obj = funcAllocObject(env, java_class);
                var mid_getClass = funcGetMethodID(env, java_class, Memory.allocUtf8String("getClass"), Memory.allocUtf8String("()Ljava/lang/Class;"));
                var clz_obj2 = funcCallObjectMethod(env, clz_obj, mid_getClass);
                var cls = funcGetObjectClass(env, clz_obj2);
                var mid_getName = funcGetMethodID(env, cls, Memory.allocUtf8String("getName"), Memory.allocUtf8String("()Ljava/lang/String;"));
                var name_jstring = funcCallObjectMethod(env, clz_obj2, mid_getName);
                var name_pchar = funcGetStringUTFChars(env, name_jstring, ptr(0));
                var class_name = ptr(name_pchar).readCString();
                funcReleaseStringUTFChars(env, name_jstring, name_pchar);

                //console.log(class_name);

                var methods_ptr = ptr(args[2]);

                var method_count = parseInt(args[3]);
                for (var i = 0; i < method_count; i++) {
                    var name_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3));
                    var sig_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3 + Process.pointerSize));
                    var fnPtr_ptr = Memory.readPointer(methods_ptr.add(i * Process.pointerSize * 3 + Process.pointerSize * 2));

                    var name = Memory.readCString(name_ptr);
                    var sig = Memory.readCString(sig_ptr);
                    var find_module = Process.findModuleByAddress(fnPtr_ptr);
                    console.log("[RegisterNatives] java_class:", class_name, "name:", name, "sig:", sig, "fnPtr:", fnPtr_ptr, "module_name:", find_module.name, "module_base:", find_module.base, "offset:", ptr(fnPtr_ptr).sub(find_module.base));

                }
            },
            onLeave: function (retval) { }
        });
    }

    ishook_libart = true;
}

hook_libart();
```

```sh
(base) ┌──(root㉿kali)-[/home/zhy/桌面/NativeHook]
└─# frida -U --no-pause -f com.example.jnitest -l hook_RegisterNatives.js
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
Spawning `com.example.jnitest`...                                       
GetFieldID is at  0x72e9722ae8 _ZN3art3JNI10GetFieldIDEP7_JNIEnvP7_jclassPKcS6_
AllocObject is at  0x72e9705470 _ZN3art3JNI11AllocObjectEP7_JNIEnvP7_jclass
GetMethodID is at  0x72e97073c4 _ZN3art3JNI11GetMethodIDEP7_JNIEnvP7_jclassPKcS6_
NewStringUTF is at  0x72e973ed10 _ZN3art3JNI12NewStringUTFEP7_JNIEnvPKc
GetObjectClass is at  0x72e9706818 _ZN3art3JNI14GetObjectClassEP7_JNIEnvP8_jobject
RegisterNatives is at  0x72e97414d8 _ZN3art3JNI15RegisterNativesEP7_JNIEnvP7_jclassPK15JNINativeMethodi
CallObjectMethod is at  0x72e9707af8 _ZN3art3JNI16CallObjectMethodEP7_JNIEnvP8_jobjectP10_jmethodIDz
GetStaticFieldID is at  0x72e9736fa4 _ZN3art3JNI16GetStaticFieldIDEP7_JNIEnvP7_jclassPKcS6_
GetStaticMethodID is at  0x72e9729420 _ZN3art3JNI17GetStaticMethodIDEP7_JNIEnvP7_jclassPKcS6_
GetStringUTFChars is at  0x72e973f680 _ZN3art3JNI17GetStringUTFCharsEP7_JNIEnvP8_jstringPh
ReleaseStringUTFChars is at  0x72e973fbcc _ZN3art3JNI21ReleaseStringUTFCharsEP7_JNIEnvP8_jstringPKc
FindClass is at  0x72e96fee94 _ZN3art3JNI9FindClassEP7_JNIEnvPKc
Spawned `com.example.jnitest`. Resuming main thread!                    
[Nexus 5X::com.example.jnitest ]-> [RegisterNatives] method_count: 0x1
[RegisterNatives] java_class: com.example.jnitest.MainActivity name: stringFromJNI3 sig: ()Ljava/lang/String; fnPtr: 0x72d224f33c module_name: libjnitest.so module_base: 0x72d2240000 offset: 0xf33c

```

根据偏移量0xf33c编写以下脚本：

```js
function hook_native3(){
    var addr = Module.findBaseAddress('libjnitest.so');
    console.log("libjnitest addr is=>",addr)
    var stringfromJNI3 = addr.add(0xf33c)
    console.log("stringfromJNI3 address is=>",stringfromJNI3)
    Interceptor.attach(stringfromJNI3,{
        onEnter:function(args){
            console.log("jinenv pointer=>",args[0])
            console.log("jobj pointer=>",args[1])
        },onLeave:function(retval){
            console.log("retval is =>",Java.vm.getEnv().getStringUtfChars(retval,null).readCString())
            console.log("================")
        }
    })
}
function main(){
    hook_native3()
}

setImmediate(main)
```

需要注意的是，虽然每次重新运行App时其函数的偏移地址是不 会改变的，但是这是建立在App未被重新编译的基础上的。如果在一 次运行后App代码发生修改，并重新在Android Studio中编译并运 行，那么新的App函数的偏移地址是会发生改变的，此时应当重新使 用hook_RegisterNative.js脚本获取相应函数的新偏移值并修改 Hook脚本，然后再进行注入。
