# HTTP(S)网络框架分析

- 原生Android网络HTTP通信库
- 第三方HTTP(S)网络请求框架
  - okHttp，square公司的开源网络请求框架
  - okhttp3（大部分安卓第三方网络框架都基于其的封装，例如Retrofit2）
  - WebSocket，是HTML5规范的一部分，与HTTP不同，遵循全双工通信机制，是一种应用层协议，表示为 `ws://xxxx.xxxxxx.xxxx/?encoding=text HTTP/1.1`是一个传统的URL。通常也使用okhttp3这个第三方网络框架来完成。
  - XMPP（可扩展消息与存在协议）是目前主流的四种即时消息协议之一，前身是Jabber，一个开源形式组织产生的网络即时通信协议。

## HttpURLConnection

### HttpURLConnection基础开发流程

demo如下

```
package com.example.httpurlconnectiondemo;

import androidx.appcompat.app.AppCompatActivity;
import android.os.Bundle;
import android.util.Log;

import java.io.IOException;
import java.io.InputStream;
import java.net.HttpURLConnection;
import java.net.MalformedURLException;
import java.net.URL;

public class MainActivity extends AppCompatActivity {

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        new Thread(new Runnable() {
            @Override
            public void run() {
                while(true){
                    try {
                        URL url = new URL("https://www.baidu.com");
                        HttpURLConnection connection = (HttpURLConnection) url.openConnection();
                        connection.setRequestMethod("GET");
                        connection.setRequestProperty("token","zhy");
                        connection.setConnectTimeout(8000);
                        connection.setReadTimeout(8000);
                        InputStream in = connection.getInputStream();

                        int bufferSize = 1024;
                        byte[]buffer = new byte[bufferSize];
                        StringBuffer stringBuffer = new StringBuffer();
                        while((in.read(buffer))!=-1){
                            stringBuffer.append(new String(buffer));
                        }
                        Log.d("zhy", stringBuffer.toString());
                        connection.disconnect();
                    } catch (IOException e) {
                        e.printStackTrace();
                    }
                    try {
                        Thread.sleep(3*1000);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }
}
```

### HttpURLConnection“自吐”脚本开发

- URL类的构造函数，其包含了目标网址的字符串。
-  setRequestMethod()和setRequestProperty()函数设置请求头 和请求参数等信息。 
- getInputStream()函数获取response。

```sh
(base) ┌──(root㉿kali)-[/home/zhy/桌面]
└─# frida-ps -U
  PID  Name
-----  ---------------------------------------------------
 3358  ATFWD-daemon                                       
10320  F-Droid                                            
 5837  Google                                             
10517  Hangouts                                           
11482  HttpUrlConnectionDemo                              
10945  Magisk Manager                                     
 9460  TIM                                                
 9211  YouTube                                            
 9025  adbd                                               
 3364  android.hardware.biometrics.fingerprint@2.1-service
  437  android.hardware.cas@1.0-service                   
  438  android.hardware.configstore@1.0-service           
  439  android.hardware.dumpstate@1.0-service.bullhead    
  440  android.hardware.graphics.allocator@2.0-service    
  441  android.hardware.usb@1.0-service                   
  442  android.hardware.wifi@1.0-service                  
  436  android.hidl.allocator@1.0-service                 
 9027  android.process.media                              
 3340  audioserver                                        
 3341  cameraserver                                       
 3359  cnd                                                
 3355  cnss-daemon                                        
 4031  com.android.bluetooth                              
 5865  com.android.nfc                                    
 4289  com.android.phone                                  
 4074  com.android.systemui                               
10210  com.google.android.apps.gcs                        
 6180  com.google.android.gms                             
 5782  com.google.android.gms.persistent                  
 5952  com.google.android.googlequicksearchbox            
10579  com.google.android.googlequicksearchbox:search     
 7451  com.google.android.ims                             
 6206  com.google.android.inputmethod.latin               
10487  com.google.android.partnersetup                    
10908  com.google.android.storagemanager                  
 8023  com.google.process.gapps                           
 5970  com.google.process.gservices                       
 8347  com.qualcomm.qcrilmsgtunnel                        
 5879  com.qualcomm.qti.rcsbootstraputil                  
 5891  com.qualcomm.qti.rcsimsbootstraputil               
 8325  com.qualcomm.telephony                             
 5852  com.quicinc.cne.CNEService                         
11381  daemonsu:0                                         
11383  daemonsu:0:11378                                   
11393  daemonsu:0:11385                                   
11435  daemonsu:0:11385                                   
 7152  daemonsu:10102                                     
 3312  daemonsu:master                                    
 7179  daemonsu:mount:0                                   
 3305  daemonsu:mount:master                              
 3342  drmserver                                          
11441  frida-helper-32                                    
11385  frida-server-15.2.2-android-arm64                  
 3362  gatekeeperd                                        
  443  healthd                                            
  369  hwservicemanager                                   
 3387  imsdatadaemon                                      
 3337  imsqmidaemon                                       
    1  init                                               
 3343  installd                                           
 9967  installer                                          
 3464  ip6tables-restore                                  
 3463  iptables-restore                                   
 3344  keystore                                           
  447  lmkd                                               
 3354  loc_launcher                                       
 3391  location-mq                                        
 9901  logcat                                             
11394  logcat                                             
  367  logd                                               
 3392  lowi-server                                        
 3352  media.codec                                        
 3346  media.extractor                                    
 3347  media.metrics                                      
 3345  mediadrmserver                                     
 3348  mediaserver                                        
 3357  mm-qcamera-daemon                                  
  446  msm_irqbalance                                     
 3349  netd                                               
 3335  netmgrd                                            
 3333  perfd                                              
  500  pm-proxy                                           
  445  pm-service                                         
 3332  qmuxd                                              
  373  qseecomd                                           
  380  qseecomd                                           
 3334  qti                                                
 3353  rild                                               
  444  rmt_storage                                        
  368  servicemanager                                     
 9965  sh                                                 
11375  sh                                                 
 3393  slim_daemon                                        
 7411  sshd [listener] 0 of 10-100 startups               
 3350  storaged                                           
10991  su                                                 
11378  su                                                 
  448  surfaceflinger                                     
10999  sush                                               
11384  sush                                               
11437  sush                                               
 3780  system_server                                      
 3336  thermal-engine                                     
  449  thermalserviced                                    
 3356  time_daemon                                        
 3363  tombstoned                                         
  348  ueventd                                            
  370  vndservicemanager                                  
  400  vold                                               
 4604  wcnss_filter                                       
 4155  webview_zygote32                                   
 3351  wificond                                           
 4396  wpa_supplicant                                     
 3339  zygote                                             
 3338  zygote64                                           
 7417  信息                                                 
10836  地圖                                                 
                                                                                                                                   
(base) ┌──(root㉿kali)-[/home/zhy/桌面]
└─# objection -g HttpUrlConnectionDemo explore
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
....example.httpurlconnectiondemo on (google: 8.1.0) [usb] # android hooking watch class_method java.net.URL.$init --dump-args --du
mp-backtrace --dump-return
(agent) Attempting to watch class java.net.URL and method $init.
(agent) Hooking java.net.URL.$init(java.lang.String)
(agent) Hooking java.net.URL.$init(java.lang.String, java.lang.String, int, java.lang.String)
(agent) Hooking java.net.URL.$init(java.lang.String, java.lang.String, int, java.lang.String, java.net.URLStreamHandler)
(agent) Hooking java.net.URL.$init(java.lang.String, java.lang.String, java.lang.String)
(agent) Hooking java.net.URL.$init(java.net.URL, java.lang.String)
(agent) Hooking java.net.URL.$init(java.net.URL, java.lang.String, java.net.URLStreamHandler)
(agent) Registering job 068716. Type: watch-method for: java.net.URL.$init
....example.httpurlconnectiondemo on (google: 8.1.0) [usb] # (agent) [068716] Called java.net.URL.URL(java.lang.String)
(agent) [068716] Backtrace:
        java.net.URL.<init>(Native Method)
        com.example.httpurlconnectiondemo.MainActivity$1.run(MainActivity.java:24)
        java.lang.Thread.run(Thread.java:764)

(agent) [068716] Arguments java.net.URL.URL(https://www.baidu.com)
(agent) [068716] Called java.net.URL.URL(java.net.URL, java.lang.String)
(agent) [068716] Backtrace:
        java.net.URL.<init>(Native Method)
        java.net.URL.<init>(URL.java:436)
        java.net.URL.<init>(Native Method)
        com.example.httpurlconnectiondemo.MainActivity$1.run(MainActivity.java:24)
        java.lang.Thread.run(Thread.java:764)

(agent) [068716] Arguments java.net.URL.URL((none), https://www.baidu.com)
(agent) [068716] Called java.net.URL.URL(java.net.URL, java.lang.String, java.net.URLStreamHandler)
(agent) [068716] Backtrace:
        java.net.URL.<init>(Native Method)
        java.net.URL.<init>(URL.java:487)
        java.net.URL.<init>(Native Method)
        java.net.URL.<init>(URL.java:436)
        java.net.URL.<init>(Native Method)
        com.example.httpurlconnectiondemo.MainActivity$1.run(MainActivity.java:24)
        java.lang.Thread.run(Thread.java:764)

(agent) [068716] Arguments java.net.URL.URL((none), https://www.baidu.com, (none))
(agent) [068716] Return Value: (none)
(agent) [068716] Return Value: (none)
(agent) [068716] Return Value: (none)

```

网址在URL的构造函数中出现了，而且出现了很多次

```
(agent) [068716] Arguments java.net.URL.URL((none), https://www.baidu.com)
(agent) [068716] Called java.net.URL.URL(java.net.URL, java.lang.String, java.net.URLStreamHandler)
```

选择调用栈最浅的，也就是第一个出现的被调用的java.net.URL.$init(java.lang.String)函数完成自吐脚本

```js
function main(){
    Java.perform(function(){
        var URL = Java.use('java.net.URL')
        URL.$init.overload('java.lang.String').implementation = function(urlstr){
            console.log('url =>',urlstr)
            var result = this.$init(urlstr)
            return result
        }
    })
}

setImmediate(main)
```

测试结果如下：

```
(base) ┌──(root㉿kali)-[/home/zhy/桌面/chap8]
└─# frida -U -l HttpURLConnection.js HttpUrlConnectionDemo
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
                                                                                
[Nexus 5X::HttpUrlConnectionDemo ]-> url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com
url => https://www.baidu.com

```

根据上面整理的关键收发包函数可以发现剩下的都是HttpURLConnection类中的函数，因此可以用如下命令去watch整个HttpURLConnection类中的所有函数。

```
....example.httpurlconnectiondemo on (google: 8.1.0) [usb] # android hooking watch class_method java.net.HttpURLConnection.$init --
dump-args --dump-backtrace --dump-return
(agent) Attempting to watch class java.net.HttpURLConnection and method $init.
(agent) Hooking java.net.HttpURLConnection.$init(java.net.URL)
(agent) Registering job 118392. Type: watch-method for: java.net.HttpURLConnection.$init
....example.httpurlconnectiondemo on (google: 8.1.0) [usb] # (agent) [118392] Called java.net.HttpURLConnection.HttpURLConnection(java.net.URL)
(agent) [118392] Backtrace:
        java.net.HttpURLConnection.<init>(Native Method)
        com.android.okhttp.internal.huc.HttpURLConnectionImpl.<init>(HttpURLConnectionImpl.java:114)
        com.android.okhttp.internal.huc.HttpURLConnectionImpl.<init>(HttpURLConnectionImpl.java:119)
        com.android.okhttp.internal.huc.HttpsURLConnectionImpl.<init>(HttpsURLConnectionImpl.java:34)
        com.android.okhttp.OkUrlFactory.open(OkUrlFactory.java:63)
        com.android.okhttp.OkUrlFactory.open(OkUrlFactory.java:54)
        com.android.okhttp.HttpHandler.openConnection(HttpHandler.java:44)
        java.net.URL.openConnection(URL.java:992)
        com.example.httpurlconnectiondemo.MainActivity$1.run(MainActivity.java:25)
        java.lang.Thread.run(Thread.java:764)

(agent) [118392] Arguments java.net.HttpURLConnection.HttpURLConnection(https://www.baidu.com)
(agent) [118392] Return Value: (none)
(agent) [118392] Called java.net.HttpURLConnection.HttpURLConnection(java.net.URL)
(agent) [118392] Backtrace:
        java.net.HttpURLConnection.<init>(Native Method)
        javax.net.ssl.HttpsURLConnection.<init>(HttpsURLConnection.java:66)
        com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.<init>(DelegatingHttpsURLConnection.java:44)
        com.android.okhttp.internal.huc.HttpsURLConnectionImpl.<init>(HttpsURLConnectionImpl.java:38)
        com.android.okhttp.internal.huc.HttpsURLConnectionImpl.<init>(HttpsURLConnectionImpl.java:34)
        com.android.okhttp.OkUrlFactory.open(OkUrlFactory.java:63)
        com.android.okhttp.OkUrlFactory.open(OkUrlFactory.java:54)
        com.android.okhttp.HttpHandler.openConnection(HttpHandler.java:44)
        java.net.URL.openConnection(URL.java:992)
        com.example.httpurlconnectiondemo.MainActivity$1.run(MainActivity.java:25)
        java.lang.Thread.run(Thread.java:764)

```

同时别忘记了Hook构造函数：

```
....example.httpurlconnectiondemo on (google: 8.1.0) [usb] # android hooking watch class java.net.HttpURLConnection
(agent) Hooking java.net.HttpURLConnection.getFollowRedirects()
(agent) Hooking java.net.HttpURLConnection.setFollowRedirects(boolean)
(agent) Hooking java.net.HttpURLConnection.disconnect()
(agent) Hooking java.net.HttpURLConnection.getErrorStream()
(agent) Hooking java.net.HttpURLConnection.getHeaderField(int)
(agent) Hooking java.net.HttpURLConnection.getHeaderFieldDate(java.lang.String, long)
(agent) Hooking java.net.HttpURLConnection.getHeaderFieldKey(int)
(agent) Hooking java.net.HttpURLConnection.getInstanceFollowRedirects()
(agent) Hooking java.net.HttpURLConnection.getPermission()
(agent) Hooking java.net.HttpURLConnection.getRequestMethod()
(agent) Hooking java.net.HttpURLConnection.getResponseCode()
(agent) Hooking java.net.HttpURLConnection.getResponseMessage()
(agent) Hooking java.net.HttpURLConnection.setChunkedStreamingMode(int)
(agent) Hooking java.net.HttpURLConnection.setFixedLengthStreamingMode(int)

```

```
[574683] Called java.net.HttpURLConnection.getFollowRedirects()
(agent) [118392] Called java.net.HttpURLConnection.HttpURLConnection(java.net.URL)

```

我们可以发现只有构造函数和一个java.net.HttpURLConnection.getFollowRedirects()函数被调用了。



通过如下命令获取HttpURLConnection的实例时发现布存在实例

```
android heap search instances java.net.HttpURLConnection
```

可以在google官方api网站上发现，该类实际上是一个抽象类，在开发中可以直接使用一个抽象类去表示，但是在运行过程中是该抽象类的具体实现类在进行工作，所以我们需要确定具体实现类：

- 第一种方法是纯逆向方法，通过之前的源代码，我们可以发现第一次出现HttpURLConnection的定义是通过URL类的openConnection()函数完成的，这个函数的返回值的类名是HttpURLConnection()函数的具体实现类

```js
function main(){
    Java.perform(function(){
        var URL = Java.use('java.net.URL')
        URL.openConnection.overload().implementation = function(){
            //console.log('url =>',urlstr)
            var result = this.openConnection()
            console.log('openConnection() returnType =>',result.$className)
            return result
        }
    })
}

setImmediate(main)
```

```
(base) ┌──(root㉿kali)-[/home/zhy/桌面/chap8]
└─# frida -U -l HttpURLConnection.js HttpUrlConnectionDemo

____

​    / _  |   Frida 15.2.2 - A world-class dynamic instrumentation toolkit
   | (_| |
​    > _  |   Commands:
   /_/ |_|       help      -> Displays the help system
   . . . .       object?   -> Display information about 'object'
   . . . .       exit/quit -> Exit
   . . . .
   . . . .   More info at https://frida.re/docs/home/
   . . . .
   . . . .   Connected to Nexus 5X (id=00e310eb64541e43)
​                                                                                
[Nexus 5X::HttpUrlConnectionDemo ]-> openConnection() returnType => com.android.okhttp.internal.huc.HttpsURLConnectionImpl
openConnection() returnType => com.android.okhttp.internal.huc.HttpsURLConnectionImpl
openConnection() returnType => com.android.okhttp.internal.huc.HttpsURLConnectionImpl
openConnection() returnType => com.android.okhttp.internal.huc.HttpsURLConnectionImpl
```

- 由于我们手中有源代码，我们可以在对应的位置打断点，然后在AndroidStudio中进行调试

最终获取参数的自吐脚本如下：

```js
function main(){
    Java.perform(function(){
        var HttpURLConnectionImpl = Java.use('com.android.okhttp.internal.huc.HttpURLConnectionImpl')
        HttpURLConnectionImpl.setRequestProperty.implementation = function(key,value){
            //console.log('url =>',urlstr)
            var result = this.setRequestProperty(key,value)
            console.log('setRequestProperty=>',key,':',value)
            return result
        }
    })
}

setImmediate(main)
```

```
(base) ┌──(root㉿kali)-[/home/zhy/桌面/chap8]
└─# frida -U -l HttpURLConnection.js HttpUrlConnectionDemo
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
                                                                                
[Nexus 5X::HttpUrlConnectionDemo ]-> setRequestProperty=> token : zhy
setRequestProperty=> token : zhy
setRequestProperty=> token : zhy
setRequestProperty=> token : zhy
setRequestProperty=> token : zhy
setRequestProperty=> token : zhy
setRequestProperty=> token : zhy

```

