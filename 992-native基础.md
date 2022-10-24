# native基础

## NDK基础

Android开发中，除了使用Java语言进行开发并编译成dex文件由ART/Dalvik虚拟机执行的方式之外，还存在使用C/C++代码完成应用开发并编译未动态链接库文件由CPU直接执行的方式。

C/C++代码的开发通常被称为native原生库开发。

相较于Java语言在Android上的开发依赖于SDK的支持，C/C++语言则需要NDK开发套件的支持，实际上NDK就是在Android中使用C/C++语言进行开发的依赖库。

**JNI**（Java Native Interface）是使java方法和C/C++函数互通的一座桥梁。

JNI使JVM中的一部分，借助JNI可以在任何实现了JNI规范的Java虚拟机中调用C/C++函数，只是Android中的JNI实现更加方便和简单。

以来NDK开发套件的原因是因为

- 性能，由于Java语言编译而成的二进制dex文件最终是Android虚拟机上运行的，即由虚拟机执行的Java代码是通过CPU完成的，这就导致使用Java语言去执行逻辑的时候需要多一层翻译，进而对程序的性能产生影响。

- 安全性，一个未加任何保护的APP程序在使用Jadx或者Jeb等工具反编译后代码逻辑几乎没有任何阅读障碍，相比于Java，C/C++这种编译型语言在编译成可执行文件后，文件内就只剩机器码了，虽然机器码可以被反编译成汇编语言甚至通过IDA等反编译神器反编译成C/C++伪代码，但是这些反编译的结果相比于Java的反编译结果可以说是完全无法阅读。

  

![](https://raw.githubusercontent.com/G-WS/Android-reverse/main/image/NDK1.png)



![](https://raw.githubusercontent.com/G-WS/Android-reverse/main/image/NDK2.png)

在 static 静 态 代 码 块 中 ， System.loadLibrary() 函 数 用 于 将 native-lib这个目标动态库加载到内存中，这里的native-lib动态库名 称是由Cmakelists.txt文件设置的；最终生成的动态链接库名称为 “lib+目标名+.so”，这里的动态链接库名称为libnative-lib.so。

 native描述符修饰的stringFromJNI()函数事实上就是一个JNI函 数，其在Java层中通过native描述符修饰来声明，真实的函数内容是 由C/C++语言实现的。点击函数声明左侧的C++图标或者按住Ctrl键后点击函数名就会跳到对应的C/C++语言实现的函数主 体

```java
package com.example.jnitest;

import androidx.appcompat.app.AppCompatActivity;

import android.os.Bundle;
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
    }

    /**
     * A native method that is implemented by the 'jnitest' native library,
     * which is packaged with this application.
     */
    public native String stringFromJNI();
}
```

```c++
#include <jni.h>
#include <string>

extern "C" JNIEXPORT jstring JNICALL
Java_com_example_jnitest_MainActivity_stringFromJNI(
        JNIEnv* env,
        jobject /* this */) {
    std::string hello = "Hello from C++";
    return env->NewStringUTF(hello.c_str());
}
```

对比其C/C++函数声明的形式和Java函数声明的形式，native层 函数明显多出了几个参数和修饰符，甚至函数名称也变得特别长，函 数返回值类型也从String变成了jstring。之所以从简单的stringFromJNI变成了Java_com_roysue_r0so_MainActivity_stringFromJNI，这是因为 JNI函数的绑定需要依赖于一个函数命名规则，让Java层能够据此找 到对应的原生函数。对应其Java中的函数声明，会发现Java_是函数 的前缀，中间的com_roysue_r0so和MainActivity分别对应Java函 数的类所在包名（将“.”替换为“_”）和相应的类名，最后的 stringFromJNI字符串则与Java中的函数名相同，每个部分使用下画 线 “_” 连 接 ， 因此JNI函数的命名规则就呼之欲出了：Java_PackageName_ClassName_MethodName 即为MethodName对应的native层函数名。

JNIEXPORT的描述符用于表明这个JNI函数是一个导出 函数可供外部调用；JNICALL则是一个空的声明，extern "C"是表明以C语言命名方式编译，如果不加这个标志会默认按照C++语言的方式进行编译，最终会出现name mangling（名称粉碎或者名称修饰机制）使得编译后的函数名不再是源代码声明的模样，最终导致在Java曾进行函数调用时无法找到相应的native实现。

返回值类型之所以从String编程jstring，是为了放置Java中数据类型和C/C++语言的数据类型相互冲突。Java和JNI的一些数据类型的表示方式对照表如下：

| Java数据类型 | JNI数据类型 |
| ------------ | ----------- |
| boolean      | jboolean    |
| byte         | jbyte       |
| char         | jchar       |
| short        | jshort      |
| int          | jint        |
| long         | jlong       |
| float        | jfloat      |
| double       | jdouble     |
| void         | void        |
| String       | jstring     |
| Object       | jobject     |
| class        | jclass      |

```java
JNIEnv* env,
        jobject /* this */
```

函数的形参比Java层的声明多出两个参数（JNIEnv*和jobject），分别用来表示当前Java线程的执行环境和相应函数所在类的对象。通过过JNIEnv*声明的变量可以执行JNI函 数，调用Java层代码与Java层完成交互，比如在stringFromJNI这 个JNI函数中是通过JNIEnv*声明的env变量完成了jstring类型的字 符串变量的创建，其他可以通过JNIEnv声明变量实现的JNI函数可以 参考jni.h头文件。第二个参数取决于在Java层中stringFromJNI()函 数是否有static描述符修饰，如果存在static描述符修饰，那么第二个 变量类型将变为jclass，用于表示相应函数所在类，若没有static修饰 则是jobject类型。

JavaVM是Java虚拟机 在JNI层的代表，JNI全局仅仅有一个JavaVM结构，而JNIEnv是当 前Java线程的执行环境，每个线程都对应一个JNIEnv结构。作为 Java虚拟机的代表，JavaVM通常出现在so文件加载后第一个运行的 普通函数JNI_Onload()的参数中，JNIEnv则是每个JNI函数的第一 个参数，JNI_Onload()函数的声明同样可以在jni.h文件中找到，具 体如下：

```c
JNIEXPORT jint JNI_OnLoad(JavaVM* vm, void* reserved);

```

## JNI函数逆向的基本流程

通过上一小节编写的demo工程经过编译后生成APK文件，我们可以发现其文件结构中相比于没有C/C++语言支持的APK多出了一个lib目录，且在lib目录下存储着一个aarch64架构的elf动态链接库文件

```sh
(base) ┌──(root㉿kali)-[/home/zhy/桌面/native]
└─# tree . -L 1
.
├── AndroidManifest.xml
├── classes2.dex
├── classes3.dex
├── classes4.dex
├── classes.dex
├── lib
├── META-INF
├── res
└── resources.arsc

3 directories, 7 files

```

```sh
(base) ┌──(root㉿kali)-[/home/zhy/桌面/native]
└─# tree lib   
lib
├── arm64-v8a
│   └── libjnitest.so
├── armeabi-v7a
│   └── libjnitest.so
├── x86
│   └── libjnitest.so
└── x86_64
    └── libjnitest.so

4 directories, 4 files

```

```sh
(base) ┌──(root㉿kali)-[/home/zhy/桌面/native]
└─# file lib/arm64-v8a/libjnitest.so 
lib/arm64-v8a/libjnitest.so: ELF 64-bit LSB shared object, ARM aarch64, version 1 (SYSV), dynamically linked, BuildID[sha1]=023ee2ff41c0642ae6e91977c89133de26ec2923, stripped

```

目前Android编译仍旧支持的架构有 arm32、arm64、x86和x86_64。

架构是向下兼容的。简单来说，64位架构的系统可以 运行32位架构的应用，32位架构的系统却无法运行64位架构的应 用，比如arm64架构可以运行arm32的应用，而arm32不可以运行 arm64的应用。

在JNI逆向过程中，首先需要找到Java层函数在native层中对应 的函数地址，有了函数地址才能使用Frida进行Hook或者使用IDA等 逆向工作进行动态分析和静态分析。根据上面介绍的JNI函数命名规 则可知，Android系统为了快速找到JNI函数在native层的对应函数， 因而其命名规则都是固定的，并且在内存中也是不会发生变化的。 

使 用 Objection 注 入 demo 进 程 并 通 过 以 下 命 令 查 看 libnative-lib.so文件是否被加载到内存中：

```
(base) ┌──(root㉿kali)-[/home/zhy/桌面/native]
└─# frida-ps -U                                           
 PID  Name
----  ---------------------------------------------------
3375  ATFWD-daemon                                       
8839  Android Pay                                        
8330  F-Droid                                            
8880  Gmail                                              
5836  Google                                             
8031  Google Play 商店                                     
9327  JniTest                                            
8991  adbd                                               
3381  android.hardware.biometrics.fingerprint@2.1-service
 439  android.hardware.cas@1.0-service                   
 440  android.hardware.configstore@1.0-service           
 441  android.hardware.dumpstate@1.0-service.bullhead    
 442  android.hardware.graphics.allocator@2.0-service    
 443  android.hardware.usb@1.0-service                   
 444  android.hardware.wifi@1.0-service                  
 438  android.hidl.allocator@1.0-service                 
8993  android.process.media                              
3352  audioserver                                        
3355  cameraserver                                       
3376  cnd                                                
3372  cnss-daemon                                        
4047  com.android.bluetooth                              
9218  com.android.defcontainer                           
5864  com.android.nfc                                    
4221  com.android.phone                                  
7156  com.android.providers.calendar                     
4083  com.android.systemui                               
8894  com.google.android.apps.gcs                        
6185  com.google.android.gms                             
5809  com.google.android.gms.persistent                  
9009  com.google.android.gms.ui                          
8168  com.google.android.gms.unstable                    
6012  com.google.android.googlequicksearchbox            
5972  com.google.android.googlequicksearchbox:search     
7496  com.google.android.ims                             
6242  com.google.android.inputmethod.latin               
8046  com.google.process.gapps                           
5994  com.google.process.gservices                       
8374  com.qualcomm.qcrilmsgtunnel                        
5877  com.qualcomm.qti.rcsbootstraputil                  
5882  com.qualcomm.qti.rcsimsbootstraputil               
8349  com.qualcomm.telephony                             
5851  com.quicinc.cne.CNEService                         
9119  daemonsu:0                                         
9121  daemonsu:0:9116                                    
9137  daemonsu:0:9124                                    
9151  daemonsu:0:9124                                    
7175  daemonsu:10102                                     
3323  daemonsu:master                                    
7199  daemonsu:mount:0                                   
3316  daemonsu:mount:master                              
3357  drmserver                                          
9154  frida-helper-32                                    
9124  frida-server-15.2.2-android-arm64                  
3379  gatekeeperd                                        
 445  healthd                                            
 369  hwservicemanager                                   
3383  imsdatadaemon                                      
3348  imsqmidaemon                                       
   1  init                                               
3358  installd                                           
9194  installer                                          
3451  ip6tables-restore                                  
3449  iptables-restore                                   
3359  keystore                                           
 449  lmkd                                               
3371  loc_launcher                                       
3446  location-mq                                        
9138  logcat                                             
9389  logcat                                             
 367  logd                                               
3447  lowi-server                                        
3369  media.codec                                        
3363  media.extractor                                    
3364  media.metrics                                      
3362  mediadrmserver                                     
3365  mediaserver                                        
3374  mm-qcamera-daemon                                  
 448  msm_irqbalance                                     
3366  netd                                               
3346  netmgrd                                            
3344  perfd                                              
 502  pm-proxy                                           
 447  pm-service                                         
3343  qmuxd                                              
 374  qseecomd                                           
 380  qseecomd                                           
3345  qti                                                
3370  rild                                               
 446  rmt_storage                                        
 368  servicemanager                                     
9113  sh                                                 
9192  sh                                                 
3448  slim_daemon                                        
7433  sshd [listener] 0 of 10-100 startups               
3367  storaged                                           
9116  su                                                 
 450  surfaceflinger                                     
9122  sush                                               
9152  sush                                               
3760  system_server                                      
3347  thermal-engine                                     
 451  thermalserviced                                    
3373  time_daemon                                        
3380  tombstoned                                         
 349  ueventd                                            
 370  vndservicemanager                                  
 400  vold                                               
4636  wcnss_filter                                       
4134  webview_zygote32                                   
3368  wificond                                           
4467  wpa_supplicant                                     
3350  zygote                                             
3349  zygote64                                           
7442  信息                                                 
8773  地圖                                                 
7631  日历                                                 
9303  移动TV                                               
8966  设置                                                 
                                                                                                                                   
(base) ┌──(root㉿kali)-[/home/zhy/桌面/native]
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
com.example.jnitest on (google: 8.1.0) [usb] # memory list modules
Save the output by adding `--json modules.json` to this command
Name                                             Base          Size                  Path
-----------------------------------------------  ------------  --------------------  ------------------------------------------------------------------------------
app_process64                                    0x559317b000  32768 (32.0 KiB)      /system/bin/app_process64
libandroid_runtime.so                            0x719240d000  1982464 (1.9 MiB)     /system/lib64/libandroid_runtime.so
libbinder.so                                     0x7193096000  561152 (548.0 KiB)    /system/lib64/libbinder.so
libcutils.so                                     0x7191705000  77824 (76.0 KiB)      /system/lib64/libcutils.so
libhwbinder.so                                   0x7191582000  163840 (160.0 KiB)    /system/lib64/libhwbinder.so
liblog.so                                        0x7192c0b000  102400 (100.0 KiB)    /system/lib64/liblog.so
libnativeloader.so                               0x7190875000  28672 (28.0 KiB)      /system/lib64/libnativeloader.so
libutils.so                                      0x71901db000  131072 (128.0 KiB)    /system/lib64/libutils.so
libwilhelm.so                                    0x7193199000  274432 (268.0 KiB)    /system/lib64/libwilhelm.so
libc++.so                                        0x71938c2000  978944 (956.0 KiB)    /system/lib64/libc++.so
libc.so                                          0x7190287000  880640 (860.0 KiB)    /system/lib64/libc.so
libm.so                                          0x7191c05000  233472 (228.0 KiB)    /system/lib64/libm.so
libdl.so                                         0x7193859000  20480 (20.0 KiB)      /system/lib64/libdl.so
libmemtrack.so                                   0x71937c6000  20480 (20.0 KiB)      /system/lib64/libmemtrack.so
libandroidfw.so                                  0x71903e5000  339968 (332.0 KiB)    /system/lib64/libandroidfw.so
libappfuse.so                                    0x719204d000  57344 (56.0 KiB)      /system/lib64/libappfuse.so
libbase.so                                       0x7190be0000  73728 (72.0 KiB)      /system/lib64/libbase.so
libcrypto.so                                     0x7192aa7000  1150976 (1.1 MiB)     /system/lib64/libcrypto.so
libnativehelper.so                               0x71917b1000  36864 (36.0 KiB)      /system/lib64/libnativehelper.so
libdebuggerd_client.so                           0x7191a18000  28672 (28.0 KiB)      /system/lib64/libdebuggerd_client.so
libui.so                                         0x7191d81000  143360 (140.0 KiB)    /system/lib64/libui.so
libgraphicsenv.so                                0x71904b2000  16384 (16.0 KiB)      /system/lib64/libgraphicsenv.so
libgui.so                                        0x71935c7000  618496 (604.0 KiB)    /system/lib64/libgui.so
libsensor.so                                     0x7191681000  94208 (92.0 KiB)      /system/lib64/libsensor.so
libinput.so                                      0x7190984000  184320 (180.0 KiB)    /system/lib64/libinput.so
libcamera_client.so                              0x7191500000  327680 (320.0 KiB)    /system/lib64/libcamera_client.so
libcamera_metadata.so                            0x71908e2000  45056 (44.0 KiB)      /system/lib64/libcamera_metadata.so
libskia.so                                       0x7190c0b000  8257536 (7.9 MiB)     /system/lib64/libskia.so
libsqlite.so                                     0x71905e6000  1138688 (1.1 MiB)     /system/lib64/libsqlite.so
libEGL.so                                        0x7191d01000  192512 (188.0 KiB)    /system/lib64/libEGL.so
libGLESv1_CM.so                                  0x71921c8000  45056 (44.0 KiB)      /system/lib64/libGLESv1_CM.so
libGLESv2.so                                     0x7191800000  94208 (92.0 KiB)      /system/lib64/libGLESv2.so
libvulkan.so                                     0x7190586000  126976 (124.0 KiB)    /system/lib64/libvulkan.so
libziparchive.so                                 0x7192698000  53248 (52.0 KiB)      /system/lib64/libziparchive.so
libETC1.so                                       0x7193885000  20480 (20.0 KiB)      /system/lib64/libETC1.so
libhardware.so                                   0x71915df000  16384 (16.0 KiB)      /system/lib64/libhardware.so
libhardware_legacy.so                            0x7191956000  16384 (16.0 KiB)      /system/lib64/libhardware_legacy.so
libselinux.so                                    0x7191c46000  102400 (100.0 KiB)    /system/lib64/libselinux.so
libicuuc.so                                      0x71909ca000  1642496 (1.6 MiB)     /system/lib64/libicuuc.so
libmedia.so                                      0x7190703000  1097728 (1.0 MiB)     /system/lib64/libmedia.so
libaudioclient.so                                0x7191416000  434176 (424.0 KiB)    /system/lib64/libaudioclient.so
libjpeg.so                                       0x7193503000  294912 (288.0 KiB)    /system/lib64/libjpeg.so
libusbhost.so                                    0x7190554000  24576 (24.0 KiB)      /system/lib64/libusbhost.so
libharfbuzz_ng.so                                0x7190902000  503808 (492.0 KiB)    /system/lib64/libharfbuzz_ng.so
libz.so                                          0x7190222000  94208 (92.0 KiB)      /system/lib64/libz.so
libpdfium.so                                     0x718fb84000  4960256 (4.7 MiB)     /system/lib64/libpdfium.so
libimg_utils.so                                  0x71922d2000  86016 (84.0 KiB)      /system/lib64/libimg_utils.so
libnetd_client.so                                0x7191bad000  20480 (20.0 KiB)      /system/lib64/libnetd_client.so
libsoundtrigger.so                               0x718fa81000  73728 (72.0 KiB)      /system/lib64/libsoundtrigger.so
libminikin.so                                    0x7191b48000  139264 (136.0 KiB)    /system/lib64/libminikin.so
libprocessgroup.so                               0x71901b7000  32768 (32.0 KiB)      /system/lib64/libprocessgroup.so
libnativebridge.so                               0x7191c98000  24576 (24.0 KiB)      /system/lib64/libnativebridge.so
libmemunreachable.so                             0x719320a000  188416 (184.0 KiB)    /system/lib64/libmemunreachable.so
libhidlbase.so                                   0x7190470000  61440 (60.0 KiB)      /system/lib64/libhidlbase.so
libhidltransport.so                              0x7193014000  413696 (404.0 KiB)    /system/lib64/libhidltransport.so
libvintf.so                                      0x719335d000  364544 (356.0 KiB)    /system/lib64/libvintf.so
libnativewindow.so                               0x7192725000  24576 (24.0 KiB)      /system/lib64/libnativewindow.so
libhwui.so                                       0x7191841000  1024000 (1000.0 KiB)  /system/lib64/libhwui.so
libbacktrace.so                                  0x7193317000  114688 (112.0 KiB)    /system/lib64/libbacktrace.so
libvndksupport.so                                0x7192676000  12288 (12.0 KiB)      /system/lib64/libvndksupport.so
libaudiomanager.so                               0x71937b2000  40960 (40.0 KiB)      /system/lib64/libaudiomanager.so
libstagefright.so                                0x719275a000  3280896 (3.1 MiB)     /system/lib64/libstagefright.so
libstagefright_foundation.so                     0x7191aa0000  274432 (268.0 KiB)    /system/lib64/libstagefright_foundation.so
libstagefright_http_support.so                   0x719234f000  24576 (24.0 KiB)      /system/lib64/libstagefright_http_support.so
android.hardware.memtrack@1.0.so                 0x71904d4000  110592 (108.0 KiB)    /system/lib64/android.hardware.memtrack@1.0.so
android.hardware.graphics.allocator@2.0.so       0x71920e7000  98304 (96.0 KiB)      /system/lib64/android.hardware.graphics.allocator@2.0.so
android.hardware.graphics.mapper@2.0.so          0x71926d9000  122880 (120.0 KiB)    /system/lib64/android.hardware.graphics.mapper@2.0.so
android.hardware.configstore@1.0.so              0x719315a000  143360 (140.0 KiB)    /system/lib64/android.hardware.configstore@1.0.so
android.hardware.configstore-utils.so            0x7192bc5000  16384 (16.0 KiB)      /system/lib64/android.hardware.configstore-utils.so
libsync.so                                       0x7192288000  16384 (16.0 KiB)      /system/lib64/libsync.so
android.hidl.token@1.0-utils.so                  0x7191a76000  20480 (20.0 KiB)      /system/lib64/android.hidl.token@1.0-utils.so
android.hardware.graphics.bufferqueue@1.0.so     0x718fac0000  253952 (248.0 KiB)    /system/lib64/android.hardware.graphics.bufferqueue@1.0.so
libdng_sdk.so                                    0x719004b000  835584 (816.0 KiB)    /system/lib64/libdng_sdk.so
libexpat.so                                      0x7193740000  163840 (160.0 KiB)    /system/lib64/libexpat.so
libft2.so                                        0x718fb06000  499712 (488.0 KiB)    /system/lib64/libft2.so
libheif.so                                       0x7192fee000  36864 (36.0 KiB)      /system/lib64/libheif.so
libicui18n.so                                    0x7191dc5000  2277376 (2.2 MiB)     /system/lib64/libicui18n.so
libpiex.so                                       0x71917d7000  106496 (104.0 KiB)    /system/lib64/libpiex.so
libpng.so                                        0x7190386000  212992 (208.0 KiB)    /system/lib64/libpng.so
libpcre2.so                                      0x71916d5000  139264 (136.0 KiB)    /system/lib64/libpcre2.so
libpackagelistparser.so                          0x7192391000  16384 (16.0 KiB)      /system/lib64/libpackagelistparser.so
libclang_rt.ubsan_standalone-aarch64-android.so  0x7192c51000  3600384 (3.4 MiB)     /system/lib64/libclang_rt.ubsan_standalone-aarch64-android.so
android.hardware.media.omx@1.0.so                0x7191602000  483328 (472.0 KiB)    /system/lib64/android.hardware.media.omx@1.0.so
android.hardware.media@1.0.so                    0x7191bf0000  32768 (32.0 KiB)      /system/lib64/android.hardware.media@1.0.so
libsonivox.so                                    0x71933c9000  401408 (392.0 KiB)    /system/lib64/libsonivox.so
libaudioutils.so                                 0x7192019000  81920 (80.0 KiB)      /system/lib64/libaudioutils.so
libmedia_helper.so                               0x719174e000  98304 (96.0 KiB)      /system/lib64/libmedia_helper.so
libmediadrm.so                                   0x7192302000  229376 (224.0 KiB)    /system/lib64/libmediadrm.so
libmediametrics.so                               0x7193822000  69632 (68.0 KiB)      /system/lib64/libmediametrics.so
libhidlmemory.so                                 0x7190162000  20480 (20.0 KiB)      /system/lib64/libhidlmemory.so
android.hidl.memory@1.0.so                       0x7190882000  143360 (140.0 KiB)    /system/lib64/android.hidl.memory@1.0.so
android.hardware.graphics.common@1.0.so          0x71923cd000  49152 (48.0 KiB)      /system/lib64/android.hardware.graphics.common@1.0.so
libtinyxml2.so                                   0x7191989000  69632 (68.0 KiB)      /system/lib64/libtinyxml2.so
libprotobuf-cpp-lite.so                          0x719221a000  270336 (264.0 KiB)    /system/lib64/libprotobuf-cpp-lite.so
libRScpp.so                                      0x71936ae000  299008 (292.0 KiB)    /system/lib64/libRScpp.so
libunwind.so                                     0x719212a000  544768 (532.0 KiB)    /system/lib64/libunwind.so
libdrmframework.so                               0x7190b99000  147456 (144.0 KiB)    /system/lib64/libdrmframework.so
libmediautils.so                                 0x71932ca000  65536 (64.0 KiB)      /system/lib64/libmediautils.so
libvorbisidec.so                                 0x719261f000  131072 (128.0 KiB)    /system/lib64/libvorbisidec.so
libstagefright_omx_utils.so                      0x7191b27000  28672 (28.0 KiB)      /system/lib64/libstagefright_omx_utils.so
libstagefright_flacdec.so                        0x71919d0000  114688 (112.0 KiB)    /system/lib64/libstagefright_flacdec.so
libstagefright_xmlparser.so                      0x71920b0000  61440 (60.0 KiB)      /system/lib64/libstagefright_xmlparser.so
android.hidl.allocator@1.0.so                    0x7191cd8000  98304 (96.0 KiB)      /system/lib64/android.hidl.allocator@1.0.so
android.hardware.cas@1.0.so                      0x71914b1000  274432 (268.0 KiB)    /system/lib64/android.hardware.cas@1.0.so
android.hardware.cas.native@1.0.so               0x7193707000  122880 (120.0 KiB)    /system/lib64/android.hardware.cas.native@1.0.so
libpowermanager.so                               0x7191d41000  28672 (28.0 KiB)      /system/lib64/libpowermanager.so
android.hidl.token@1.0.so                        0x71934d8000  102400 (100.0 KiB)    /system/lib64/android.hidl.token@1.0.so
libstdc++.so                                     0x719053b000  20480 (20.0 KiB)      /system/lib64/libstdc++.so
libspeexresampler.so                             0x71935a8000  24576 (24.0 KiB)      /system/lib64/libspeexresampler.so
android.hardware.drm@1.0.so                      0x7193240000  434176 (424.0 KiB)    /system/lib64/android.hardware.drm@1.0.so
liblzma.so                                       0x719024f000  172032 (168.0 KiB)    /system/lib64/liblzma.so
libmedia_omx.so                                  0x7193459000  389120 (380.0 KiB)    /system/lib64/libmedia_omx.so
libart.so                                        0x710efe0000  6352896 (6.1 MiB)     /system/lib64/libart.so
liblz4.so                                        0x710ef51000  86016 (84.0 KiB)      /system/lib64/liblz4.so
libtombstoned_client.so                          0x710efa3000  24576 (24.0 KiB)      /system/lib64/libtombstoned_client.so
libsigchain.so                                   0x710fa42000  12288 (12.0 KiB)      /system/lib64/libsigchain.so
boot.oat                                         0x7075f000    8482816 (8.1 MiB)     /system/framework/arm64/boot.oat
boot-core-libart.oat                             0x70f76000    3497984 (3.3 MiB)     /system/framework/arm64/boot-core-libart.oat
boot-conscrypt.oat                               0x712cc000    487424 (476.0 KiB)    /system/framework/arm64/boot-conscrypt.oat
boot-okhttp.oat                                  0x71343000    622592 (608.0 KiB)    /system/framework/arm64/boot-okhttp.oat
boot-bouncycastle.oat                            0x713db000    487424 (476.0 KiB)    /system/framework/arm64/boot-bouncycastle.oat
boot-apache-xml.oat                              0x71452000    36864 (36.0 KiB)      /system/framework/arm64/boot-apache-xml.oat
boot-legacy-test.oat                             0x7145b000    36864 (36.0 KiB)      /system/framework/arm64/boot-legacy-test.oat
boot-ext.oat                                     0x71464000    278528 (272.0 KiB)    /system/framework/arm64/boot-ext.oat
boot-framework.oat                               0x714a8000    24780800 (23.6 MiB)   /system/framework/arm64/boot-framework.oat
boot-telephony-common.oat                        0x72c4a000    3039232 (2.9 MiB)     /system/framework/arm64/boot-telephony-common.oat
boot-voip-common.oat                             0x72f30000    94208 (92.0 KiB)      /system/framework/arm64/boot-voip-common.oat
boot-ims-common.oat                              0x72f47000    114688 (112.0 KiB)    /system/framework/arm64/boot-ims-common.oat
boot-org.apache.http.legacy.boot.oat             0x72f63000    638976 (624.0 KiB)    /system/framework/arm64/boot-org.apache.http.legacy.boot.oat
boot-android.hidl.base-V1.0-java.oat             0x72fff000    24576 (24.0 KiB)      /system/framework/arm64/boot-android.hidl.base-V1.0-java.oat
boot-android.hidl.manager-V1.0-java.oat          0x73005000    45056 (44.0 KiB)      /system/framework/arm64/boot-android.hidl.manager-V1.0-java.oat
libandroid.so                                    0x7109adc000  122880 (120.0 KiB)    /system/lib64/libandroid.so
libaaudio.so                                     0x7109a83000  196608 (192.0 KiB)    /system/lib64/libaaudio.so
libcamera2ndk.so                                 0x7109985000  126976 (124.0 KiB)    /system/lib64/libcamera2ndk.so
libmediandk.so                                   0x71099d8000  114688 (112.0 KiB)    /system/lib64/libmediandk.so
libmedia_jni.so                                  0x710990c000  438272 (428.0 KiB)    /system/lib64/libmedia_jni.so
libmidi.so                                       0x7109a17000  73728 (72.0 KiB)      /system/lib64/libmidi.so
libmtp.so                                        0x71098c0000  180224 (176.0 KiB)    /system/lib64/libmtp.so
libexif.so                                       0x7109a40000  237568 (232.0 KiB)    /system/lib64/libexif.so
libGLESv3.so                                     0x7109891000  94208 (92.0 KiB)      /system/lib64/libGLESv3.so
libjnigraphics.so                                0x7109851000  12288 (12.0 KiB)      /system/lib64/libjnigraphics.so
libneuralnetworks.so                             0x7109582000  2207744 (2.1 MiB)     /system/lib64/libneuralnetworks.so
libtextclassifier_hash.so                        0x7109548000  20480 (20.0 KiB)      /system/lib64/libtextclassifier_hash.so
android.hardware.neuralnetworks@1.0.so           0x71097f5000  278528 (272.0 KiB)    /system/lib64/android.hardware.neuralnetworks@1.0.so
libOpenMAXAL.so                                  0x7109514000  16384 (16.0 KiB)      /system/lib64/libOpenMAXAL.so
libOpenSLES.so                                   0x71094c0000  20480 (20.0 KiB)      /system/lib64/libOpenSLES.so
libRS.so                                         0x7109428000  77824 (76.0 KiB)      /system/lib64/libRS.so
android.hardware.renderscript@1.0.so             0x7109455000  389120 (380.0 KiB)    /system/lib64/android.hardware.renderscript@1.0.so
libwebviewchromium_plat_support.so               0x71093cf000  20480 (20.0 KiB)      /system/lib64/libwebviewchromium_plat_support.so
libjavacore.so                                   0x7109352000  303104 (296.0 KiB)    /system/lib64/libjavacore.so
libopenjdk.so                                    0x7107c03000  245760 (240.0 KiB)    /system/lib64/libopenjdk.so
libssl.so                                        0x7107c71000  290816 (284.0 KiB)    /system/lib64/libssl.so
libopenjdkjvm.so                                 0x7107ccb000  40960 (40.0 KiB)      /system/lib64/libopenjdkjvm.so
libart-compiler.so                               0x7107921000  2953216 (2.8 MiB)     /system/lib64/libart-compiler.so
libart-dexlayout.so                              0x710768a000  221184 (216.0 KiB)    /system/lib64/libart-dexlayout.so
libvixl-arm.so                                   0x71077d9000  1200128 (1.1 MiB)     /system/lib64/libvixl-arm.so
libvixl-arm64.so                                 0x71076c5000  962560 (940.0 KiB)    /system/lib64/libvixl-arm64.so
libsoundpool.so                                  0x7104d8d000  57344 (56.0 KiB)      /system/lib64/libsoundpool.so
libjavacrypto.so                                 0x7104d43000  245760 (240.0 KiB)    /system/lib64/libjavacrypto.so
android.hardware.graphics.mapper@2.0-impl.so     0x7104cb5000  45056 (44.0 KiB)      /vendor/lib64/hw/android.hardware.graphics.mapper@2.0-impl.so
libEGL_adreno.so                                 0x7104c1d000  114688 (112.0 KiB)    /vendor/lib64/egl/libEGL_adreno.so
libadreno_utils.so                               0x7104c6f000  49152 (48.0 KiB)      /vendor/lib64/libadreno_utils.so
libgsl.so                                        0x7104ad8000  1114112 (1.1 MiB)     /vendor/lib64/libgsl.so
libGLESv2_adreno.so                              0x7104719000  3416064 (3.3 MiB)     /vendor/lib64/egl/libGLESv2_adreno.so
libllvm-glnext.so                                0x7100e88000  13807616 (13.2 MiB)   /vendor/lib64/libllvm-glnext.so
libGLESv1_CM_adreno.so                           0x71046cc000  212992 (208.0 KiB)    /vendor/lib64/egl/libGLESv1_CM_adreno.so
eglSubDriverAndroid.so                           0x71046a8000  69632 (68.0 KiB)      /vendor/lib64/egl/eglSubDriverAndroid.so
libcompiler_rt.so                                0x7100de5000  585728 (572.0 KiB)    /system/lib64/libcompiler_rt.so
libwebviewchromium_loader.so                     0x7104665000  16384 (16.0 KiB)      /system/lib64/libwebviewchromium_loader.so
base.odex                                        0x70f9683000  176128 (172.0 KiB)    /data/app/com.example.jnitest-pLGf0uUkjB1xcA9znOj9MQ==/oat/arm64/base.odex
libjnitest.so                                    0x70f9484000  221184 (216.0 KiB)    /data/app/com.example.jnitest-pLGf0uUkjB1xcA9znOj9MQ==/lib/arm64/libjnitest...
gralloc.msm8992.so                               0x70f905e000  65536 (64.0 KiB)      /vendor/lib64/hw/gralloc.msm8992.so
libmemalloc.so                                   0x70f9089000  32768 (32.0 KiB)      /system/lib64/libmemalloc.so
libqdMetaData.so                                 0x70f944b000  12288 (12.0 KiB)      /system/lib64/libqdMetaData.so
libqdutils.so                                    0x70f9031000  49152 (48.0 KiB)      /system/lib64/libqdutils.so
libqservice.so                                   0x70f90c1000  45056 (44.0 KiB)      /system/lib64/libqservice.so
frida-agent-64.so                                0x70f650d000  22810624 (21.8 MiB)   /data/local/tmp/re.frida.server/frida-agent-64.so
linux-vdso.so.1                                  0x71940bb000  4096 (4.0 KiB)        linux-vdso.so.1
linker64                                         0x71940bd000  999424 (976.0 KiB)    /system/bin/linker64

```

```
libjnitest.so                                    0x70f9484000  221184 (216.0 KiB)    /data/app/com.example.jnitest-pLGf0uUkjB1xcA9znOj9MQ==/lib/arm64/libjnitest...

```



在确认内存中已经加载相应的so文件后可以使用如下命令查看相应模块的所有导出符号：

```
com.example.jnitest on (google: 8.1.0) [usb] # memory list exports libjnitest.so
Save the output by adding `--json exports.json` to this command
Type      Name                                                                                                             Address
--------  ---------------------------------------------------------------------------------------------------------------  ------------
variable  _ZTSN10__cxxabiv117__array_type_infoE                                                                            0x70f94ae608
function  __cxa_call_unexpected                                                                                            0x70f949533c
function  _ZNSt12length_errorD1Ev                                                                                          0x70f9495b80
variable  _ZTVN10__cxxabiv123__fundamental_type_infoE                                                                      0x70f94b7f40
variable  _ZTIPDn                                                                                                          0x70f94b7ff0
function  __cxa_free_exception                                                                                             0x70f949425c
variable  _ZTIN10__cxxabiv117__array_type_infoE                                                                            0x70f94b8798
function  _ZN10__cxxabiv121__isOurExceptionClassEPK17_Unwind_Exception                                                     0x70f94941f0
function  _ZNSt11logic_errorC2EPKc                                                                                         0x70f9493f28
variable  _ZTIPDs                                                                                                          0x70f94b86d0
variable  _ZTVN10__cxxabiv129__pointer_to_member_type_infoE                                                                0x70f94b89c8
variable  _ZTIPDu                                                                                                          0x70f94b8680
function  _ZnamSt11align_val_tRKSt9nothrow_t                                                                               0x70f9493e50
function  _ZNSt10bad_typeidD2Ev                                                                                            0x70f94a8a38
variable  __cxa_new_handler                                                                                                0x70f94b9038
function  _ZNSt13bad_exceptionD1Ev                                                                                         0x70f9495b14
variable  _ZTVSt15underflow_error                                                                                          0x70f94b6068
function  _ZdlPvm                                                                                                          0x70f9493d64
variable  _ZTVSt11range_error                                                                                              0x70f94b5fd0
function  _ZNSt6__ndk117_DeallocateCaller27__do_deallocate_handle_sizeEPvm                                                 0x70f9493330
function  _ZdaPvSt11align_val_t                                                                                            0x70f9493e8c
function  _ZNSt11logic_errorC1ERKS_                                                                                        0x70f9493fa8
variable  _ZTSN10__cxxabiv123__fundamental_type_infoE                                                                      0x70f94ae4f0
function  _ZNSt13runtime_errorC1EPKc                                                                                       0x70f94940c4
function  __gxx_personality_v0                                                                                             0x70f9494b54
function  __cxa_throw                                                                                                      0x70f94942b0
variable  _ZTISt12length_error                                                                                             0x70f94b5f78
function  _ZNSt9bad_allocC2Ev                                                                                              0x70f9495b38
function  _ZNSt12domain_errorD0Ev                                                                                          0x70f9495cf0
variable  _ZTSSt16invalid_argument                                                                                         0x70f94ac58c
variable  _ZTSN10__cxxabiv116__enum_type_infoE                                                                             0x70f94ae62a
function  _ZNSt6__ndk111char_traitsIcE6assignERcRKc                                                                        0x70f94938c8
variable  _ZTSSt12out_of_range                                                                                             0x70f94ac5b2
function  _ZdlPvmSt11align_val_t                                                                                           0x70f9493e84
function  _ZNSt12length_errorD2Ev                                                                                          0x70f9495b80
variable  _ZTISt8bad_cast                                                                                                  0x70f94b8a80
variable  _ZTVN10__cxxabiv121__vmi_class_type_infoE                                                                        0x70f94b88f0
variable  _ZTVSt9bad_alloc                                                                                                 0x70f94b5d70
function  _ZSt14set_unexpectedPFvvE                                                                                        0x70f9496064
function  _ZNKSt13bad_exception4whatEv                                                                                     0x70f9495b2c
function  _ZdlPv                                                                                                           0x70f9493d5c
function  _ZNSt13runtime_errorC2ERKS_                                                                                      0x70f9494144
variable  _ZTISt16invalid_argument                                                                                         0x70f94b5f38
function  __cxa_uncaught_exceptions                                                                                        0x70f9494904
function  _ZNSt11logic_erroraSERKS_                                                                                        0x70f9493fd8
function  _ZNSt13runtime_errorC1ERKNSt6__ndk112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE                     0x70f9494038
function  _ZdlPvSt11align_val_tRKSt9nothrow_t                                                                              0x70f9493e80
function  __cxa_current_exception_type                                                                                     0x70f9494590
function  _ZNSt13bad_exceptionD2Ev                                                                                         0x70f9495b14
function  _ZN7_JNIEnv12NewStringUTFEPKc                                                                                    0x70f949307c
variable  _ZTIa                                                                                                            0x70f94b8170
function  _ZSt10unexpectedv                                                                                                0x70f9494ac0
variable  _ZTIb                                                                                                            0x70f94b8030
variable  _ZTSN10__cxxabiv121__vmi_class_type_infoE                                                                        0x70f94ae670
function  _ZNKSt6__ndk121__basic_string_commonILb1EE20__throw_length_errorEv                                               0x70f9493608
variable  _ZTIc                                                                                                            0x70f94b80d0
variable  _ZTId                                                                                                            0x70f94b8580
variable  _ZTIe                                                                                                            0x70f94b85d0
variable  _ZTIf                                                                                                            0x70f94b8530
variable  _ZTIg                                                                                                            0x70f94b8620
variable  _ZTSN10__cxxabiv129__pointer_to_member_type_infoE                                                                0x70f94ae4aa
variable  _ZTIh                                                                                                            0x70f94b8120
variable  _ZTIi                                                                                                            0x70f94b8260
function  __cxa_rethrow_primary_exception                                                                                  0x70f9494750
function  _ZNSt9type_infoD0Ev                                                                                              0x70f94a89d8
variable  _ZTIj                                                                                                            0x70f94b82b0
variable  _ZTSSt15underflow_error                                                                                          0x70f94ac5f8
variable  _ZTIl                                                                                                            0x70f94b8300
variable  _ZTVN10__cxxabiv120__si_class_type_infoE                                                                         0x70f94b8888
variable  _ZTIm                                                                                                            0x70f94b8350
function  _ZNSt12domain_errorD1Ev                                                                                          0x70f9495b80
function  _ZnwmRKSt9nothrow_t                                                                                              0x70f9493d00
variable  _ZTIn                                                                                                            0x70f94b8440
function  _ZNSt6__ndk112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE10__align_itILm16EEEmm                         0x70f9493a98
variable  _ZTVSt8bad_cast                                                                                                  0x70f94b8a00
variable  _ZTISt10bad_typeid                                                                                               0x70f94b8a98
function  _ZNSt8bad_castD0Ev                                                                                               0x70f94a89f4
variable  _ZTIo                                                                                                            0x70f94b8490
function  _ZNSt9exceptionD0Ev                                                                                              0x70f9495b18
function  _ZNSt10bad_typeidC1Ev                                                                                            0x70f94a8a24
variable  _ZTSPKa                                                                                                          0x70f94ae556
variable  _ZTSSt10bad_typeid                                                                                               0x70f94ae6cd
variable  _ZTSPKb                                                                                                          0x70f94ae532
variable  _ZTIs                                                                                                            0x70f94b81c0
variable  _ZTSPKc                                                                                                          0x70f94ae544
variable  _ZTIt                                                                                                            0x70f94b8210
function  _ZSt13set_terminatePFvvE                                                                                         0x70f949608c
variable  _ZTSPKd                                                                                                          0x70f94ae5ce
function  _ZNSt6__ndk112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEEC2IDnEEPKc                                     0x70f9493024
variable  _ZTSPKe                                                                                                          0x70f94ae5d7
function  _ZSt17__throw_bad_allocv                                                                                         0x70f9493c68
variable  _ZTIv                                                                                                            0x70f94b7f90
function  _ZNSt13runtime_errorC2EPKc                                                                                       0x70f94940c4
variable  _ZTSPKf                                                                                                          0x70f94ae5c5
variable  _ZTSPKg                                                                                                          0x70f94ae5e0
variable  _ZTIw                                                                                                            0x70f94b8080
variable  _ZTSPa                                                                                                           0x70f94ae553
variable  _ZTSSt20bad_array_new_length                                                                                     0x70f94ac552
function  _ZnamRKSt9nothrow_t                                                                                              0x70f9493d30
variable  _ZTVSt16invalid_argument                                                                                         0x70f94b5f10
variable  _ZTSPKh                                                                                                          0x70f94ae54d
variable  _ZTVSt12length_error                                                                                             0x70f94b5f50
variable  _ZTSPb                                                                                                           0x70f94ae52f
variable  _ZTIx                                                                                                            0x70f94b83a0
variable  _ZTIy                                                                                                            0x70f94b83f0
variable  _ZTSPc                                                                                                           0x70f94ae541
variable  _ZTSPKi                                                                                                          0x70f94ae571
variable  _ZTSPKj                                                                                                          0x70f94ae57a
variable  _ZTSPd                                                                                                           0x70f94ae5cb
variable  _ZTIN10__cxxabiv121__vmi_class_type_infoE                                                                        0x70f94b8940
variable  _ZTSPe                                                                                                           0x70f94ae5d4
variable  _ZTIN10__cxxabiv123__fundamental_type_infoE                                                                      0x70f94b7f78
function  _ZNKSt10bad_typeid4whatEv                                                                                        0x70f94a8a60
variable  _ZTSPKl                                                                                                          0x70f94ae583
variable  _ZTSPf                                                                                                           0x70f94ae5c2
function  _ZNKSt8bad_cast4whatEv                                                                                           0x70f94a8a18
function  _ZNSt20bad_array_new_lengthD0Ev                                                                                  0x70f9495b70
variable  _ZTSPKm                                                                                                          0x70f94ae58c
variable  _ZTSPg                                                                                                           0x70f94ae5dd
variable  _ZTVN10__cxxabiv117__class_type_infoE                                                                            0x70f94b8838
function  _ZNSt14overflow_errorD0Ev                                                                                        0x70f9495ebc
function  _ZSt9terminatev                                                                                                  0x70f9494a68
variable  _ZTSPKn                                                                                                          0x70f94ae5a7
variable  _ZTSPh                                                                                                           0x70f94ae54a
variable  _ZTSN10__cxxabiv117__pbase_type_infoE                                                                            0x70f94ae43f
variable  _ZTISt13bad_exception                                                                                            0x70f94b5e20
variable  _ZTSPi                                                                                                           0x70f94ae56e
variable  _ZTSSt9type_info                                                                                                 0x70f94ae6b4
variable  _ZTSPKo                                                                                                          0x70f94ae5b0
variable  _ZTSPj                                                                                                           0x70f94ae577
variable  _ZTVN10__cxxabiv120__function_type_infoE                                                                         0x70f94b87b0
variable  _ZTSPl                                                                                                           0x70f94ae580
variable  _ZTIN10__cxxabiv117__pbase_type_infoE                                                                            0x70f94b7ea8
variable  _ZTSPm                                                                                                           0x70f94ae589
function  _ZnwmSt11align_val_t                                                                                             0x70f9493d74
variable  _ZTIN10__cxxabiv116__shim_type_infoE                                                                             0x70f94b7e78
variable  _ZTSPKs                                                                                                          0x70f94ae55f
variable  _ZTSPn                                                                                                           0x70f94ae5a4
variable  _ZTSPKt                                                                                                          0x70f94ae568
variable  _ZTSPo                                                                                                           0x70f94ae5ad
function  _ZSt14get_unexpectedv                                                                                            0x70f9494a58
variable  _ZTSPKv                                                                                                          0x70f94ae51d
variable  _ZTSSt9exception                                                                                                 0x70f94ac526
variable  _ZTSPKw                                                                                                          0x70f94ae53b
variable  _ZTSPKx                                                                                                          0x70f94ae595
variable  _ZTSPs                                                                                                           0x70f94ae55c
function  _ZNKSt20bad_array_new_length4whatEv                                                                              0x70f9495b74
variable  _ZTSPKy                                                                                                          0x70f94ae59e
variable  _ZTSPt                                                                                                           0x70f94ae565
function  __cxa_increment_exception_refcount                                                                               0x70f94946c0
variable  _ZTSPv                                                                                                           0x70f94ae51a
variable  _ZTSPw                                                                                                           0x70f94ae538
function  _ZNSt6__ndk111char_traitsIcE4copyEPcPKcm                                                                         0x70f94937ec
variable  _ZTSPx                                                                                                           0x70f94ae592
function  _ZNSt9type_infoD1Ev                                                                                              0x70f94a89d4
variable  _ZTSPy                                                                                                           0x70f94ae59b
function  _ZNSt6__ndk112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE6__initEPKcm                                   0x70f9493420
function  _ZNSt12domain_errorD2Ev                                                                                          0x70f9495b80
variable  _ZTSSt13bad_exception                                                                                            0x70f94ac533
function  _ZNSt8bad_castD1Ev                                                                                               0x70f94a89f0
function  _ZNSt13runtime_errorC1ERKS_                                                                                      0x70f9494144
function  _ZNSt9exceptionD1Ev                                                                                              0x70f9495b14
variable  _ZTSSt12length_error                                                                                             0x70f94ac5a1
function  _ZNSt10bad_typeidC2Ev                                                                                            0x70f94a8a24
variable  _ZTIDh                                                                                                           0x70f94b84e0
variable  _ZTIDi                                                                                                           0x70f94b8710
variable  _ZTSPDh                                                                                                          0x70f94ae5b7
variable  _ZTSPDi                                                                                                          0x70f94ae5ff
function  __cxa_uncaught_exception                                                                                         0x70f94948dc
function  _ZNSt13runtime_errorD0Ev                                                                                         0x70f9495c8c
variable  _ZTVN10__cxxabiv116__shim_type_infoE                                                                             0x70f94b7f08
variable  _ZTIDn                                                                                                           0x70f94b7fe0
variable  _ZTSPDn                                                                                                          0x70f94ae524
function  _ZdaPvmSt11align_val_t                                                                                           0x70f9493e94
function  _ZNSt6__ndk117_DeallocateCaller9__do_callEPv                                                                     0x70f9493358
variable  _ZTIDs                                                                                                           0x70f94b86c0
function  _ZNSt20bad_array_new_lengthD1Ev                                                                                  0x70f9495b14
variable  _ZTVSt10bad_typeid                                                                                               0x70f94b8a28
function  _ZdaPvRKSt9nothrow_t                                                                                             0x70f9493d6c
function  _ZNSt14overflow_errorD1Ev                                                                                        0x70f9495c38
variable  _ZTIDu                                                                                                           0x70f94b8670
variable  _ZTSPDs                                                                                                          0x70f94ae5f3
variable  _ZTSa                                                                                                            0x70f94ae551
variable  _ZTIN10__cxxabiv119__pointer_type_infoE                                                                          0x70f94b7ec0
function  _ZNSt15underflow_errorD0Ev                                                                                       0x70f9495f18
variable  _ZTSPDu                                                                                                          0x70f94ae5e7
variable  _ZTVSt14overflow_error                                                                                           0x70f94b6028
variable  _ZTSb                                                                                                            0x70f94ae52d
variable  _ZTSc                                                                                                            0x70f94ae53f
variable  _ZTSd                                                                                                            0x70f94ae5c9
function  _ZNSt9bad_allocD0Ev                                                                                              0x70f9495b4c
variable  _ZTSe                                                                                                            0x70f94ae5d2
variable  _ZTSf                                                                                                            0x70f94ae5c0
variable  _ZTSg                                                                                                            0x70f94ae5db
variable  _ZTSSt13runtime_error                                                                                            0x70f94ac5d3
variable  _ZTSh                                                                                                            0x70f94ae548
variable  _ZTSi                                                                                                            0x70f94ae56c
variable  _ZTSN10__cxxabiv119__pointer_type_infoE                                                                          0x70f94ae461
variable  _ZTSj                                                                                                            0x70f94ae575
function  __cxa_rethrow                                                                                                    0x70f94945dc
variable  _ZTISt12domain_error                                                                                             0x70f94b5ef8
variable  _ZTSl                                                                                                            0x70f94ae57e
variable  _ZTSm                                                                                                            0x70f94ae587
variable  _ZTISt9type_info                                                                                                 0x70f94b8a70
function  __cxa_end_catch                                                                                                  0x70f9494464
variable  _ZTIN10__cxxabiv120__si_class_type_infoE                                                                         0x70f94b88d8
variable  _ZTSn                                                                                                            0x70f94ae5a2
function  _ZNSt6__ndk111char_traitsIcE6lengthEPKc                                                                          0x70f9493554
variable  _ZTSo                                                                                                            0x70f94ae5ab
function  _ZNSt13runtime_erroraSERKS_                                                                                      0x70f9494174
function  _ZSt13get_terminatev                                                                                             0x70f9494ad8
function  _ZNSt9type_infoD2Ev                                                                                              0x70f94a89d4
variable  __cxa_unexpected_handler                                                                                         0x70f94b9010
variable  _ZTSSt9bad_alloc                                                                                                 0x70f94ac545
function  _ZSt15set_new_handlerPFvvE                                                                                       0x70f9494b28
variable  _ZTSs                                                                                                            0x70f94ae55a
variable  _ZTSt                                                                                                            0x70f94ae563
function  _ZNSt11logic_errorD0Ev                                                                                           0x70f9495bd4
variable  _ZTISt9exception                                                                                                 0x70f94b5de8
variable  _ZTISt13runtime_error                                                                                            0x70f94b5ff8
function  _ZNSt8bad_castD2Ev                                                                                               0x70f94a89f0
variable  _ZTSv                                                                                                            0x70f94ae518
function  _ZNSt9exceptionD2Ev                                                                                              0x70f9495b14
function  _ZNSt11logic_errorC2ERKNSt6__ndk112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE                       0x70f9493e9c
variable  _ZTSw                                                                                                            0x70f94ae536
variable  _ZTVN10__cxxabiv117__array_type_infoE                                                                            0x70f94b8760
function  _ZNSt11range_errorD0Ev                                                                                           0x70f9495e60
function  __dynamic_cast                                                                                                   0x70f94a7c9c
function  _ZNSt12out_of_rangeD0Ev                                                                                          0x70f9495e04
variable  _ZTSx                                                                                                            0x70f94ae590
variable  _ZTSy                                                                                                            0x70f94ae599
variable  _ZTSSt8bad_cast                                                                                                  0x70f94ae6c1
function  _ZNSt13runtime_errorD1Ev                                                                                         0x70f9495c38
variable  __cxa_terminate_handler                                                                                          0x70f94b9008
variable  _ZTIN10__cxxabiv120__function_type_infoE                                                                         0x70f94b7ed8
function  Java_com_example_jnitest_MainActivity_stringFromJNI                                                              0x70f9492f6c
function  _ZNSt20bad_array_new_lengthD2Ev                                                                                  0x70f9495b14
function  _ZNSt14overflow_errorD2Ev                                                                                        0x70f9495c38
function  _ZNSt16invalid_argumentD0Ev                                                                                      0x70f9495d4c
function  __cxa_allocate_exception                                                                                         0x70f9494210
function  _ZNSt15underflow_errorD1Ev                                                                                       0x70f9495c38
variable  _ZTSN10__cxxabiv120__si_class_type_infoE                                                                         0x70f94ae64b
function  _ZNSt9bad_allocD1Ev                                                                                              0x70f9495b14
variable  _ZTIN10__cxxabiv116__enum_type_infoE                                                                             0x70f94b8820
function  _ZdaPv                                                                                                           0x70f9493d68
function  _ZNKSt11logic_error4whatEv                                                                                       0x70f9495c30
variable  _ZTSN10__cxxabiv117__class_type_infoE                                                                            0x70f94ae41d
function  _ZN10__cxxabiv119__getExceptionClassEPK17_Unwind_Exception                                                       0x70f94941e8
function  __cxa_get_globals                                                                                                0x70f9494924
function  _ZNSt6__ndk117__compressed_pairINS_12basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE5__repES5_EC2ILb1EvEEv  0x70f94933ec
function  __cxa_get_exception_ptr                                                                                          0x70f94943c8
variable  _ZTISt14overflow_error                                                                                           0x70f94b6050
function  __cxa_pure_virtual                                                                                               0x70f94a8a6c
function  _ZNSt11logic_errorD1Ev                                                                                           0x70f9495b80
variable  _ZTIPKDh                                                                                                         0x70f94b8510
variable  _ZTIPKDi                                                                                                         0x70f94b8740
function  _Znwm                                                                                                            0x70f9493c9c
variable  _ZTSN10__cxxabiv120__function_type_infoE                                                                         0x70f94ae485
function  _ZNSt11range_errorD1Ev                                                                                           0x70f9495c38
function  _ZNSt12out_of_rangeD1Ev                                                                                          0x70f9495b80
variable  _ZTSSt11logic_error                                                                                              0x70f94ac57c
variable  _ZTVSt12domain_error                                                                                             0x70f94b5eb8
variable  _ZTIPKDn                                                                                                         0x70f94b8010
variable  _ZTVN10__cxxabiv119__pointer_type_infoE                                                                          0x70f94b8990
function  _ZNSt13runtime_errorD2Ev                                                                                         0x70f9495c38
variable  _ZTISt15underflow_error                                                                                          0x70f94b6090
function  __cxa_decrement_exception_refcount                                                                               0x70f9494540
variable  _ZTVN10__cxxabiv116__enum_type_infoE                                                                             0x70f94b87e8
variable  _ZTISt9bad_alloc                                                                                                 0x70f94b5e38
function  __cxa_allocate_dependent_exception                                                                               0x70f9494278
variable  _ZTIPKDs                                                                                                         0x70f94b86f0
function  _ZnamSt11align_val_t                                                                                             0x70f9493e4c
variable  _ZTIPKDu                                                                                                         0x70f94b86a0
variable  _ZTISt20bad_array_new_length                                                                                     0x70f94b5e50
function  __cxa_demangle                                                                                                   0x70f94961f0
variable  _ZTSPKDh                                                                                                         0x70f94ae5bb
variable  _ZTSPKDi                                                                                                         0x70f94ae603
function  _ZNKSt9exception4whatEv                                                                                          0x70f9495b1c
function  _ZNSt16invalid_argumentD1Ev                                                                                      0x70f9495b80
function  _ZNSt15underflow_errorD2Ev                                                                                       0x70f9495c38
function  _ZSt15get_new_handlerv                                                                                           0x70f9494b44
function  _ZNSt8bad_castC1Ev                                                                                               0x70f94a89dc
function  _ZNSt9bad_allocD2Ev                                                                                              0x70f9495b14
variable  _ZTSPKDn                                                                                                         0x70f94ae528
function  _ZdlPvRKSt9nothrow_t                                                                                             0x70f9493d60
function  __cxa_deleted_virtual                                                                                            0x70f94a8a80
variable  _ZTIN10__cxxabiv129__pointer_to_member_type_infoE                                                                0x70f94b7ef0
function  _ZNSt10bad_typeidD0Ev                                                                                            0x70f94a8a3c
variable  _ZTSSt11range_error                                                                                              0x70f94ac5c3
variable  _ZTSPKDs                                                                                                         0x70f94ae5f7
variable  _ZTVSt13bad_exception                                                                                            0x70f94b5df8
variable  _ZTSPKDu                                                                                                         0x70f94ae5eb
variable  _ZTIN10__cxxabiv117__class_type_infoE                                                                            0x70f94b7e90
function  _ZNKSt9bad_alloc4whatEv                                                                                          0x70f9495b50
function  _ZNSt20bad_array_new_lengthC1Ev                                                                                  0x70f9495b5c
variable  _ZTSDh                                                                                                           0x70f94ae5b4
function  _ZNSt11logic_errorD2Ev                                                                                           0x70f9495b80
variable  _ZTSDi                                                                                                           0x70f94ae5fc
function  _ZNSt11logic_errorC1EPKc                                                                                         0x70f9493f28
function  _ZNSt11range_errorD2Ev                                                                                           0x70f9495c38
function  _ZNSt12out_of_rangeD2Ev                                                                                          0x70f9495b80
variable  _ZTSDn                                                                                                           0x70f94ae521
function  __cxa_free_dependent_exception                                                                                   0x70f94942ac
variable  _ZTSN10__cxxabiv116__shim_type_infoE                                                                             0x70f94ae3fc
function  _ZNSt11logic_errorC2ERKS_                                                                                        0x70f9493fa8
variable  _ZTIPKa                                                                                                          0x70f94b81a0
variable  _ZTIPKb                                                                                                          0x70f94b8060
variable  _ZTISt12out_of_range                                                                                             0x70f94b5fb8
variable  _ZTIPKc                                                                                                          0x70f94b8100
variable  _ZTIPKd                                                                                                          0x70f94b85b0
function  _ZdaPvSt11align_val_tRKSt9nothrow_t                                                                              0x70f9493e90
variable  _ZTSDs                                                                                                           0x70f94ae5f0
variable  _ZTIPKe                                                                                                          0x70f94b8600
variable  _ZTIPKf                                                                                                          0x70f94b8560
variable  _ZTSDu                                                                                                           0x70f94ae5e4
variable  _ZTVN10__cxxabiv117__pbase_type_infoE                                                                            0x70f94b8958
function  _ZnwmSt11align_val_tRKSt9nothrow_t                                                                               0x70f9493e20
function  _ZNKSt13runtime_error4whatEv                                                                                     0x70f9495ce8
variable  _ZTIPKg                                                                                                          0x70f94b8650
variable  _ZSt7nothrow                                                                                                     0x70f94ac293
variable  _ZTIPKh                                                                                                          0x70f94b8150
function  __cxa_current_primary_exception                                                                                  0x70f94946dc
function  _ZNSt12length_errorD0Ev                                                                                          0x70f9495da8
variable  _ZTISt11logic_error                                                                                              0x70f94b5ee0
variable  _ZTIPKi                                                                                                          0x70f94b8290
variable  _ZTVSt20bad_array_new_length                                                                                     0x70f94b5d98
function  _ZNSt13runtime_errorC2ERKNSt6__ndk112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE                     0x70f9494038
variable  _ZTIPKj                                                                                                          0x70f94b82e0
function  _ZNSt16invalid_argumentD2Ev                                                                                      0x70f9495b80
variable  _ZTIPa                                                                                                           0x70f94b8180
variable  _ZTIPKl                                                                                                          0x70f94b8330
variable  _ZTIPb                                                                                                           0x70f94b8040
variable  _ZTIPKm                                                                                                          0x70f94b8380
variable  _ZTSSt14overflow_error                                                                                           0x70f94ac5e5
variable  _ZTIPKn                                                                                                          0x70f94b8470
variable  _ZTIPc                                                                                                           0x70f94b80e0
variable  _ZTIPKo                                                                                                          0x70f94b84c0
variable  _ZTIPd                                                                                                           0x70f94b8590
function  _ZNSt8bad_castC2Ev                                                                                               0x70f94a89dc
variable  _ZTIPe                                                                                                           0x70f94b85e0
variable  _ZTIPf                                                                                                           0x70f94b8540
function  _ZNSt10bad_typeidD1Ev                                                                                            0x70f94a8a38
variable  _ZTIPg                                                                                                           0x70f94b8630
variable  _ZTIPKs                                                                                                          0x70f94b81f0
variable  _ZTVSt12out_of_range                                                                                             0x70f94b5f90
variable  _ZTIPh                                                                                                           0x70f94b8130
variable  _ZTIPKt                                                                                                          0x70f94b8240
function  _ZNSt13bad_exceptionD0Ev                                                                                         0x70f9495b28
variable  _ZTIPi                                                                                                           0x70f94b8270
variable  _ZTIPj                                                                                                           0x70f94b82c0
variable  _ZTIPKv                                                                                                          0x70f94b7fc0
function  _Znam                                                                                                            0x70f9493d2c
variable  _ZTIPKw                                                                                                          0x70f94b80b0
variable  _ZTIPl                                                                                                           0x70f94b8310
variable  _ZTIPKx                                                                                                          0x70f94b83d0
function  _ZdlPvSt11align_val_t                                                                                            0x70f9493e7c
variable  _ZTIPm                                                                                                           0x70f94b8360
variable  _ZTIPKy                                                                                                          0x70f94b8420
variable  _ZTSSt12domain_error                                                                                             0x70f94ac56b
variable  _ZTIPn                                                                                                           0x70f94b8450
function  _ZNSt6__ndk112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEED2Ev                                           0x70f94930d8
variable  _ZTIPo                                                                                                           0x70f94b84a0
function  _ZNSt20bad_array_new_lengthC2Ev                                                                                  0x70f9495b5c
variable  _ZTIPs                                                                                                           0x70f94b81d0
variable  _ZTISt11range_error                                                                                              0x70f94b6010
variable  _ZTVSt9type_info                                                                                                 0x70f94b8a50
variable  _ZTIPt                                                                                                           0x70f94b8220
function  __cxa_begin_catch                                                                                                0x70f94943d0
variable  _ZTIPv                                                                                                           0x70f94b7fa0
variable  _ZTIPw                                                                                                           0x70f94b8090
function  __cxa_get_globals_fast                                                                                           0x70f94949b4
function  _ZNSt9bad_allocC1Ev                                                                                              0x70f9495b38
variable  _ZTIPx                                                                                                           0x70f94b83b0
variable  _ZTIPy                                                                                                           0x70f94b8400
variable  _ZTVSt11logic_error                                                                                              0x70f94b5e68
function  _ZN10__cxxabiv119__setExceptionClassEP17_Unwind_Exceptionm                                                       0x70f94941d4
variable  _ZTVSt9exception                                                                                                 0x70f94b5dc0
function  _ZNSt11logic_errorC1ERKNSt6__ndk112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE                       0x70f9493e9c
function  _ZdaPvm                                                                                                          0x70f9493d70
variable  _ZTIPDh                                                                                                          0x70f94b84f0
variable  _ZTIPDi                                                                                                          0x70f94b8720
variable  _ZTVSt13runtime_error                                                                                            0x70f94b5e90

```

可以发现在 源码中的Java_PackageName_ClassName_MethodName格式的 Java_com_roysue_r0so_MainActivity_stringFromJNI()函数名并 未发生任何改变，且相应函数地址为0x70f9492f6c。需要注意的 是，在Android中每次加载模块时基地址都是变化的，最终会导致每 次加载后的函数地址也是不同的，因此这里得到的函数绝对地址是不 可靠的，真正可靠的是函数地址相对于模块基地址的偏移0x70f9492f6c-0x70f9484000=0xef6c.只有这个偏移值每次重新运行都是不变的，我们可以通过这 个偏移值在静态分析时找到相应的函数。

```
function  Java_com_example_jnitest_MainActivity_stringFromJNI                                                              0x70f9492f6c
```

对demo进行修改：

```java
    public native String stringFromJNI2();

```

```c
JNIEXPORT jstring JNICALL
Java_com_example_jnitest_MainActivity_stringFromJNI2(JNIEnv *env, jobject thiz) {
        std::string hello = "Hello from C++ stringFrom JNI2";
        return env->NewStringUTF(hello.c_str());
}
```

```
(base) ┌──(root㉿kali)-[/home/zhy/桌面/native]
└─# objection -g JniTest explore
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
com.example.jnitest on (google: 8.1.0) [usb] # memory list modules
Save the output by adding `--json modules.json` to this command
Name                                             Base          Size                  Path
-----------------------------------------------  ------------  --------------------  ------------------------------------------------------------------------------
app_process64                                    0x559317b000  32768 (32.0 KiB)      /system/bin/app_process64
libandroid_runtime.so                            0x719240d000  1982464 (1.9 MiB)     /system/lib64/libandroid_runtime.so
libbinder.so                                     0x7193096000  561152 (548.0 KiB)    /system/lib64/libbinder.so
libcutils.so                                     0x7191705000  77824 (76.0 KiB)      /system/lib64/libcutils.so
libhwbinder.so                                   0x7191582000  163840 (160.0 KiB)    /system/lib64/libhwbinder.so
liblog.so                                        0x7192c0b000  102400 (100.0 KiB)    /system/lib64/liblog.so
libnativeloader.so                               0x7190875000  28672 (28.0 KiB)      /system/lib64/libnativeloader.so
libutils.so                                      0x71901db000  131072 (128.0 KiB)    /system/lib64/libutils.so
libwilhelm.so                                    0x7193199000  274432 (268.0 KiB)    /system/lib64/libwilhelm.so
libc++.so                                        0x71938c2000  978944 (956.0 KiB)    /system/lib64/libc++.so
libc.so                                          0x7190287000  880640 (860.0 KiB)    /system/lib64/libc.so
libm.so                                          0x7191c05000  233472 (228.0 KiB)    /system/lib64/libm.so
libdl.so                                         0x7193859000  20480 (20.0 KiB)      /system/lib64/libdl.so
libmemtrack.so                                   0x71937c6000  20480 (20.0 KiB)      /system/lib64/libmemtrack.so
libandroidfw.so                                  0x71903e5000  339968 (332.0 KiB)    /system/lib64/libandroidfw.so
libappfuse.so                                    0x719204d000  57344 (56.0 KiB)      /system/lib64/libappfuse.so
libbase.so                                       0x7190be0000  73728 (72.0 KiB)      /system/lib64/libbase.so
libcrypto.so                                     0x7192aa7000  1150976 (1.1 MiB)     /system/lib64/libcrypto.so
libnativehelper.so                               0x71917b1000  36864 (36.0 KiB)      /system/lib64/libnativehelper.so
libdebuggerd_client.so                           0x7191a18000  28672 (28.0 KiB)      /system/lib64/libdebuggerd_client.so
libui.so                                         0x7191d81000  143360 (140.0 KiB)    /system/lib64/libui.so
libgraphicsenv.so                                0x71904b2000  16384 (16.0 KiB)      /system/lib64/libgraphicsenv.so
libgui.so                                        0x71935c7000  618496 (604.0 KiB)    /system/lib64/libgui.so
libsensor.so                                     0x7191681000  94208 (92.0 KiB)      /system/lib64/libsensor.so
libinput.so                                      0x7190984000  184320 (180.0 KiB)    /system/lib64/libinput.so
libcamera_client.so                              0x7191500000  327680 (320.0 KiB)    /system/lib64/libcamera_client.so
libcamera_metadata.so                            0x71908e2000  45056 (44.0 KiB)      /system/lib64/libcamera_metadata.so
libskia.so                                       0x7190c0b000  8257536 (7.9 MiB)     /system/lib64/libskia.so
libsqlite.so                                     0x71905e6000  1138688 (1.1 MiB)     /system/lib64/libsqlite.so
libEGL.so                                        0x7191d01000  192512 (188.0 KiB)    /system/lib64/libEGL.so
libGLESv1_CM.so                                  0x71921c8000  45056 (44.0 KiB)      /system/lib64/libGLESv1_CM.so
libGLESv2.so                                     0x7191800000  94208 (92.0 KiB)      /system/lib64/libGLESv2.so
libvulkan.so                                     0x7190586000  126976 (124.0 KiB)    /system/lib64/libvulkan.so
libziparchive.so                                 0x7192698000  53248 (52.0 KiB)      /system/lib64/libziparchive.so
libETC1.so                                       0x7193885000  20480 (20.0 KiB)      /system/lib64/libETC1.so
libhardware.so                                   0x71915df000  16384 (16.0 KiB)      /system/lib64/libhardware.so
libhardware_legacy.so                            0x7191956000  16384 (16.0 KiB)      /system/lib64/libhardware_legacy.so
libselinux.so                                    0x7191c46000  102400 (100.0 KiB)    /system/lib64/libselinux.so
libicuuc.so                                      0x71909ca000  1642496 (1.6 MiB)     /system/lib64/libicuuc.so
libmedia.so                                      0x7190703000  1097728 (1.0 MiB)     /system/lib64/libmedia.so
libaudioclient.so                                0x7191416000  434176 (424.0 KiB)    /system/lib64/libaudioclient.so
libjpeg.so                                       0x7193503000  294912 (288.0 KiB)    /system/lib64/libjpeg.so
libusbhost.so                                    0x7190554000  24576 (24.0 KiB)      /system/lib64/libusbhost.so
libharfbuzz_ng.so                                0x7190902000  503808 (492.0 KiB)    /system/lib64/libharfbuzz_ng.so
libz.so                                          0x7190222000  94208 (92.0 KiB)      /system/lib64/libz.so
libpdfium.so                                     0x718fb84000  4960256 (4.7 MiB)     /system/lib64/libpdfium.so
libimg_utils.so                                  0x71922d2000  86016 (84.0 KiB)      /system/lib64/libimg_utils.so
libnetd_client.so                                0x7191bad000  20480 (20.0 KiB)      /system/lib64/libnetd_client.so
libsoundtrigger.so                               0x718fa81000  73728 (72.0 KiB)      /system/lib64/libsoundtrigger.so
libminikin.so                                    0x7191b48000  139264 (136.0 KiB)    /system/lib64/libminikin.so
libprocessgroup.so                               0x71901b7000  32768 (32.0 KiB)      /system/lib64/libprocessgroup.so
libnativebridge.so                               0x7191c98000  24576 (24.0 KiB)      /system/lib64/libnativebridge.so
libmemunreachable.so                             0x719320a000  188416 (184.0 KiB)    /system/lib64/libmemunreachable.so
libhidlbase.so                                   0x7190470000  61440 (60.0 KiB)      /system/lib64/libhidlbase.so
libhidltransport.so                              0x7193014000  413696 (404.0 KiB)    /system/lib64/libhidltransport.so
libvintf.so                                      0x719335d000  364544 (356.0 KiB)    /system/lib64/libvintf.so
libnativewindow.so                               0x7192725000  24576 (24.0 KiB)      /system/lib64/libnativewindow.so
libhwui.so                                       0x7191841000  1024000 (1000.0 KiB)  /system/lib64/libhwui.so
libbacktrace.so                                  0x7193317000  114688 (112.0 KiB)    /system/lib64/libbacktrace.so
libvndksupport.so                                0x7192676000  12288 (12.0 KiB)      /system/lib64/libvndksupport.so
libaudiomanager.so                               0x71937b2000  40960 (40.0 KiB)      /system/lib64/libaudiomanager.so
libstagefright.so                                0x719275a000  3280896 (3.1 MiB)     /system/lib64/libstagefright.so
libstagefright_foundation.so                     0x7191aa0000  274432 (268.0 KiB)    /system/lib64/libstagefright_foundation.so
libstagefright_http_support.so                   0x719234f000  24576 (24.0 KiB)      /system/lib64/libstagefright_http_support.so
android.hardware.memtrack@1.0.so                 0x71904d4000  110592 (108.0 KiB)    /system/lib64/android.hardware.memtrack@1.0.so
android.hardware.graphics.allocator@2.0.so       0x71920e7000  98304 (96.0 KiB)      /system/lib64/android.hardware.graphics.allocator@2.0.so
android.hardware.graphics.mapper@2.0.so          0x71926d9000  122880 (120.0 KiB)    /system/lib64/android.hardware.graphics.mapper@2.0.so
android.hardware.configstore@1.0.so              0x719315a000  143360 (140.0 KiB)    /system/lib64/android.hardware.configstore@1.0.so
android.hardware.configstore-utils.so            0x7192bc5000  16384 (16.0 KiB)      /system/lib64/android.hardware.configstore-utils.so
libsync.so                                       0x7192288000  16384 (16.0 KiB)      /system/lib64/libsync.so
android.hidl.token@1.0-utils.so                  0x7191a76000  20480 (20.0 KiB)      /system/lib64/android.hidl.token@1.0-utils.so
android.hardware.graphics.bufferqueue@1.0.so     0x718fac0000  253952 (248.0 KiB)    /system/lib64/android.hardware.graphics.bufferqueue@1.0.so
libdng_sdk.so                                    0x719004b000  835584 (816.0 KiB)    /system/lib64/libdng_sdk.so
libexpat.so                                      0x7193740000  163840 (160.0 KiB)    /system/lib64/libexpat.so
libft2.so                                        0x718fb06000  499712 (488.0 KiB)    /system/lib64/libft2.so
libheif.so                                       0x7192fee000  36864 (36.0 KiB)      /system/lib64/libheif.so
libicui18n.so                                    0x7191dc5000  2277376 (2.2 MiB)     /system/lib64/libicui18n.so
libpiex.so                                       0x71917d7000  106496 (104.0 KiB)    /system/lib64/libpiex.so
libpng.so                                        0x7190386000  212992 (208.0 KiB)    /system/lib64/libpng.so
libpcre2.so                                      0x71916d5000  139264 (136.0 KiB)    /system/lib64/libpcre2.so
libpackagelistparser.so                          0x7192391000  16384 (16.0 KiB)      /system/lib64/libpackagelistparser.so
libclang_rt.ubsan_standalone-aarch64-android.so  0x7192c51000  3600384 (3.4 MiB)     /system/lib64/libclang_rt.ubsan_standalone-aarch64-android.so
android.hardware.media.omx@1.0.so                0x7191602000  483328 (472.0 KiB)    /system/lib64/android.hardware.media.omx@1.0.so
android.hardware.media@1.0.so                    0x7191bf0000  32768 (32.0 KiB)      /system/lib64/android.hardware.media@1.0.so
libsonivox.so                                    0x71933c9000  401408 (392.0 KiB)    /system/lib64/libsonivox.so
libaudioutils.so                                 0x7192019000  81920 (80.0 KiB)      /system/lib64/libaudioutils.so
libmedia_helper.so                               0x719174e000  98304 (96.0 KiB)      /system/lib64/libmedia_helper.so
libmediadrm.so                                   0x7192302000  229376 (224.0 KiB)    /system/lib64/libmediadrm.so
libmediametrics.so                               0x7193822000  69632 (68.0 KiB)      /system/lib64/libmediametrics.so
libhidlmemory.so                                 0x7190162000  20480 (20.0 KiB)      /system/lib64/libhidlmemory.so
android.hidl.memory@1.0.so                       0x7190882000  143360 (140.0 KiB)    /system/lib64/android.hidl.memory@1.0.so
android.hardware.graphics.common@1.0.so          0x71923cd000  49152 (48.0 KiB)      /system/lib64/android.hardware.graphics.common@1.0.so
libtinyxml2.so                                   0x7191989000  69632 (68.0 KiB)      /system/lib64/libtinyxml2.so
libprotobuf-cpp-lite.so                          0x719221a000  270336 (264.0 KiB)    /system/lib64/libprotobuf-cpp-lite.so
libRScpp.so                                      0x71936ae000  299008 (292.0 KiB)    /system/lib64/libRScpp.so
libunwind.so                                     0x719212a000  544768 (532.0 KiB)    /system/lib64/libunwind.so
libdrmframework.so                               0x7190b99000  147456 (144.0 KiB)    /system/lib64/libdrmframework.so
libmediautils.so                                 0x71932ca000  65536 (64.0 KiB)      /system/lib64/libmediautils.so
libvorbisidec.so                                 0x719261f000  131072 (128.0 KiB)    /system/lib64/libvorbisidec.so
libstagefright_omx_utils.so                      0x7191b27000  28672 (28.0 KiB)      /system/lib64/libstagefright_omx_utils.so
libstagefright_flacdec.so                        0x71919d0000  114688 (112.0 KiB)    /system/lib64/libstagefright_flacdec.so
libstagefright_xmlparser.so                      0x71920b0000  61440 (60.0 KiB)      /system/lib64/libstagefright_xmlparser.so
android.hidl.allocator@1.0.so                    0x7191cd8000  98304 (96.0 KiB)      /system/lib64/android.hidl.allocator@1.0.so
android.hardware.cas@1.0.so                      0x71914b1000  274432 (268.0 KiB)    /system/lib64/android.hardware.cas@1.0.so
android.hardware.cas.native@1.0.so               0x7193707000  122880 (120.0 KiB)    /system/lib64/android.hardware.cas.native@1.0.so
libpowermanager.so                               0x7191d41000  28672 (28.0 KiB)      /system/lib64/libpowermanager.so
android.hidl.token@1.0.so                        0x71934d8000  102400 (100.0 KiB)    /system/lib64/android.hidl.token@1.0.so
libstdc++.so                                     0x719053b000  20480 (20.0 KiB)      /system/lib64/libstdc++.so
libspeexresampler.so                             0x71935a8000  24576 (24.0 KiB)      /system/lib64/libspeexresampler.so
android.hardware.drm@1.0.so                      0x7193240000  434176 (424.0 KiB)    /system/lib64/android.hardware.drm@1.0.so
liblzma.so                                       0x719024f000  172032 (168.0 KiB)    /system/lib64/liblzma.so
libmedia_omx.so                                  0x7193459000  389120 (380.0 KiB)    /system/lib64/libmedia_omx.so
libart.so                                        0x710efe0000  6352896 (6.1 MiB)     /system/lib64/libart.so
liblz4.so                                        0x710ef51000  86016 (84.0 KiB)      /system/lib64/liblz4.so
libtombstoned_client.so                          0x710efa3000  24576 (24.0 KiB)      /system/lib64/libtombstoned_client.so
libsigchain.so                                   0x710fa42000  12288 (12.0 KiB)      /system/lib64/libsigchain.so
boot.oat                                         0x7075f000    8482816 (8.1 MiB)     /system/framework/arm64/boot.oat
boot-core-libart.oat                             0x70f76000    3497984 (3.3 MiB)     /system/framework/arm64/boot-core-libart.oat
boot-conscrypt.oat                               0x712cc000    487424 (476.0 KiB)    /system/framework/arm64/boot-conscrypt.oat
boot-okhttp.oat                                  0x71343000    622592 (608.0 KiB)    /system/framework/arm64/boot-okhttp.oat
boot-bouncycastle.oat                            0x713db000    487424 (476.0 KiB)    /system/framework/arm64/boot-bouncycastle.oat
boot-apache-xml.oat                              0x71452000    36864 (36.0 KiB)      /system/framework/arm64/boot-apache-xml.oat
boot-legacy-test.oat                             0x7145b000    36864 (36.0 KiB)      /system/framework/arm64/boot-legacy-test.oat
boot-ext.oat                                     0x71464000    278528 (272.0 KiB)    /system/framework/arm64/boot-ext.oat
boot-framework.oat                               0x714a8000    24780800 (23.6 MiB)   /system/framework/arm64/boot-framework.oat
boot-telephony-common.oat                        0x72c4a000    3039232 (2.9 MiB)     /system/framework/arm64/boot-telephony-common.oat
boot-voip-common.oat                             0x72f30000    94208 (92.0 KiB)      /system/framework/arm64/boot-voip-common.oat
boot-ims-common.oat                              0x72f47000    114688 (112.0 KiB)    /system/framework/arm64/boot-ims-common.oat
boot-org.apache.http.legacy.boot.oat             0x72f63000    638976 (624.0 KiB)    /system/framework/arm64/boot-org.apache.http.legacy.boot.oat
boot-android.hidl.base-V1.0-java.oat             0x72fff000    24576 (24.0 KiB)      /system/framework/arm64/boot-android.hidl.base-V1.0-java.oat
boot-android.hidl.manager-V1.0-java.oat          0x73005000    45056 (44.0 KiB)      /system/framework/arm64/boot-android.hidl.manager-V1.0-java.oat
libandroid.so                                    0x7109adc000  122880 (120.0 KiB)    /system/lib64/libandroid.so
libaaudio.so                                     0x7109a83000  196608 (192.0 KiB)    /system/lib64/libaaudio.so
libcamera2ndk.so                                 0x7109985000  126976 (124.0 KiB)    /system/lib64/libcamera2ndk.so
libmediandk.so                                   0x71099d8000  114688 (112.0 KiB)    /system/lib64/libmediandk.so
libmedia_jni.so                                  0x710990c000  438272 (428.0 KiB)    /system/lib64/libmedia_jni.so
libmidi.so                                       0x7109a17000  73728 (72.0 KiB)      /system/lib64/libmidi.so
libmtp.so                                        0x71098c0000  180224 (176.0 KiB)    /system/lib64/libmtp.so
libexif.so                                       0x7109a40000  237568 (232.0 KiB)    /system/lib64/libexif.so
libGLESv3.so                                     0x7109891000  94208 (92.0 KiB)      /system/lib64/libGLESv3.so
libjnigraphics.so                                0x7109851000  12288 (12.0 KiB)      /system/lib64/libjnigraphics.so
libneuralnetworks.so                             0x7109582000  2207744 (2.1 MiB)     /system/lib64/libneuralnetworks.so
libtextclassifier_hash.so                        0x7109548000  20480 (20.0 KiB)      /system/lib64/libtextclassifier_hash.so
android.hardware.neuralnetworks@1.0.so           0x71097f5000  278528 (272.0 KiB)    /system/lib64/android.hardware.neuralnetworks@1.0.so
libOpenMAXAL.so                                  0x7109514000  16384 (16.0 KiB)      /system/lib64/libOpenMAXAL.so
libOpenSLES.so                                   0x71094c0000  20480 (20.0 KiB)      /system/lib64/libOpenSLES.so
libRS.so                                         0x7109428000  77824 (76.0 KiB)      /system/lib64/libRS.so
android.hardware.renderscript@1.0.so             0x7109455000  389120 (380.0 KiB)    /system/lib64/android.hardware.renderscript@1.0.so
libwebviewchromium_plat_support.so               0x71093cf000  20480 (20.0 KiB)      /system/lib64/libwebviewchromium_plat_support.so
libjavacore.so                                   0x7109352000  303104 (296.0 KiB)    /system/lib64/libjavacore.so
libopenjdk.so                                    0x7107c03000  245760 (240.0 KiB)    /system/lib64/libopenjdk.so
libssl.so                                        0x7107c71000  290816 (284.0 KiB)    /system/lib64/libssl.so
libopenjdkjvm.so                                 0x7107ccb000  40960 (40.0 KiB)      /system/lib64/libopenjdkjvm.so
libart-compiler.so                               0x7107921000  2953216 (2.8 MiB)     /system/lib64/libart-compiler.so
libart-dexlayout.so                              0x710768a000  221184 (216.0 KiB)    /system/lib64/libart-dexlayout.so
libvixl-arm.so                                   0x71077d9000  1200128 (1.1 MiB)     /system/lib64/libvixl-arm.so
libvixl-arm64.so                                 0x71076c5000  962560 (940.0 KiB)    /system/lib64/libvixl-arm64.so
libsoundpool.so                                  0x7104d8d000  57344 (56.0 KiB)      /system/lib64/libsoundpool.so
libjavacrypto.so                                 0x7104d43000  245760 (240.0 KiB)    /system/lib64/libjavacrypto.so
android.hardware.graphics.mapper@2.0-impl.so     0x7104cb5000  45056 (44.0 KiB)      /vendor/lib64/hw/android.hardware.graphics.mapper@2.0-impl.so
libEGL_adreno.so                                 0x7104c1d000  114688 (112.0 KiB)    /vendor/lib64/egl/libEGL_adreno.so
libadreno_utils.so                               0x7104c6f000  49152 (48.0 KiB)      /vendor/lib64/libadreno_utils.so
libgsl.so                                        0x7104ad8000  1114112 (1.1 MiB)     /vendor/lib64/libgsl.so
libGLESv2_adreno.so                              0x7104719000  3416064 (3.3 MiB)     /vendor/lib64/egl/libGLESv2_adreno.so
libllvm-glnext.so                                0x7100e88000  13807616 (13.2 MiB)   /vendor/lib64/libllvm-glnext.so
libGLESv1_CM_adreno.so                           0x71046cc000  212992 (208.0 KiB)    /vendor/lib64/egl/libGLESv1_CM_adreno.so
eglSubDriverAndroid.so                           0x71046a8000  69632 (68.0 KiB)      /vendor/lib64/egl/eglSubDriverAndroid.so
libcompiler_rt.so                                0x7100de5000  585728 (572.0 KiB)    /system/lib64/libcompiler_rt.so
libwebviewchromium_loader.so                     0x7104665000  16384 (16.0 KiB)      /system/lib64/libwebviewchromium_loader.so
base.odex                                        0x70f998e000  176128 (172.0 KiB)    /data/app/com.example.jnitest-_kMYsv5DEnZnBL4QS2mXJQ==/oat/arm64/base.odex
libjnitest.so                                    0x70f9784000  221184 (216.0 KiB)    /data/app/com.example.jnitest-_kMYsv5DEnZnBL4QS2mXJQ==/lib/arm64/libjnitest...
frida-agent-64.so                                0x70f6899000  22810624 (21.8 MiB)   /data/local/tmp/re.frida.server/frida-agent-64.so
linux-vdso.so.1                                  0x71940bb000  4096 (4.0 KiB)        linux-vdso.so.1
linker64                                         0x71940bd000  999424 (976.0 KiB)    /system/bin/linker64
com.example.jnitest on (google: 8.1.0) [usb] # memory list exports libjnitest.so
Save the output by adding `--json exports.json` to this command
Type      Name                                                                                                             Address
--------  ---------------------------------------------------------------------------------------------------------------  ------------
variable  _ZTSN10__cxxabiv117__array_type_infoE                                                                            0x70f97ae740
function  __cxa_call_unexpected                                                                                            0x70f9795454
function  _ZNSt12length_errorD1Ev                                                                                          0x70f9795c98
variable  _ZTVN10__cxxabiv123__fundamental_type_infoE                                                                      0x70f97b7f40
variable  _ZTIPDn                                                                                                          0x70f97b7ff0
function  __cxa_free_exception                                                                                             0x70f9794374
variable  _ZTIN10__cxxabiv117__array_type_infoE                                                                            0x70f97b8798
function  _ZN10__cxxabiv121__isOurExceptionClassEPK17_Unwind_Exception                                                     0x70f9794308
function  _ZNSt11logic_errorC2EPKc                                                                                         0x70f9794040
variable  _ZTIPDs                                                                                                          0x70f97b86d0
variable  _ZTVN10__cxxabiv129__pointer_to_member_type_infoE                                                                0x70f97b89c8
variable  _ZTIPDu                                                                                                          0x70f97b8680
function  _ZnamSt11align_val_tRKSt9nothrow_t                                                                               0x70f9793f68
function  _ZNSt10bad_typeidD2Ev                                                                                            0x70f97a8b50
variable  __cxa_new_handler                                                                                                0x70f97b9038
function  _ZNSt13bad_exceptionD1Ev                                                                                         0x70f9795c2c
variable  _ZTVSt15underflow_error                                                                                          0x70f97b6068
function  _ZdlPvm                                                                                                          0x70f9793e7c
variable  _ZTVSt11range_error                                                                                              0x70f97b5fd0
function  _ZNSt6__ndk117_DeallocateCaller27__do_deallocate_handle_sizeEPvm                                                 0x70f9793448
function  _ZdaPvSt11align_val_t                                                                                            0x70f9793fa4
function  _ZNSt11logic_errorC1ERKS_                                                                                        0x70f97940c0
variable  _ZTSN10__cxxabiv123__fundamental_type_infoE                                                                      0x70f97ae628
function  _ZNSt13runtime_errorC1EPKc                                                                                       0x70f97941dc
function  __gxx_personality_v0                                                                                             0x70f9794c6c
function  __cxa_throw                                                                                                      0x70f97943c8
variable  _ZTISt12length_error                                                                                             0x70f97b5f78
function  _ZNSt9bad_allocC2Ev                                                                                              0x70f9795c50
function  _ZNSt12domain_errorD0Ev                                                                                          0x70f9795e08
variable  _ZTSSt16invalid_argument                                                                                         0x70f97ac6c4
function  _Z52Java_com_example_jnitest_MainActivity_stringFromJNI2P7_JNIEnvP8_jobject                                      0x70f97931ac
variable  _ZTSN10__cxxabiv116__enum_type_infoE                                                                             0x70f97ae762
function  _ZNSt6__ndk111char_traitsIcE6assignERcRKc                                                                        0x70f97939e0
variable  _ZTSSt12out_of_range                                                                                             0x70f97ac6ea
function  _ZdlPvmSt11align_val_t                                                                                           0x70f9793f9c
function  _ZNSt12length_errorD2Ev                                                                                          0x70f9795c98
variable  _ZTISt8bad_cast                                                                                                  0x70f97b8a80
variable  _ZTVN10__cxxabiv121__vmi_class_type_infoE                                                                        0x70f97b88f0
variable  _ZTVSt9bad_alloc                                                                                                 0x70f97b5d70
function  _ZSt14set_unexpectedPFvvE                                                                                        0x70f979617c
function  _ZNKSt13bad_exception4whatEv                                                                                     0x70f9795c44
function  _ZdlPv                                                                                                           0x70f9793e74
function  _ZNSt13runtime_errorC2ERKS_                                                                                      0x70f979425c
variable  _ZTISt16invalid_argument                                                                                         0x70f97b5f38
function  __cxa_uncaught_exceptions                                                                                        0x70f9794a1c
function  _ZNSt11logic_erroraSERKS_                                                                                        0x70f97940f0
function  _ZNSt13runtime_errorC1ERKNSt6__ndk112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE                     0x70f9794150
function  _ZdlPvSt11align_val_tRKSt9nothrow_t                                                                              0x70f9793f98
function  __cxa_current_exception_type                                                                                     0x70f97946a8
function  _ZNSt13bad_exceptionD2Ev                                                                                         0x70f9795c2c
function  _ZN7_JNIEnv12NewStringUTFEPKc                                                                                    0x70f97930dc
variable  _ZTIa                                                                                                            0x70f97b8170
function  _ZSt10unexpectedv                                                                                                0x70f9794bd8
variable  _ZTIb                                                                                                            0x70f97b8030
variable  _ZTSN10__cxxabiv121__vmi_class_type_infoE                                                                        0x70f97ae7a8
function  _ZNKSt6__ndk121__basic_string_commonILb1EE20__throw_length_errorEv                                               0x70f9793720
variable  _ZTIc                                                                                                            0x70f97b80d0
variable  _ZTId                                                                                                            0x70f97b8580
variable  _ZTIe                                                                                                            0x70f97b85d0
variable  _ZTIf                                                                                                            0x70f97b8530
variable  _ZTIg                                                                                                            0x70f97b8620
variable  _ZTSN10__cxxabiv129__pointer_to_member_type_infoE                                                                0x70f97ae5e2
variable  _ZTIh                                                                                                            0x70f97b8120
variable  _ZTIi                                                                                                            0x70f97b8260
function  __cxa_rethrow_primary_exception                                                                                  0x70f9794868
function  _ZNSt9type_infoD0Ev                                                                                              0x70f97a8af0
variable  _ZTIj                                                                                                            0x70f97b82b0
variable  _ZTSSt15underflow_error                                                                                          0x70f97ac730
variable  _ZTIl                                                                                                            0x70f97b8300
variable  _ZTVN10__cxxabiv120__si_class_type_infoE                                                                         0x70f97b8888
variable  _ZTIm                                                                                                            0x70f97b8350
function  _ZNSt12domain_errorD1Ev                                                                                          0x70f9795c98
function  _ZnwmRKSt9nothrow_t                                                                                              0x70f9793e18
variable  _ZTIn                                                                                                            0x70f97b8440
function  _ZNSt6__ndk112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE10__align_itILm16EEEmm                         0x70f9793bb0
variable  _ZTVSt8bad_cast                                                                                                  0x70f97b8a00
variable  _ZTISt10bad_typeid                                                                                               0x70f97b8a98
function  _ZNSt8bad_castD0Ev                                                                                               0x70f97a8b0c
variable  _ZTIo                                                                                                            0x70f97b8490
function  _ZNSt9exceptionD0Ev                                                                                              0x70f9795c30
function  _ZNSt10bad_typeidC1Ev                                                                                            0x70f97a8b3c
variable  _ZTSPKa                                                                                                          0x70f97ae68e
variable  _ZTSSt10bad_typeid                                                                                               0x70f97ae805
variable  _ZTSPKb                                                                                                          0x70f97ae66a
variable  _ZTIs                                                                                                            0x70f97b81c0
variable  _ZTSPKc                                                                                                          0x70f97ae67c
variable  _ZTIt                                                                                                            0x70f97b8210
function  _ZSt13set_terminatePFvvE                                                                                         0x70f97961a4
variable  _ZTSPKd                                                                                                          0x70f97ae706
function  _ZNSt6__ndk112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEEC2IDnEEPKc                                     0x70f9793084
variable  _ZTSPKe                                                                                                          0x70f97ae70f
function  _ZSt17__throw_bad_allocv                                                                                         0x70f9793d80
variable  _ZTIv                                                                                                            0x70f97b7f90
function  _ZNSt13runtime_errorC2EPKc                                                                                       0x70f97941dc
variable  _ZTSPKf                                                                                                          0x70f97ae6fd
variable  _ZTSPKg                                                                                                          0x70f97ae718
variable  _ZTIw                                                                                                            0x70f97b8080
variable  _ZTSPa                                                                                                           0x70f97ae68b
variable  _ZTSSt20bad_array_new_length                                                                                     0x70f97ac68a
function  _ZnamRKSt9nothrow_t                                                                                              0x70f9793e48
variable  _ZTVSt16invalid_argument                                                                                         0x70f97b5f10
variable  _ZTSPKh                                                                                                          0x70f97ae685
variable  _ZTVSt12length_error                                                                                             0x70f97b5f50
variable  _ZTSPb                                                                                                           0x70f97ae667
variable  _ZTIx                                                                                                            0x70f97b83a0
variable  _ZTIy                                                                                                            0x70f97b83f0
variable  _ZTSPc                                                                                                           0x70f97ae679
variable  _ZTSPKi                                                                                                          0x70f97ae6a9
variable  _ZTSPKj                                                                                                          0x70f97ae6b2
variable  _ZTSPd                                                                                                           0x70f97ae703
variable  _ZTIN10__cxxabiv121__vmi_class_type_infoE                                                                        0x70f97b8940
variable  _ZTSPe                                                                                                           0x70f97ae70c
variable  _ZTIN10__cxxabiv123__fundamental_type_infoE                                                                      0x70f97b7f78
function  _ZNKSt10bad_typeid4whatEv                                                                                        0x70f97a8b78
variable  _ZTSPKl                                                                                                          0x70f97ae6bb
variable  _ZTSPf                                                                                                           0x70f97ae6fa
function  _ZNKSt8bad_cast4whatEv                                                                                           0x70f97a8b30
function  _ZNSt20bad_array_new_lengthD0Ev                                                                                  0x70f9795c88
variable  _ZTSPKm                                                                                                          0x70f97ae6c4
variable  _ZTSPg                                                                                                           0x70f97ae715
variable  _ZTVN10__cxxabiv117__class_type_infoE                                                                            0x70f97b8838
function  _ZNSt14overflow_errorD0Ev                                                                                        0x70f9795fd4
function  _ZSt9terminatev                                                                                                  0x70f9794b80
variable  _ZTSPKn                                                                                                          0x70f97ae6df
variable  _ZTSPh                                                                                                           0x70f97ae682
variable  _ZTSN10__cxxabiv117__pbase_type_infoE                                                                            0x70f97ae577
variable  _ZTISt13bad_exception                                                                                            0x70f97b5e20
variable  _ZTSPi                                                                                                           0x70f97ae6a6
variable  _ZTSSt9type_info                                                                                                 0x70f97ae7ec
variable  _ZTSPKo                                                                                                          0x70f97ae6e8
variable  _ZTSPj                                                                                                           0x70f97ae6af
variable  _ZTVN10__cxxabiv120__function_type_infoE                                                                         0x70f97b87b0
variable  _ZTSPl                                                                                                           0x70f97ae6b8
variable  _ZTIN10__cxxabiv117__pbase_type_infoE                                                                            0x70f97b7ea8
variable  _ZTSPm                                                                                                           0x70f97ae6c1
function  _ZnwmSt11align_val_t                                                                                             0x70f9793e8c
variable  _ZTIN10__cxxabiv116__shim_type_infoE                                                                             0x70f97b7e78
variable  _ZTSPKs                                                                                                          0x70f97ae697
variable  _ZTSPn                                                                                                           0x70f97ae6dc
variable  _ZTSPKt                                                                                                          0x70f97ae6a0
variable  _ZTSPo                                                                                                           0x70f97ae6e5
function  _ZSt14get_unexpectedv                                                                                            0x70f9794b70
variable  _ZTSPKv                                                                                                          0x70f97ae655
variable  _ZTSSt9exception                                                                                                 0x70f97ac65e
variable  _ZTSPKw                                                                                                          0x70f97ae673
variable  _ZTSPKx                                                                                                          0x70f97ae6cd
variable  _ZTSPs                                                                                                           0x70f97ae694
function  _ZNKSt20bad_array_new_length4whatEv                                                                              0x70f9795c8c
variable  _ZTSPKy                                                                                                          0x70f97ae6d6
variable  _ZTSPt                                                                                                           0x70f97ae69d
function  __cxa_increment_exception_refcount                                                                               0x70f97947d8
variable  _ZTSPv                                                                                                           0x70f97ae652
variable  _ZTSPw                                                                                                           0x70f97ae670
function  _ZNSt6__ndk111char_traitsIcE4copyEPcPKcm                                                                         0x70f9793904
variable  _ZTSPx                                                                                                           0x70f97ae6ca
function  _ZNSt9type_infoD1Ev                                                                                              0x70f97a8aec
variable  _ZTSPy                                                                                                           0x70f97ae6d3
function  _ZNSt6__ndk112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE6__initEPKcm                                   0x70f9793538
function  _ZNSt12domain_errorD2Ev                                                                                          0x70f9795c98
variable  _ZTSSt13bad_exception                                                                                            0x70f97ac66b
function  _ZNSt8bad_castD1Ev                                                                                               0x70f97a8b08
function  _ZNSt13runtime_errorC1ERKS_                                                                                      0x70f979425c
function  _ZNSt9exceptionD1Ev                                                                                              0x70f9795c2c
variable  _ZTSSt12length_error                                                                                             0x70f97ac6d9
function  _ZNSt10bad_typeidC2Ev                                                                                            0x70f97a8b3c
variable  _ZTIDh                                                                                                           0x70f97b84e0
variable  _ZTIDi                                                                                                           0x70f97b8710
variable  _ZTSPDh                                                                                                          0x70f97ae6ef
variable  _ZTSPDi                                                                                                          0x70f97ae737
function  __cxa_uncaught_exception                                                                                         0x70f97949f4
function  _ZNSt13runtime_errorD0Ev                                                                                         0x70f9795da4
variable  _ZTVN10__cxxabiv116__shim_type_infoE                                                                             0x70f97b7f08
variable  _ZTIDn                                                                                                           0x70f97b7fe0
variable  _ZTSPDn                                                                                                          0x70f97ae65c
function  _ZdaPvmSt11align_val_t                                                                                           0x70f9793fac
function  _ZNSt6__ndk117_DeallocateCaller9__do_callEPv                                                                     0x70f9793470
variable  _ZTIDs                                                                                                           0x70f97b86c0
function  _ZNSt20bad_array_new_lengthD1Ev                                                                                  0x70f9795c2c
variable  _ZTVSt10bad_typeid                                                                                               0x70f97b8a28
function  _ZdaPvRKSt9nothrow_t                                                                                             0x70f9793e84
function  _ZNSt14overflow_errorD1Ev                                                                                        0x70f9795d50
variable  _ZTIDu                                                                                                           0x70f97b8670
variable  _ZTSPDs                                                                                                          0x70f97ae72b
variable  _ZTSa                                                                                                            0x70f97ae689
variable  _ZTIN10__cxxabiv119__pointer_type_infoE                                                                          0x70f97b7ec0
function  _ZNSt15underflow_errorD0Ev                                                                                       0x70f9796030
variable  _ZTSPDu                                                                                                          0x70f97ae71f
variable  _ZTVSt14overflow_error                                                                                           0x70f97b6028
variable  _ZTSb                                                                                                            0x70f97ae665
variable  _ZTSc                                                                                                            0x70f97ae677
variable  _ZTSd                                                                                                            0x70f97ae701
function  _ZNSt9bad_allocD0Ev                                                                                              0x70f9795c64
variable  _ZTSe                                                                                                            0x70f97ae70a
variable  _ZTSf                                                                                                            0x70f97ae6f8
variable  _ZTSg                                                                                                            0x70f97ae713
variable  _ZTSSt13runtime_error                                                                                            0x70f97ac70b
variable  _ZTSh                                                                                                            0x70f97ae680
variable  _ZTSi                                                                                                            0x70f97ae6a4
variable  _ZTSN10__cxxabiv119__pointer_type_infoE                                                                          0x70f97ae599
variable  _ZTSj                                                                                                            0x70f97ae6ad
function  __cxa_rethrow                                                                                                    0x70f97946f4
variable  _ZTISt12domain_error                                                                                             0x70f97b5ef8
variable  _ZTSl                                                                                                            0x70f97ae6b6
variable  _ZTSm                                                                                                            0x70f97ae6bf
variable  _ZTISt9type_info                                                                                                 0x70f97b8a70
function  __cxa_end_catch                                                                                                  0x70f979457c
variable  _ZTIN10__cxxabiv120__si_class_type_infoE                                                                         0x70f97b88d8
variable  _ZTSn                                                                                                            0x70f97ae6da
function  _ZNSt6__ndk111char_traitsIcE6lengthEPKc                                                                          0x70f979366c
variable  _ZTSo                                                                                                            0x70f97ae6e3
function  _ZNSt13runtime_erroraSERKS_                                                                                      0x70f979428c
function  _ZSt13get_terminatev                                                                                             0x70f9794bf0
function  _ZNSt9type_infoD2Ev                                                                                              0x70f97a8aec
variable  __cxa_unexpected_handler                                                                                         0x70f97b9010
variable  _ZTSSt9bad_alloc                                                                                                 0x70f97ac67d
function  _ZSt15set_new_handlerPFvvE                                                                                       0x70f9794c40
variable  _ZTSs                                                                                                            0x70f97ae692
variable  _ZTSt                                                                                                            0x70f97ae69b
function  _ZNSt11logic_errorD0Ev                                                                                           0x70f9795cec
variable  _ZTISt9exception                                                                                                 0x70f97b5de8
variable  _ZTISt13runtime_error                                                                                            0x70f97b5ff8
function  _ZNSt8bad_castD2Ev                                                                                               0x70f97a8b08
variable  _ZTSv                                                                                                            0x70f97ae650
function  _ZNSt9exceptionD2Ev                                                                                              0x70f9795c2c
function  _ZNSt11logic_errorC2ERKNSt6__ndk112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE                       0x70f9793fb4
variable  _ZTSw                                                                                                            0x70f97ae66e
variable  _ZTVN10__cxxabiv117__array_type_infoE                                                                            0x70f97b8760
function  _ZNSt11range_errorD0Ev                                                                                           0x70f9795f78
function  __dynamic_cast                                                                                                   0x70f97a7db4
function  _ZNSt12out_of_rangeD0Ev                                                                                          0x70f9795f1c
variable  _ZTSx                                                                                                            0x70f97ae6c8
variable  _ZTSy                                                                                                            0x70f97ae6d1
variable  _ZTSSt8bad_cast                                                                                                  0x70f97ae7f9
function  _ZNSt13runtime_errorD1Ev                                                                                         0x70f9795d50
variable  __cxa_terminate_handler                                                                                          0x70f97b9008
variable  _ZTIN10__cxxabiv120__function_type_infoE                                                                         0x70f97b7ed8
function  Java_com_example_jnitest_MainActivity_stringFromJNI                                                              0x70f9792fcc
function  _ZNSt20bad_array_new_lengthD2Ev                                                                                  0x70f9795c2c
function  _ZNSt14overflow_errorD2Ev                                                                                        0x70f9795d50
function  _ZNSt16invalid_argumentD0Ev                                                                                      0x70f9795e64
function  __cxa_allocate_exception                                                                                         0x70f9794328
function  _ZNSt15underflow_errorD1Ev                                                                                       0x70f9795d50
variable  _ZTSN10__cxxabiv120__si_class_type_infoE                                                                         0x70f97ae783
function  _ZNSt9bad_allocD1Ev                                                                                              0x70f9795c2c
variable  _ZTIN10__cxxabiv116__enum_type_infoE                                                                             0x70f97b8820
function  _ZdaPv                                                                                                           0x70f9793e80
function  _ZNKSt11logic_error4whatEv                                                                                       0x70f9795d48
variable  _ZTSN10__cxxabiv117__class_type_infoE                                                                            0x70f97ae555
function  _ZN10__cxxabiv119__getExceptionClassEPK17_Unwind_Exception                                                       0x70f9794300
function  __cxa_get_globals                                                                                                0x70f9794a3c
function  _ZNSt6__ndk117__compressed_pairINS_12basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEE5__repES5_EC2ILb1EvEEv  0x70f9793504
function  __cxa_get_exception_ptr                                                                                          0x70f97944e0
variable  _ZTISt14overflow_error                                                                                           0x70f97b6050
function  __cxa_pure_virtual                                                                                               0x70f97a8b84
function  _ZNSt11logic_errorD1Ev                                                                                           0x70f9795c98
variable  _ZTIPKDh                                                                                                         0x70f97b8510
variable  _ZTIPKDi                                                                                                         0x70f97b8740
function  _Znwm                                                                                                            0x70f9793db4
variable  _ZTSN10__cxxabiv120__function_type_infoE                                                                         0x70f97ae5bd
function  _ZNSt11range_errorD1Ev                                                                                           0x70f9795d50
function  _ZNSt12out_of_rangeD1Ev                                                                                          0x70f9795c98
variable  _ZTSSt11logic_error                                                                                              0x70f97ac6b4
variable  _ZTVSt12domain_error                                                                                             0x70f97b5eb8
variable  _ZTIPKDn                                                                                                         0x70f97b8010
variable  _ZTVN10__cxxabiv119__pointer_type_infoE                                                                          0x70f97b8990
function  _ZNSt13runtime_errorD2Ev                                                                                         0x70f9795d50
variable  _ZTISt15underflow_error                                                                                          0x70f97b6090
function  __cxa_decrement_exception_refcount                                                                               0x70f9794658
variable  _ZTVN10__cxxabiv116__enum_type_infoE                                                                             0x70f97b87e8
variable  _ZTISt9bad_alloc                                                                                                 0x70f97b5e38
function  __cxa_allocate_dependent_exception                                                                               0x70f9794390
variable  _ZTIPKDs                                                                                                         0x70f97b86f0
function  _ZnamSt11align_val_t                                                                                             0x70f9793f64
variable  _ZTIPKDu                                                                                                         0x70f97b86a0
variable  _ZTISt20bad_array_new_length                                                                                     0x70f97b5e50
function  __cxa_demangle                                                                                                   0x70f9796308
variable  _ZTSPKDh                                                                                                         0x70f97ae6f3
variable  _ZTSPKDi                                                                                                         0x70f97ae73b
function  _ZNKSt9exception4whatEv                                                                                          0x70f9795c34
function  _ZNSt16invalid_argumentD1Ev                                                                                      0x70f9795c98
function  _ZNSt15underflow_errorD2Ev                                                                                       0x70f9795d50
function  _ZSt15get_new_handlerv                                                                                           0x70f9794c5c
function  _ZNSt8bad_castC1Ev                                                                                               0x70f97a8af4
function  _ZNSt9bad_allocD2Ev                                                                                              0x70f9795c2c
variable  _ZTSPKDn                                                                                                         0x70f97ae660
function  _ZdlPvRKSt9nothrow_t                                                                                             0x70f9793e78
function  __cxa_deleted_virtual                                                                                            0x70f97a8b98
variable  _ZTIN10__cxxabiv129__pointer_to_member_type_infoE                                                                0x70f97b7ef0
function  _ZNSt10bad_typeidD0Ev                                                                                            0x70f97a8b54
variable  _ZTSSt11range_error                                                                                              0x70f97ac6fb
variable  _ZTSPKDs                                                                                                         0x70f97ae72f
variable  _ZTVSt13bad_exception                                                                                            0x70f97b5df8
variable  _ZTSPKDu                                                                                                         0x70f97ae723
variable  _ZTIN10__cxxabiv117__class_type_infoE                                                                            0x70f97b7e90
function  _ZNKSt9bad_alloc4whatEv                                                                                          0x70f9795c68
function  _ZNSt20bad_array_new_lengthC1Ev                                                                                  0x70f9795c74
variable  _ZTSDh                                                                                                           0x70f97ae6ec
function  _ZNSt11logic_errorD2Ev                                                                                           0x70f9795c98
variable  _ZTSDi                                                                                                           0x70f97ae734
function  _ZNSt11logic_errorC1EPKc                                                                                         0x70f9794040
function  _ZNSt11range_errorD2Ev                                                                                           0x70f9795d50
function  _ZNSt12out_of_rangeD2Ev                                                                                          0x70f9795c98
variable  _ZTSDn                                                                                                           0x70f97ae659
function  __cxa_free_dependent_exception                                                                                   0x70f97943c4
variable  _ZTSN10__cxxabiv116__shim_type_infoE                                                                             0x70f97ae534
function  _ZNSt11logic_errorC2ERKS_                                                                                        0x70f97940c0
variable  _ZTIPKa                                                                                                          0x70f97b81a0
variable  _ZTIPKb                                                                                                          0x70f97b8060
variable  _ZTISt12out_of_range                                                                                             0x70f97b5fb8
variable  _ZTIPKc                                                                                                          0x70f97b8100
variable  _ZTIPKd                                                                                                          0x70f97b85b0
function  _ZdaPvSt11align_val_tRKSt9nothrow_t                                                                              0x70f9793fa8
variable  _ZTSDs                                                                                                           0x70f97ae728
variable  _ZTIPKe                                                                                                          0x70f97b8600
variable  _ZTIPKf                                                                                                          0x70f97b8560
variable  _ZTSDu                                                                                                           0x70f97ae71c
variable  _ZTVN10__cxxabiv117__pbase_type_infoE                                                                            0x70f97b8958
function  _ZnwmSt11align_val_tRKSt9nothrow_t                                                                               0x70f9793f38
function  _ZNKSt13runtime_error4whatEv                                                                                     0x70f9795e00
variable  _ZTIPKg                                                                                                          0x70f97b8650
variable  _ZSt7nothrow                                                                                                     0x70f97ac3ca
variable  _ZTIPKh                                                                                                          0x70f97b8150
function  __cxa_current_primary_exception                                                                                  0x70f97947f4
function  _ZNSt12length_errorD0Ev                                                                                          0x70f9795ec0
variable  _ZTISt11logic_error                                                                                              0x70f97b5ee0
variable  _ZTIPKi                                                                                                          0x70f97b8290
variable  _ZTVSt20bad_array_new_length                                                                                     0x70f97b5d98
function  _ZNSt13runtime_errorC2ERKNSt6__ndk112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE                     0x70f9794150
variable  _ZTIPKj                                                                                                          0x70f97b82e0
function  _ZNSt16invalid_argumentD2Ev                                                                                      0x70f9795c98
variable  _ZTIPa                                                                                                           0x70f97b8180
variable  _ZTIPKl                                                                                                          0x70f97b8330
variable  _ZTIPb                                                                                                           0x70f97b8040
variable  _ZTIPKm                                                                                                          0x70f97b8380
variable  _ZTSSt14overflow_error                                                                                           0x70f97ac71d
variable  _ZTIPKn                                                                                                          0x70f97b8470
variable  _ZTIPc                                                                                                           0x70f97b80e0
variable  _ZTIPKo                                                                                                          0x70f97b84c0
variable  _ZTIPd                                                                                                           0x70f97b8590
function  _ZNSt8bad_castC2Ev                                                                                               0x70f97a8af4
variable  _ZTIPe                                                                                                           0x70f97b85e0
variable  _ZTIPf                                                                                                           0x70f97b8540
function  _ZNSt10bad_typeidD1Ev                                                                                            0x70f97a8b50
variable  _ZTIPg                                                                                                           0x70f97b8630
variable  _ZTIPKs                                                                                                          0x70f97b81f0
variable  _ZTVSt12out_of_range                                                                                             0x70f97b5f90
variable  _ZTIPh                                                                                                           0x70f97b8130
variable  _ZTIPKt                                                                                                          0x70f97b8240
function  _ZNSt13bad_exceptionD0Ev                                                                                         0x70f9795c40
variable  _ZTIPi                                                                                                           0x70f97b8270
variable  _ZTIPj                                                                                                           0x70f97b82c0
variable  _ZTIPKv                                                                                                          0x70f97b7fc0
function  _Znam                                                                                                            0x70f9793e44
variable  _ZTIPKw                                                                                                          0x70f97b80b0
variable  _ZTIPl                                                                                                           0x70f97b8310
variable  _ZTIPKx                                                                                                          0x70f97b83d0
function  _ZdlPvSt11align_val_t                                                                                            0x70f9793f94
variable  _ZTIPm                                                                                                           0x70f97b8360
variable  _ZTIPKy                                                                                                          0x70f97b8420
variable  _ZTSSt12domain_error                                                                                             0x70f97ac6a3
variable  _ZTIPn                                                                                                           0x70f97b8450
function  _ZNSt6__ndk112basic_stringIcNS_11char_traitsIcEENS_9allocatorIcEEED2Ev                                           0x70f9793138
variable  _ZTIPo                                                                                                           0x70f97b84a0
function  _ZNSt20bad_array_new_lengthC2Ev                                                                                  0x70f9795c74
variable  _ZTIPs                                                                                                           0x70f97b81d0
variable  _ZTISt11range_error                                                                                              0x70f97b6010
variable  _ZTVSt9type_info                                                                                                 0x70f97b8a50
variable  _ZTIPt                                                                                                           0x70f97b8220
function  __cxa_begin_catch                                                                                                0x70f97944e8
variable  _ZTIPv                                                                                                           0x70f97b7fa0
variable  _ZTIPw                                                                                                           0x70f97b8090
function  __cxa_get_globals_fast                                                                                           0x70f9794acc
function  _ZNSt9bad_allocC1Ev                                                                                              0x70f9795c50
variable  _ZTIPx                                                                                                           0x70f97b83b0
variable  _ZTIPy                                                                                                           0x70f97b8400
variable  _ZTVSt11logic_error                                                                                              0x70f97b5e68
function  _ZN10__cxxabiv119__setExceptionClassEP17_Unwind_Exceptionm                                                       0x70f97942ec
variable  _ZTVSt9exception                                                                                                 0x70f97b5dc0
function  _ZNSt11logic_errorC1ERKNSt6__ndk112basic_stringIcNS0_11char_traitsIcEENS0_9allocatorIcEEEE                       0x70f9793fb4
function  _ZdaPvm                                                                                                          0x70f9793e88
variable  _ZTIPDh                                                                                                          0x70f97b84f0
variable  _ZTIPDi                                                                                                          0x70f97b8720
variable  _ZTVSt13runtime_error                                                                                            0x70f97b5e90

```

```
function  _Z52Java_com_example_jnitest_MainActivity_stringFromJNI2P7_JNIEnvP8_jobject                                      0x70f97931ac

```

再次遍历后会发现编译完成后的函数名虽然存在原始函数名的字符串，但是发生了一些变化，这种变化就是名称粉碎机制所导致的。

被名称粉碎机制破坏的函数名的hi可以还原的。

推荐工具未Linux系统自带的C++filt工具对函数名进行还原

```
c++filt _Z52Java_com_example_jnitest_MainActivity_stringFromJNI2P7_JNIEnvP8_jobject Java_com_example_jnitest_MainActivity_stringFromJNI2(_JNIEnv*,_jobject*)

```

但是报了一个不理解的错误

```
zsh: unknown file attribute: _
在博客中搜索时说有可能是spark-shell的问题，后面再看看
```

