# Android抓包详解

## 抓包介绍

在Android APP的逆向分析中，抓包通常是指通过一些手段获取APP与服务器之间传输的铭文网络数据信息，这些网络数据信息往往是逆向分析的切入点，通过抓包得到的信息可以快速定位关键接口函数的位置，为调试代码逻辑提供了便利。

抓包主要分为两种：

- Hook抓包：Hook抓包实际上是指通过对发包函数的Hook来达到抓包的作用。
- 中间人抓包：指将原来一段完整的客户端-服务器通信方式割裂成两段客户端-服务器的通信。中间人的抓包在OSI七层网体模型的结构通常会被分成以下两种情况
  - 应用层：Http(s)协议抓包
  - 会话层：Socket通信抓包

## 工具推荐

Burpsuite：应用层Http(s)协议数据抓包

Charles：简单抓包，用的舒服轻松。

不推荐Fidder（因为Fidder无法导入客户端整数（p12,Client SSL Certificates））

会话层抓包推荐：

Charles

tcpdump+WireShark

## Http(S)协议抓包配置

### Http抓包配置

首先将计算机和测试手机连接在同一个局域网中，并且确保手机和计算机可以相互访问。

由于测试主机是虚拟机，首先关闭虚拟机，然后再虚拟机设置中把网络适配器的网络连接方式设置为桥接模式，需要注意的是

[(73条消息) kali虚拟机如何使用桥接模式连接外网_Cold_L i的博客-CSDN博客_kali桥接](https://blog.csdn.net/qq_40317852/article/details/120381805)

由于按照上述配置桥接后会导致物理机连不上网

最后决定从windows搭建抓包环境

电脑打开移动热点，用手机连接电脑的移动热点。

通过电脑ping通手机后

[Charles手机抓包 - 简书 (jianshu.com)](https://www.jianshu.com/p/551711c121f0)

再proxy setting中设置好端口

如果要对HTTPS抓包的话，还需要设置Proxy -> SSL Proxying Settings -> SSL Proxying -> Add，添加所有的域名和端口

点击Help再打开SSL Proxying 然后点击install Charles Root Certificates on aMobile Device or Remote Browser可以看到手机端安装证书提示。

长按连接网络，根据提示设置手动代理，在手机浏览器中访问chls.pro/ssl，然后下载证书，安装好证书，就可以抓包了。

仅仅将证书安装为用户信任的证书还不够我们可以通过以下命令将证书修改成系统自带的证书

```sh
C:\Users\33551>adb shell
bullhead:/ $ su

bullhead:/ # cd data/misc/user/0/cacerts-added/
bullhead:/data/misc/user/0/cacerts-added # ls
068ea8b5.0
bullhead:/data/misc/user/0/cacerts-added # mount -o remount,rw /system

1|bullhead:/data/misc/user/0/cacerts-added # cp * /etc/security/cacerts/

bullhead:/data/misc/user/0/cacerts-added # chmod 777 /etc/security/cacerts/*
bullhead:/data/misc/user/0/cacerts-added # mount -o remount,ro /system
bullhead:/data/misc/user/0/cacerts-added # reboot
```

为了配合手机上的代理设置，我们还需要在Charles上进行代理 设置。在依次单击Proxy→Proxy Settings→Enable SOCKS proxy 后单击OK按钮，就完成了SOCKS5模式的代理配置。此时重新启动 Charles的抓包，会发现比原先HTTP模式抓的包更多

## 应用层抓包核心原理

HTTPS协议的整个通信过程主要分成发起请求、验证身份、协商密 钥、加密通信阶段

![](https://github.com/G-WS/Android-reverse/blob/main/image/Https%E9%80%9A%E8%AE%AF%E8%BF%87%E7%A8%8B.png?raw=true)

首先客户端向服务器发起访问请求，服务器接收到客户端的请求后，便将服务器使用的证书公钥发给客户端，客户端通过本地配置的信任证书与服务器端传输的公钥进行比对，其中证书文件由第三方可信任机构颁发，此时，如果客户端校验是爱则会提示公钥不被信任，即会提示“您的连接不是私密连接”，如果客户端校验成功则会将客户端只用的证书公钥使用服务器传输的公钥加密后发送给服务器。服务器在接收到加密的客户端公钥后，会使用自己的私钥解密数据获取客户端的公钥进行加密后传输给客户端作为一段会话的标志。这就是HTTPS通信协议的密钥协商过程。后面通过sessionkey对通信数据进行加密。

> HTTPS=HTTP+加密+认证+完整性保护



在HTTPS上的应用层抓包原理也是基于中间人攻击的方式。 HTTPS上的应用层抓包原理主要“攻破”的是HTTPS传输过程中验 证身份的步骤，我们在配置抓包环境时是将Charles证书加入到系统 本身信任的证书中，当应用进行通信时，如果没有进一步的安全保护 措施，那么客户端接收到的服务器证书即使是Charles证书也会继续通信，整个过程可以表示如下：

![](https://github.com/G-WS/Android-reverse/blob/main/image/HTTPS%E4%B8%AD%E9%97%B4%E4%BA%BA%E6%8A%93%E5%8C%85.png?raw=true)

应对通过手动给系统安装证书从而导致中间人攻击继续生效的风险，App 也对这类攻击推出了对抗手段，主要有以下两种方式：

-  SSL Pinning，又称证书绑定，可以说是客户端校验服务器的进 阶版：该种方式不仅校验服务器证书是否是系统中的可信凭证，在通信过程中甚至连系统内置的证书都不信任而只信任App 指定的证书。一旦发现服务器证书为非指定证书即停止通信， 最终导致即使将Charles证书安装到系统信任凭据中也无法生效。 
- 服务器校验客户端。这种方式发生在HTTPS验证身份阶段，服务 器在接收到客户端的公钥后，在发送session key之前先对客户端的公钥进行验证，如果不是信任的证书公钥，服务器就中止 和客户端的通信。

逆向开发和分析人员对这些App对抗抓包 的手段也有了一些绕过的方式，比如在应对客户端校验服务器的情况 时，考虑到对证书校验的代码是写于App内部的，自然而然地就可以 通过Hook修改校验服务器的代码，从而使得判断的机制失效。在这 方面Objection本身可以通过以下命令完成SSL Pinning Bypass的功能：

```sh
android sslpinning disable
```

开源的项目DroidSSLUnpinning中添加了一部分Objection框架中所没有的Bypass证书校验，项目地址如下：

[WooyunDota/DroidSSLUnpinning: Android certificate pinning disable tools (github.com)](https://github.com/WooyunDota/DroidSSLUnpinning)

> 由于SSL Pinning的功能是由开发者自定义的，因此并不存在一 个通用的解决方案，Objection和DroidSSLUnpinning也只是对常见 的App所使用的网络框架中对证书进行校验的代码逻辑进行了Hook修 改。一旦App中的代码被混淆或者使用了未知的框架，这些App的客 户端校验服务器的逻辑就需要安全人员自行分析，不过上述两种方案 已几乎可以覆盖目前已知的所有种类的证书绑定

在应对服务器校验客户端的对抗手段中，服务器并不掌握在分析 人员手中，因此在中间人的状态下与服务器进行通信的实际上已经变 成抓包软件，比如Charles。通常来说，我们所能做的对抗手段就是 将App中内置的证书导入Charles中，使得服务端认为自己仍旧是在 与其信任的客户端进行通信，最终达到欺骗服务器的作用。 具体在操作过程中需要完成两项工作：第一，找到证书文件和相 应的证书密码（这个过程是需要逆向分析的，后续会以案例进行详细 的介绍）；第二，在找到证书和密码后将其导入抓包软件中，比如导 入 Charles 。 打 开 Charles ， 依 次 单 击 Proxy→SSL Proxy Settings→Client Certificates→Add，添加新的证书，输入指定的域名IP以及端口并导入p12格式或者pem格式的这个证书，之后即可将Charles伪装成使用特定证书的客户 端，最终达到正常抓包的目的。

## Hook模拟抓包

使用Hook方式来抓包，所面对的就是直接的代码。相应 地，在Android的Java世界中，其实就是面对类及其函数。

目标是使用Objection对一个发生网络通信的App进行Hook抓包。

Hook目标为移动TV.APK

首先准备好Hook环境，切换到~/.objection目录下，删除旧的objection.log，以保证Objection运行的时候的日志只有本次的测试APP的内容，启动APP进行，运行以下命令获取APP已经加载的所有类。

```
(base) ┌──(root㉿kali)-[/home/zhy]
└─# cd ~/.objection/
                                                                                                                                    
(base) ┌──(root㉿kali)-[~/.objection]
└─# rm objection.log                         
                                                                                                                                    
(base) ┌──(root㉿kali)-[~/.objection]
└─# frida-ps -U          
 PID  Name
----  ---------------------------------------------------
3120  ATFWD-daemon                                       
8744  Android Pay                                        
8295  F-Droid                                            
8778  Gmail                                              
5598  Google                                             
8917  Google Play 商店                                     
9141  TIM                                                
7245  adbd                                               
3126  android.hardware.biometrics.fingerprint@2.1-service
 436  android.hardware.cas@1.0-service                   
 437  android.hardware.configstore@1.0-service           
 438  android.hardware.dumpstate@1.0-service.bullhead    
 439  android.hardware.graphics.allocator@2.0-service    
3114  android.hardware.media.omx@1.0-service             
 440  android.hardware.usb@1.0-service                   
 441  android.hardware.wifi@1.0-service                  
 435  android.hidl.allocator@1.0-service                 
5794  android.process.media                              
3102  audioserver                                        
3103  cameraserver                                       
3121  cnd                                                
3117  cnss-daemon                                        
3797  com.android.bluetooth                              
5639  com.android.nfc                                    
4052  com.android.phone                                  
7020  com.android.providers.calendar                     
3839  com.android.systemui                               
8865  com.google.android.apps.gcs                        
5980  com.google.android.gms                             
5556  com.google.android.gms.persistent                  
9030  com.google.android.gms.unstable                    
8997  com.google.android.gms:car                         
5735  com.google.android.googlequicksearchbox            
8830  com.google.android.googlequicksearchbox:search     
7413  com.google.android.ims                             
6038  com.google.android.inputmethod.latin               
7802  com.google.process.gapps                           
5752  com.google.process.gservices                       
8354  com.qualcomm.qcrilmsgtunnel                        
5653  com.qualcomm.qti.rcsbootstraputil                  
5664  com.qualcomm.qti.rcsimsbootstraputil               
8334  com.qualcomm.telephony                             
5626  com.quicinc.cne.CNEService                         
9162  com.tencent.tim                                    
7040  daemonsu:10102                                     
3075  daemonsu:master                                    
7068  daemonsu:mount:0                                   
3068  daemonsu:mount:master                              
3104  drmserver                                          
4160  entropy.bin                                        
3124  gatekeeper                                         
 442  healthd                                            
 369  hwservicemanager                                   
3145  imsdatadaemon                                      
3099  imsqmidaemon                                       
   1  init                                               
3105  installd                                           
3153  ip6tables-restore                                  
3152  iptables-restore                                   
3106  keystore                                           
 446  lmkd                                               
3116  loc_launcher                                       
3149  location-mq                                        
 367  logd                                               
3150  lowi-server                                        
3108  media.extractor                                    
3109  media.metrics                                      
3107  mediadrmserver                                     
3110  mediaserver                                        
3119  mm-qcamera-daemon                                  
 445  msm_irqbalance.conf                                
3111  netd                                               
3097  netmgrd                                            
3095  perfd                                              
 500  pm-proxy                                           
 444  pm-service                                         
3094  qmuxd                                              
 371  qseecomd                                           
 380  qseecomd                                           
3096  qti                                                
3115  rild                                               
 443  rmt_storage                                        
 368  servicemanager                                     
9554  sh                                                 
3151  slim_daemon                                        
7293  sshd [listener] 0 of 10-100 startups               
3112  storaged                                           
 447  surfaceflinger                                     
3518  system_server                                      
3098  thermal-engine                                     
 448  thermalserviced                                    
3118  time_daemon                                        
3125  tombstoned                                         
 348  ueventd                                            
 370  vndbinder                                          
 400  vold                                               
4327  wcnss_filter                                       
3880  webview_zygote32                                   
3113  wificond                                           
3101  zygote                                             
3100  zygote64                                           
7362  信息                                                 
8686  地圖                                                 
7528  日历                                                 
9506  移动TV                                               
4077  设置            
```

可以看到移动TVAPP的进程

```
(base) ┌──(root㉿kali)-[~/.objection]
└─# objection -g 移动TV explore
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
com.cz.babySister on (google: 8.1.0) [usb] # android hooking list classes
[B
[C
[D
[F
[I
[J
[Landroid.animation.Keyframe$FloatKeyframe;
[Landroid.animation.PropertyValuesHolder;
[Landroid.app.LoaderManagerImpl;
[Landroid.app.assist.AssistStructure$ViewNode;
[Landroid.content.UndoOwner;
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
[Landroid.graphics.Bitmap;
[Landroid.graphics.Canvas$EdgeType;
[Landroid.graphics.ColorSpace$Model;
[Landroid.graphics.ColorSpace$Named;
[Landroid.graphics.ColorSpace;
[Landroid.graphics.FontFamily;
[Landroid.graphics.Matrix$ScaleToFit;
[Landroid.graphics.Paint$Align;
[Landroid.graphics.Paint$Cap;
[Landroid.graphics.Paint$Join;
[Landroid.graphics.Paint$Style;
[Landroid.graphics.Path$FillType;
[Landroid.graphics.Point;
[Landroid.graphics.PorterDuff$Mode;
[Landroid.graphics.Rect;
[Landroid.graphics.Region$Op;
[Landroid.graphics.Shader$TileMode;
[Landroid.graphics.Typeface;
[Landroid.graphics.drawable.Drawable;
[Landroid.graphics.drawable.GradientDrawable$Orientation;
[Landroid.graphics.drawable.LayerDrawable$ChildDrawable;
[Landroid.graphics.fonts.FontVariationAxis;
[Landroid.hardware.camera2.params.Face;
[Landroid.hardware.camera2.params.HighSpeedVideoConfiguration;
[Landroid.hardware.camera2.params.MeteringRectangle;
[Landroid.hardware.camera2.params.StreamConfiguration;
[Landroid.hardware.camera2.params.StreamConfigurationDuration;
[Landroid.hardware.soundtrigger.SoundTrigger$ConfidenceLevel;
[Landroid.hardware.soundtrigger.SoundTrigger$Keyphrase;
[Landroid.hardware.soundtrigger.SoundTrigger$KeyphraseRecognitionExtra;
[Landroid.icu.impl.CacheValue$Strength;
[Landroid.icu.impl.CacheValue;
[Landroid.icu.impl.CurrencyData$CurrencySpacingInfo$SpacingPattern;
[Landroid.icu.impl.CurrencyData$CurrencySpacingInfo$SpacingType;
[Landroid.icu.impl.ICUResourceBundle$OpenType;
[Landroid.icu.impl.TimeZoneNamesImpl$ZNames$NameTypeIndex;
..............
```

退出Objection，然后将objection.log文件重命名为objectionTest.log并保存到其他目录。通过cat和grep过滤网络框架。

```
(base) ┌──(root㉿kali)-[~/.objection]
└─# mv objection.log objectionTest.log
                                                                                                                                    
(base) ┌──(root㉿kali)-[~/.objection]
└─# ls
objection_history  objection_old.log  objectionTest.log  plugins  version_info
                                                                                                                                    
(base) ┌──(root㉿kali)-[~/.objection]
└─# cat objectionTest.log |grep -i HttpURLConnection
com.android.okhttp.internal.huc.HttpURLConnectionImpl
java.net.HttpURLConnection
                                                                                                                                    
(base) ┌──(root㉿kali)-[~/.objection]
└─# cat objectionTest.log |grep -i okhttp3          
                                                                                                                                    
(base) ┌──(root㉿kali)-[~/.objection]
└─# cat objectionTest.log |grep -i okhttp 
[Lcom.android.okhttp.CipherSuite;
[Lcom.android.okhttp.ConnectionSpec;
[Lcom.android.okhttp.HttpUrl$Builder$ParseResult;
[Lcom.android.okhttp.Protocol;
[Lcom.android.okhttp.TlsVersion;
com.android.okhttp.Address
com.android.okhttp.Authenticator
com.android.okhttp.CacheControl
com.android.okhttp.CacheControl$Builder
com.android.okhttp.CertificatePinner
com.android.okhttp.CertificatePinner$Builder
com.android.okhttp.CipherSuite
com.android.okhttp.ConfigAwareConnectionPool
com.android.okhttp.ConfigAwareConnectionPool$1
com.android.okhttp.Connection
com.android.okhttp.ConnectionPool
com.android.okhttp.ConnectionPool$1
com.android.okhttp.ConnectionSpec
com.android.okhttp.ConnectionSpec$Builder
com.android.okhttp.Dispatcher
com.android.okhttp.Dns
com.android.okhttp.Dns$1
com.android.okhttp.Handshake
com.android.okhttp.Headers
com.android.okhttp.Headers$Builder
com.android.okhttp.HttpHandler
com.android.okhttp.HttpHandler$CleartextURLFilter
com.android.okhttp.HttpUrl
com.android.okhttp.HttpUrl$Builder
com.android.okhttp.HttpUrl$Builder$ParseResult
com.android.okhttp.HttpsHandler
com.android.okhttp.OkHttpClient
com.android.okhttp.OkHttpClient$1
com.android.okhttp.OkUrlFactory
com.android.okhttp.Protocol
com.android.okhttp.Request
com.android.okhttp.Request$Builder
com.android.okhttp.RequestBody
com.android.okhttp.RequestBody$2
com.android.okhttp.Response
com.android.okhttp.Response$Builder
com.android.okhttp.ResponseBody
com.android.okhttp.Route
com.android.okhttp.TlsVersion
com.android.okhttp.internal.ConnectionSpecSelector
com.android.okhttp.internal.Internal
com.android.okhttp.internal.OptionalMethod
com.android.okhttp.internal.RouteDatabase
com.android.okhttp.internal.URLFilter
com.android.okhttp.internal.Util
com.android.okhttp.internal.Util$1
com.android.okhttp.internal.http.AuthenticatorAdapter
com.android.okhttp.internal.http.CacheStrategy
com.android.okhttp.internal.http.CacheStrategy$Factory
com.android.okhttp.internal.http.Http1xStream
com.android.okhttp.internal.http.Http1xStream$AbstractSource
com.android.okhttp.internal.http.Http1xStream$ChunkedSource
com.android.okhttp.internal.http.Http1xStream$FixedLengthSource
com.android.okhttp.internal.http.HttpEngine
com.android.okhttp.internal.http.HttpEngine$1
com.android.okhttp.internal.http.HttpMethod
com.android.okhttp.internal.http.HttpStream
com.android.okhttp.internal.http.OkHeaders$1
com.android.okhttp.internal.http.RealResponseBody
com.android.okhttp.internal.http.RequestException
com.android.okhttp.internal.http.RequestLine
com.android.okhttp.internal.http.RetryableSink
com.android.okhttp.internal.http.RouteException
com.android.okhttp.internal.http.RouteSelector
com.android.okhttp.internal.http.StatusLine
com.android.okhttp.internal.http.StreamAllocation
com.android.okhttp.internal.huc.DelegatingHttpsURLConnection
com.android.okhttp.internal.huc.HttpURLConnectionImpl
com.android.okhttp.internal.huc.HttpsURLConnectionImpl
com.android.okhttp.internal.tls.OkHostnameVerifier
com.android.okhttp.okio.AsyncTimeout
com.android.okhttp.okio.AsyncTimeout$1
com.android.okhttp.okio.AsyncTimeout$2
com.android.okhttp.okio.AsyncTimeout$Watchdog
com.android.okhttp.okio.BufferedSink
com.android.okhttp.okio.BufferedSource
com.android.okhttp.okio.ForwardingTimeout
com.android.okhttp.okio.GzipSource
com.android.okhttp.okio.InflaterSource
com.android.okhttp.okio.Okio$1
com.android.okhttp.okio.Okio$2
com.android.okhttp.okio.Okio$3
com.android.okhttp.okio.RealBufferedSink
com.android.okhttp.okio.RealBufferedSink$1
com.android.okhttp.okio.RealBufferedSource
com.android.okhttp.okio.RealBufferedSource$1
com.android.okhttp.okio.Segment
com.android.okhttp.okio.Sink
com.android.okhttp.okio.Source
com.android.okhttp.okio.Timeout
com.android.okhttp.okio.Timeout$1

```

利用Objection中的-c参数 执行指定文件中所有的Objection命令，通过这种近乎是trace的功能 对第三步过滤的所有关键类进行Hook，进而顺利定位关键通信函数。

将第三步过滤出来的关键类保存到文件中，然后通过一些文本编辑器可以快速输入多行来补全命令，通过vscode打开该文件，按下alt shift 鼠标的方式对每一行行首输入 android hooking watch class然后即可对所有命令前补全。

```sh
(base) ┌──(root㉿kali)-[/home/zhy/桌面]
└─# objection -g 移动TV explore -c "1.txt"
Using USB device `Nexus 5X`
Agent injected and responds ok!
Running commands from file...
Running: 'android hooking watch class com.android.okhttp.Address':

(agent) Hooking com.android.okhttp.Address.equals(java.lang.Object)
(agent) Hooking com.android.okhttp.Address.getAuthenticator()
(agent) Hooking com.android.okhttp.Address.getCertificatePinner()
(agent) Hooking com.android.okhttp.Address.getConnectionSpecs()
(agent) Hooking com.android.okhttp.Address.getDns()
(agent) Hooking com.android.okhttp.Address.getHostnameVerifier()
(agent) Hooking com.android.okhttp.Address.getProtocols()
(agent) Hooking com.android.okhttp.Address.getProxy()
(agent) Hooking com.android.okhttp.Address.getProxySelector()
(agent) Hooking com.android.okhttp.Address.getSocketFactory()
(agent) Hooking com.android.okhttp.Address.getSslSocketFactory()
(agent) Hooking com.android.okhttp.Address.getUriHost()
(agent) Hooking com.android.okhttp.Address.getUriPort()
(agent) Hooking com.android.okhttp.Address.hashCode()
(agent) Hooking com.android.okhttp.Address.url()
(agent) Registering job 695246. Type: watch-class for: com.android.okhttp.Address
Running: 'android hooking watch class com.android.okhttp.Authenticator':

(agent) Hooking com.android.okhttp.Authenticator.authenticate(java.net.Proxy, com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Authenticator.authenticateProxy(java.net.Proxy, com.android.okhttp.Response)
(agent) Registering job 287335. Type: watch-class for: com.android.okhttp.Authenticator
Running: 'android hooking watch class com.android.okhttp.CacheControl':

(agent) Hooking com.android.okhttp.CacheControl.headerValue()
(agent) Hooking com.android.okhttp.CacheControl.parse(com.android.okhttp.Headers)
(agent) Hooking com.android.okhttp.CacheControl.isPrivate()
(agent) Hooking com.android.okhttp.CacheControl.isPublic()
(agent) Hooking com.android.okhttp.CacheControl.maxAgeSeconds()
(agent) Hooking com.android.okhttp.CacheControl.maxStaleSeconds()
(agent) Hooking com.android.okhttp.CacheControl.minFreshSeconds()
(agent) Hooking com.android.okhttp.CacheControl.mustRevalidate()
(agent) Hooking com.android.okhttp.CacheControl.noCache()
(agent) Hooking com.android.okhttp.CacheControl.noStore()
(agent) Hooking com.android.okhttp.CacheControl.noTransform()
(agent) Hooking com.android.okhttp.CacheControl.onlyIfCached()
(agent) Hooking com.android.okhttp.CacheControl.sMaxAgeSeconds()
(agent) Hooking com.android.okhttp.CacheControl.toString()
(agent) Registering job 875765. Type: watch-class for: com.android.okhttp.CacheControl
Running: 'android hooking watch class com.android.okhttp.CacheControl$Builder':

(agent) Hooking com.android.okhttp.CacheControl$Builder.build()
(agent) Hooking com.android.okhttp.CacheControl$Builder.maxAge(int, java.util.concurrent.TimeUnit)
(agent) Hooking com.android.okhttp.CacheControl$Builder.maxStale(int, java.util.concurrent.TimeUnit)
(agent) Hooking com.android.okhttp.CacheControl$Builder.minFresh(int, java.util.concurrent.TimeUnit)
(agent) Hooking com.android.okhttp.CacheControl$Builder.noCache()
(agent) Hooking com.android.okhttp.CacheControl$Builder.noStore()
(agent) Hooking com.android.okhttp.CacheControl$Builder.noTransform()
(agent) Hooking com.android.okhttp.CacheControl$Builder.onlyIfCached()
(agent) Registering job 565857. Type: watch-class for: com.android.okhttp.CacheControl$Builder
Running: 'android hooking watch class com.android.okhttp.CertificatePinner':

(agent) Hooking com.android.okhttp.CertificatePinner.pin(java.security.cert.Certificate)
(agent) Hooking com.android.okhttp.CertificatePinner.sha1(java.security.cert.X509Certificate)
(agent) Hooking com.android.okhttp.CertificatePinner.check(java.lang.String, java.util.List)
(agent) Hooking com.android.okhttp.CertificatePinner.check(java.lang.String, [Ljava.security.cert.Certificate;)
(agent) Hooking com.android.okhttp.CertificatePinner.findMatchingPins(java.lang.String)
(agent) Registering job 091902. Type: watch-class for: com.android.okhttp.CertificatePinner
Running: 'android hooking watch class com.android.okhttp.CertificatePinner$Builder':

(agent) Hooking com.android.okhttp.CertificatePinner$Builder.-get0(com.android.okhttp.CertificatePinner$Builder)
(agent) Hooking com.android.okhttp.CertificatePinner$Builder.add(java.lang.String, [Ljava.lang.String;)
(agent) Hooking com.android.okhttp.CertificatePinner$Builder.build()
(agent) Registering job 513586. Type: watch-class for: com.android.okhttp.CertificatePinner$Builder
Running: 'android hooking watch class com.android.okhttp.CipherSuite':

(agent) Hooking com.android.okhttp.CipherSuite.forJavaName(java.lang.String)
(agent) Hooking com.android.okhttp.CipherSuite.valueOf(java.lang.String)
(agent) Hooking com.android.okhttp.CipherSuite.valueOf()
(agent) Hooking com.android.okhttp.CipherSuite.values()
(agent) Registering job 624453. Type: watch-class for: com.android.okhttp.CipherSuite
Running: 'android hooking watch class com.android.okhttp.ConfigAwareConnectionPool':

(agent) Hooking com.android.okhttp.ConfigAwareConnectionPool.-set0(com.android.okhttp.ConfigAwareConnectionPool, com.android.okhttp.ConnectionPool)
(agent) Hooking com.android.okhttp.ConfigAwareConnectionPool.getInstance()
(agent) Hooking com.android.okhttp.ConfigAwareConnectionPool.get()
(agent) Registering job 508942. Type: watch-class for: com.android.okhttp.ConfigAwareConnectionPool
Running: 'android hooking watch class com.android.okhttp.ConfigAwareConnectionPool$1':

(agent) Hooking com.android.okhttp.ConfigAwareConnectionPool$1.onNetworkConfigurationChanged()
(agent) Registering job 717254. Type: watch-class for: com.android.okhttp.ConfigAwareConnectionPool$1
Running: 'android hooking watch class com.android.okhttp.Connection':

(agent) Hooking com.android.okhttp.Connection.getHandshake()
(agent) Hooking com.android.okhttp.Connection.getProtocol()
(agent) Hooking com.android.okhttp.Connection.getRoute()
(agent) Hooking com.android.okhttp.Connection.getSocket()
(agent) Registering job 941423. Type: watch-class for: com.android.okhttp.Connection
Running: 'android hooking watch class com.android.okhttp.ConnectionPool':

(agent) Hooking com.android.okhttp.ConnectionPool.getDefault()
(agent) Hooking com.android.okhttp.ConnectionPool.pruneAndGetAllocationCount(com.android.okhttp.internal.io.RealConnection, long)
(agent) Hooking com.android.okhttp.ConnectionPool.cleanup(long)
(agent) Hooking com.android.okhttp.ConnectionPool.connectionBecameIdle(com.android.okhttp.internal.io.RealConnection)
(agent) Hooking com.android.okhttp.ConnectionPool.evictAll()
(agent) Hooking com.android.okhttp.ConnectionPool.get(com.android.okhttp.Address, com.android.okhttp.internal.http.StreamAllocation)
(agent) Hooking com.android.okhttp.ConnectionPool.getConnectionCount()
(agent) Hooking com.android.okhttp.ConnectionPool.getHttpConnectionCount()
(agent) Hooking com.android.okhttp.ConnectionPool.getIdleConnectionCount()
(agent) Hooking com.android.okhttp.ConnectionPool.getMultiplexedConnectionCount()
(agent) Hooking com.android.okhttp.ConnectionPool.getSpdyConnectionCount()
(agent) Hooking com.android.okhttp.ConnectionPool.put(com.android.okhttp.internal.io.RealConnection)
(agent) Hooking com.android.okhttp.ConnectionPool.setCleanupRunnableForTest(java.lang.Runnable)
(agent) Registering job 255176. Type: watch-class for: com.android.okhttp.ConnectionPool
Running: 'android hooking watch class com.android.okhttp.ConnectionPool$1':

(agent) Hooking com.android.okhttp.ConnectionPool$1.run()
(agent) Registering job 635787. Type: watch-class for: com.android.okhttp.ConnectionPool$1
Running: 'android hooking watch class com.android.okhttp.ConnectionSpec':

(agent) Hooking com.android.okhttp.ConnectionSpec.-get0(com.android.okhttp.ConnectionSpec)
(agent) Hooking com.android.okhttp.ConnectionSpec.-get1(com.android.okhttp.ConnectionSpec)
(agent) Hooking com.android.okhttp.ConnectionSpec.-get2(com.android.okhttp.ConnectionSpec)
(agent) Hooking com.android.okhttp.ConnectionSpec.-get3(com.android.okhttp.ConnectionSpec)
(agent) Hooking com.android.okhttp.ConnectionSpec.nonEmptyIntersection([Ljava.lang.String;, [Ljava.lang.String;)
(agent) Hooking com.android.okhttp.ConnectionSpec.supportedSpec(javax.net.ssl.SSLSocket, boolean)
(agent) Hooking com.android.okhttp.ConnectionSpec.apply(javax.net.ssl.SSLSocket, boolean)
(agent) Hooking com.android.okhttp.ConnectionSpec.cipherSuites()
(agent) Hooking com.android.okhttp.ConnectionSpec.equals(java.lang.Object)
(agent) Hooking com.android.okhttp.ConnectionSpec.hashCode()
(agent) Hooking com.android.okhttp.ConnectionSpec.isCompatible(javax.net.ssl.SSLSocket)
(agent) Hooking com.android.okhttp.ConnectionSpec.isTls()
(agent) Hooking com.android.okhttp.ConnectionSpec.supportsTlsExtensions()
(agent) Hooking com.android.okhttp.ConnectionSpec.tlsVersions()
(agent) Hooking com.android.okhttp.ConnectionSpec.toString()
(agent) Registering job 851414. Type: watch-class for: com.android.okhttp.ConnectionSpec
Running: 'android hooking watch class com.android.okhttp.ConnectionSpec$Builder':

(agent) Hooking com.android.okhttp.ConnectionSpec$Builder.-get0(com.android.okhttp.ConnectionSpec$Builder)
(agent) Hooking com.android.okhttp.ConnectionSpec$Builder.-get1(com.android.okhttp.ConnectionSpec$Builder)
(agent) Hooking com.android.okhttp.ConnectionSpec$Builder.-get2(com.android.okhttp.ConnectionSpec$Builder)
(agent) Hooking com.android.okhttp.ConnectionSpec$Builder.-get3(com.android.okhttp.ConnectionSpec$Builder)
(agent) Hooking com.android.okhttp.ConnectionSpec$Builder.allEnabledCipherSuites()
(agent) Hooking com.android.okhttp.ConnectionSpec$Builder.allEnabledTlsVersions()
(agent) Hooking com.android.okhttp.ConnectionSpec$Builder.build()
(agent) Hooking com.android.okhttp.ConnectionSpec$Builder.cipherSuites([Lcom.android.okhttp.CipherSuite;)
(agent) Hooking com.android.okhttp.ConnectionSpec$Builder.cipherSuites([Ljava.lang.String;)
(agent) Hooking com.android.okhttp.ConnectionSpec$Builder.supportsTlsExtensions(boolean)
(agent) Hooking com.android.okhttp.ConnectionSpec$Builder.tlsVersions([Lcom.android.okhttp.TlsVersion;)
(agent) Hooking com.android.okhttp.ConnectionSpec$Builder.tlsVersions([Ljava.lang.String;)
(agent) Registering job 346111. Type: watch-class for: com.android.okhttp.ConnectionSpec$Builder
Running: 'android hooking watch class com.android.okhttp.Dispatcher':

(agent) Hooking com.android.okhttp.Dispatcher.promoteCalls()
(agent) Hooking com.android.okhttp.Dispatcher.runningCallsForHost(com.android.okhttp.Call$AsyncCall)
(agent) Hooking com.android.okhttp.Dispatcher.cancel(java.lang.Object)
(agent) Hooking com.android.okhttp.Dispatcher.enqueue(com.android.okhttp.Call$AsyncCall)
(agent) Hooking com.android.okhttp.Dispatcher.executed(com.android.okhttp.Call)
(agent) Hooking com.android.okhttp.Dispatcher.finished(com.android.okhttp.Call$AsyncCall)
(agent) Hooking com.android.okhttp.Dispatcher.finished(com.android.okhttp.Call)
(agent) Hooking com.android.okhttp.Dispatcher.getExecutorService()
(agent) Hooking com.android.okhttp.Dispatcher.getMaxRequests()
(agent) Hooking com.android.okhttp.Dispatcher.getMaxRequestsPerHost()
(agent) Hooking com.android.okhttp.Dispatcher.getQueuedCallCount()
(agent) Hooking com.android.okhttp.Dispatcher.getRunningCallCount()
(agent) Hooking com.android.okhttp.Dispatcher.setMaxRequests(int)
(agent) Hooking com.android.okhttp.Dispatcher.setMaxRequestsPerHost(int)
(agent) Registering job 973716. Type: watch-class for: com.android.okhttp.Dispatcher
Running: 'android hooking watch class com.android.okhttp.Dns':

(agent) Hooking com.android.okhttp.Dns.lookup(java.lang.String)
(agent) Registering job 524935. Type: watch-class for: com.android.okhttp.Dns
Running: 'android hooking watch class com.android.okhttp.Dns$1':

(agent) Hooking com.android.okhttp.Dns$1.lookup(java.lang.String)
(agent) Registering job 360849. Type: watch-class for: com.android.okhttp.Dns$1
Running: 'android hooking watch class com.android.okhttp.Handshake':

(agent) Hooking com.android.okhttp.Handshake.get(java.lang.String, java.util.List, java.util.List)
(agent) Hooking com.android.okhttp.Handshake.get(javax.net.ssl.SSLSession)
(agent) Hooking com.android.okhttp.Handshake.cipherSuite()
(agent) Hooking com.android.okhttp.Handshake.equals(java.lang.Object)
(agent) Hooking com.android.okhttp.Handshake.hashCode()
(agent) Hooking com.android.okhttp.Handshake.localCertificates()
(agent) Hooking com.android.okhttp.Handshake.localPrincipal()
(agent) Hooking com.android.okhttp.Handshake.peerCertificates()
(agent) Hooking com.android.okhttp.Handshake.peerPrincipal()
(agent) Registering job 108014. Type: watch-class for: com.android.okhttp.Handshake
Running: 'android hooking watch class com.android.okhttp.Headers':

(agent) Hooking com.android.okhttp.Headers.get([Ljava.lang.String;, java.lang.String)
(agent) Hooking com.android.okhttp.Headers.get(java.lang.String)
(agent) Hooking com.android.okhttp.Headers.of(java.util.Map)
(agent) Hooking com.android.okhttp.Headers.of([Ljava.lang.String;)
(agent) Hooking com.android.okhttp.Headers.getDate(java.lang.String)
(agent) Hooking com.android.okhttp.Headers.name(int)
(agent) Hooking com.android.okhttp.Headers.names()
(agent) Hooking com.android.okhttp.Headers.newBuilder()
(agent) Hooking com.android.okhttp.Headers.size()
(agent) Hooking com.android.okhttp.Headers.toMultimap()
(agent) Hooking com.android.okhttp.Headers.toString()
(agent) Hooking com.android.okhttp.Headers.value(int)
(agent) Hooking com.android.okhttp.Headers.values(java.lang.String)
(agent) Registering job 237839. Type: watch-class for: com.android.okhttp.Headers
Running: 'android hooking watch class com.android.okhttp.Headers$Builder':

(agent) Hooking com.android.okhttp.Headers$Builder.-get0(com.android.okhttp.Headers$Builder)
(agent) Hooking com.android.okhttp.Headers$Builder.checkNameAndValue(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.Headers$Builder.add(java.lang.String)
(agent) Hooking com.android.okhttp.Headers$Builder.add(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.Headers$Builder.addLenient(java.lang.String)
(agent) Hooking com.android.okhttp.Headers$Builder.addLenient(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.Headers$Builder.build()
(agent) Hooking com.android.okhttp.Headers$Builder.get(java.lang.String)
(agent) Hooking com.android.okhttp.Headers$Builder.removeAll(java.lang.String)
(agent) Hooking com.android.okhttp.Headers$Builder.set(java.lang.String, java.lang.String)
(agent) Registering job 425143. Type: watch-class for: com.android.okhttp.Headers$Builder
Running: 'android hooking watch class com.android.okhttp.HttpHandler':

(agent) Hooking com.android.okhttp.HttpHandler.createHttpOkUrlFactory(java.net.Proxy)
(agent) Hooking com.android.okhttp.HttpHandler.getDefaultPort()
(agent) Hooking com.android.okhttp.HttpHandler.newOkUrlFactory(java.net.Proxy)
(agent) Hooking com.android.okhttp.HttpHandler.openConnection(java.net.URL)
(agent) Hooking com.android.okhttp.HttpHandler.openConnection(java.net.URL, java.net.Proxy)
(agent) Registering job 996967. Type: watch-class for: com.android.okhttp.HttpHandler
Running: 'android hooking watch class com.android.okhttp.HttpHandler$CleartextURLFilter':

(agent) Hooking com.android.okhttp.HttpHandler$CleartextURLFilter.checkURLPermitted(java.net.URL)
(agent) Registering job 671770. Type: watch-class for: com.android.okhttp.HttpHandler$CleartextURLFilter
Running: 'android hooking watch class com.android.okhttp.HttpUrl':

(agent) Hooking com.android.okhttp.HttpUrl.-get0(com.android.okhttp.HttpUrl)
(agent) Hooking com.android.okhttp.HttpUrl.-get1(com.android.okhttp.HttpUrl)
(agent) Hooking com.android.okhttp.HttpUrl.-get2(com.android.okhttp.HttpUrl)
(agent) Hooking com.android.okhttp.HttpUrl.-getcom-android-okhttp-HttpUrl$Builder$ParseResultSwitchesValues()
(agent) Hooking com.android.okhttp.HttpUrl.-wrap0(java.lang.String, int, int, java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl.canonicalize(java.lang.String, int, int, java.lang.String, boolean, boolean, boolean, boolean)
(agent) Hooking com.android.okhttp.HttpUrl.canonicalize(java.lang.String, java.lang.String, boolean, boolean, boolean, boolean)
(agent) Hooking com.android.okhttp.HttpUrl.canonicalize(com.android.okhttp.okio.Buffer, java.lang.String, int, int, java.lang.String, boolean, boolean, boolean, boolean)
(agent) Hooking com.android.okhttp.HttpUrl.decodeHexDigit(char)
(agent) Hooking com.android.okhttp.HttpUrl.defaultPort(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl.delimiterOffset(java.lang.String, int, int, java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl.get(java.net.URI)
(agent) Hooking com.android.okhttp.HttpUrl.get(java.net.URL)
(agent) Hooking com.android.okhttp.HttpUrl.getChecked(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl.namesAndValuesToQueryString(java.lang.StringBuilder, java.util.List)
(agent) Hooking com.android.okhttp.HttpUrl.parse(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl.pathSegmentsToString(java.lang.StringBuilder, java.util.List)
(agent) Hooking com.android.okhttp.HttpUrl.percentDecode(java.lang.String, int, int, boolean)
(agent) Hooking com.android.okhttp.HttpUrl.percentDecode(java.lang.String, boolean)
(agent) Hooking com.android.okhttp.HttpUrl.percentDecode(java.util.List, boolean)
(agent) Hooking com.android.okhttp.HttpUrl.percentDecode(com.android.okhttp.okio.Buffer, java.lang.String, int, int, boolean)
(agent) Hooking com.android.okhttp.HttpUrl.percentEncoded(java.lang.String, int, int)
(agent) Hooking com.android.okhttp.HttpUrl.queryStringToNamesAndValues(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl.encodedFragment()
(agent) Hooking com.android.okhttp.HttpUrl.encodedPassword()
(agent) Hooking com.android.okhttp.HttpUrl.encodedPath()
(agent) Hooking com.android.okhttp.HttpUrl.encodedPathSegments()
(agent) Hooking com.android.okhttp.HttpUrl.encodedQuery()
(agent) Hooking com.android.okhttp.HttpUrl.encodedUsername()
(agent) Hooking com.android.okhttp.HttpUrl.equals(java.lang.Object)
(agent) Hooking com.android.okhttp.HttpUrl.fragment()
(agent) Hooking com.android.okhttp.HttpUrl.hashCode()
(agent) Hooking com.android.okhttp.HttpUrl.host()
(agent) Hooking com.android.okhttp.HttpUrl.isHttps()
(agent) Hooking com.android.okhttp.HttpUrl.newBuilder()
(agent) Hooking com.android.okhttp.HttpUrl.password()
(agent) Hooking com.android.okhttp.HttpUrl.pathSegments()
(agent) Hooking com.android.okhttp.HttpUrl.pathSize()
(agent) Hooking com.android.okhttp.HttpUrl.port()
(agent) Hooking com.android.okhttp.HttpUrl.query()
(agent) Hooking com.android.okhttp.HttpUrl.queryParameter(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl.queryParameterName(int)
(agent) Hooking com.android.okhttp.HttpUrl.queryParameterNames()
(agent) Hooking com.android.okhttp.HttpUrl.queryParameterValue(int)
(agent) Hooking com.android.okhttp.HttpUrl.queryParameterValues(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl.querySize()
(agent) Hooking com.android.okhttp.HttpUrl.resolve(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl.scheme()
(agent) Hooking com.android.okhttp.HttpUrl.toString()
(agent) Hooking com.android.okhttp.HttpUrl.uri()
(agent) Hooking com.android.okhttp.HttpUrl.url()
(agent) Hooking com.android.okhttp.HttpUrl.username()
(agent) Registering job 397848. Type: watch-class for: com.android.okhttp.HttpUrl
Running: 'android hooking watch class com.android.okhttp.HttpUrl$Builder':

(agent) Hooking com.android.okhttp.HttpUrl$Builder.canonicalizeHost(java.lang.String, int, int)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.containsInvalidHostnameAsciiCodes(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.decodeIpv4Suffix(java.lang.String, int, int, [B, int)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.decodeIpv6(java.lang.String, int, int)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.domainToAscii(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.inet6AddressToAscii([B)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.isDot(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.isDotDot(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.parsePort(java.lang.String, int, int)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.pop()
(agent) Hooking com.android.okhttp.HttpUrl$Builder.portColonOffset(java.lang.String, int, int)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.push(java.lang.String, int, int, boolean, boolean)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.removeAllCanonicalQueryParameters(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.resolvePath(java.lang.String, int, int)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.schemeDelimiterOffset(java.lang.String, int, int)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.skipLeadingAsciiWhitespace(java.lang.String, int, int)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.skipTrailingAsciiWhitespace(java.lang.String, int, int)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.slashCount(java.lang.String, int, int)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.addEncodedPathSegment(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.addEncodedQueryParameter(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.addPathSegment(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.addQueryParameter(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.build()
(agent) Hooking com.android.okhttp.HttpUrl$Builder.effectivePort()
(agent) Hooking com.android.okhttp.HttpUrl$Builder.encodedFragment(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.encodedPassword(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.encodedPath(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.encodedQuery(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.encodedUsername(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.fragment(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.host(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.parse(com.android.okhttp.HttpUrl, java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.password(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.port(int)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.query(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.reencodeForUri()
(agent) Hooking com.android.okhttp.HttpUrl$Builder.removeAllEncodedQueryParameters(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.removeAllQueryParameters(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.removePathSegment(int)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.scheme(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.setEncodedPathSegment(int, java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.setEncodedQueryParameter(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.setPathSegment(int, java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.setQueryParameter(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder.toString()
(agent) Hooking com.android.okhttp.HttpUrl$Builder.username(java.lang.String)
(agent) Registering job 562627. Type: watch-class for: com.android.okhttp.HttpUrl$Builder
Running: 'android hooking watch class com.android.okhttp.HttpUrl$Builder$ParseResult':

(agent) Hooking com.android.okhttp.HttpUrl$Builder$ParseResult.valueOf(java.lang.String)
(agent) Hooking com.android.okhttp.HttpUrl$Builder$ParseResult.valueOf()
(agent) Hooking com.android.okhttp.HttpUrl$Builder$ParseResult.values()
(agent) Registering job 698696. Type: watch-class for: com.android.okhttp.HttpUrl$Builder$ParseResult
Running: 'android hooking watch class com.android.okhttp.HttpsHandler':

(agent) Hooking com.android.okhttp.HttpsHandler.createHttpsOkUrlFactory(java.net.Proxy)
(agent) Hooking com.android.okhttp.HttpsHandler.getDefaultPort()
(agent) Hooking com.android.okhttp.HttpsHandler.newOkUrlFactory(java.net.Proxy)
(agent) Registering job 226237. Type: watch-class for: com.android.okhttp.HttpsHandler
Running: 'android hooking watch class com.android.okhttp.OkHttpClient':

(agent) Hooking com.android.okhttp.OkHttpClient.getDefaultSSLSocketFactory()
(agent) Hooking com.android.okhttp.OkHttpClient.cancel(java.lang.Object)
(agent) Hooking com.android.okhttp.OkHttpClient.clone()
(agent) Hooking com.android.okhttp.OkHttpClient.clone()
(agent) Hooking com.android.okhttp.OkHttpClient.copyWithDefaults()
(agent) Hooking com.android.okhttp.OkHttpClient.getAuthenticator()
(agent) Hooking com.android.okhttp.OkHttpClient.getCache()
(agent) Hooking com.android.okhttp.OkHttpClient.getCertificatePinner()
(agent) Hooking com.android.okhttp.OkHttpClient.getConnectTimeout()
(agent) Hooking com.android.okhttp.OkHttpClient.getConnectionPool()
(agent) Hooking com.android.okhttp.OkHttpClient.getConnectionSpecs()
(agent) Hooking com.android.okhttp.OkHttpClient.getCookieHandler()
(agent) Hooking com.android.okhttp.OkHttpClient.getDispatcher()
(agent) Hooking com.android.okhttp.OkHttpClient.getDns()
(agent) Hooking com.android.okhttp.OkHttpClient.getFollowRedirects()
(agent) Hooking com.android.okhttp.OkHttpClient.getFollowSslRedirects()
(agent) Hooking com.android.okhttp.OkHttpClient.getHostnameVerifier()
(agent) Hooking com.android.okhttp.OkHttpClient.getProtocols()
(agent) Hooking com.android.okhttp.OkHttpClient.getProxy()
(agent) Hooking com.android.okhttp.OkHttpClient.getProxySelector()
(agent) Hooking com.android.okhttp.OkHttpClient.getReadTimeout()
(agent) Hooking com.android.okhttp.OkHttpClient.getRetryOnConnectionFailure()
(agent) Hooking com.android.okhttp.OkHttpClient.getSocketFactory()
(agent) Hooking com.android.okhttp.OkHttpClient.getSslSocketFactory()
(agent) Hooking com.android.okhttp.OkHttpClient.getWriteTimeout()
(agent) Hooking com.android.okhttp.OkHttpClient.interceptors()
(agent) Hooking com.android.okhttp.OkHttpClient.internalCache()
(agent) Hooking com.android.okhttp.OkHttpClient.networkInterceptors()
(agent) Hooking com.android.okhttp.OkHttpClient.newCall(com.android.okhttp.Request)
(agent) Hooking com.android.okhttp.OkHttpClient.routeDatabase()
(agent) Hooking com.android.okhttp.OkHttpClient.setAuthenticator(com.android.okhttp.Authenticator)
(agent) Hooking com.android.okhttp.OkHttpClient.setCache(com.android.okhttp.Cache)
(agent) Hooking com.android.okhttp.OkHttpClient.setCertificatePinner(com.android.okhttp.CertificatePinner)
(agent) Hooking com.android.okhttp.OkHttpClient.setConnectTimeout(long, java.util.concurrent.TimeUnit)
(agent) Hooking com.android.okhttp.OkHttpClient.setConnectionPool(com.android.okhttp.ConnectionPool)
(agent) Hooking com.android.okhttp.OkHttpClient.setConnectionSpecs(java.util.List)
(agent) Hooking com.android.okhttp.OkHttpClient.setCookieHandler(java.net.CookieHandler)
(agent) Hooking com.android.okhttp.OkHttpClient.setDispatcher(com.android.okhttp.Dispatcher)
(agent) Hooking com.android.okhttp.OkHttpClient.setDns(com.android.okhttp.Dns)
(agent) Hooking com.android.okhttp.OkHttpClient.setFollowRedirects(boolean)
(agent) Hooking com.android.okhttp.OkHttpClient.setFollowSslRedirects(boolean)
(agent) Hooking com.android.okhttp.OkHttpClient.setHostnameVerifier(javax.net.ssl.HostnameVerifier)
(agent) Hooking com.android.okhttp.OkHttpClient.setInternalCache(com.android.okhttp.internal.InternalCache)
(agent) Hooking com.android.okhttp.OkHttpClient.setProtocols(java.util.List)
(agent) Hooking com.android.okhttp.OkHttpClient.setProxy(java.net.Proxy)
(agent) Hooking com.android.okhttp.OkHttpClient.setProxySelector(java.net.ProxySelector)
(agent) Hooking com.android.okhttp.OkHttpClient.setReadTimeout(long, java.util.concurrent.TimeUnit)
(agent) Hooking com.android.okhttp.OkHttpClient.setRetryOnConnectionFailure(boolean)
(agent) Hooking com.android.okhttp.OkHttpClient.setSocketFactory(javax.net.SocketFactory)
(agent) Hooking com.android.okhttp.OkHttpClient.setSslSocketFactory(javax.net.ssl.SSLSocketFactory)
(agent) Hooking com.android.okhttp.OkHttpClient.setWriteTimeout(long, java.util.concurrent.TimeUnit)
(agent) Registering job 937203. Type: watch-class for: com.android.okhttp.OkHttpClient
Running: 'android hooking watch class com.android.okhttp.OkHttpClient$1':

(agent) Hooking com.android.okhttp.OkHttpClient$1.addLenient(com.android.okhttp.Headers$Builder, java.lang.String)
(agent) Hooking com.android.okhttp.OkHttpClient$1.addLenient(com.android.okhttp.Headers$Builder, java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.OkHttpClient$1.apply(com.android.okhttp.ConnectionSpec, javax.net.ssl.SSLSocket, boolean)
(agent) Hooking com.android.okhttp.OkHttpClient$1.callEngineGetStreamAllocation(com.android.okhttp.Call)
(agent) Hooking com.android.okhttp.OkHttpClient$1.callEnqueue(com.android.okhttp.Call, com.android.okhttp.Callback, boolean)
(agent) Hooking com.android.okhttp.OkHttpClient$1.connectionBecameIdle(com.android.okhttp.ConnectionPool, com.android.okhttp.internal.io.RealConnection)
(agent) Hooking com.android.okhttp.OkHttpClient$1.get(com.android.okhttp.ConnectionPool, com.android.okhttp.Address, com.android.okhttp.internal.http.StreamAllocation)
(agent) Hooking com.android.okhttp.OkHttpClient$1.getHttpUrlChecked(java.lang.String)
(agent) Hooking com.android.okhttp.OkHttpClient$1.internalCache(com.android.okhttp.OkHttpClient)
(agent) Hooking com.android.okhttp.OkHttpClient$1.put(com.android.okhttp.ConnectionPool, com.android.okhttp.internal.io.RealConnection)
(agent) Hooking com.android.okhttp.OkHttpClient$1.routeDatabase(com.android.okhttp.ConnectionPool)
(agent) Hooking com.android.okhttp.OkHttpClient$1.setCache(com.android.okhttp.OkHttpClient, com.android.okhttp.internal.InternalCache)
(agent) Registering job 013029. Type: watch-class for: com.android.okhttp.OkHttpClient$1
Running: 'android hooking watch class com.android.okhttp.OkUrlFactory':

(agent) Hooking com.android.okhttp.OkUrlFactory.client()
(agent) Hooking com.android.okhttp.OkUrlFactory.clone()
(agent) Hooking com.android.okhttp.OkUrlFactory.clone()
(agent) Hooking com.android.okhttp.OkUrlFactory.createURLStreamHandler(java.lang.String)
(agent) Hooking com.android.okhttp.OkUrlFactory.open(java.net.URL)
(agent) Hooking com.android.okhttp.OkUrlFactory.open(java.net.URL, java.net.Proxy)
(agent) Hooking com.android.okhttp.OkUrlFactory.setUrlFilter(com.android.okhttp.internal.URLFilter)
(agent) Registering job 692517. Type: watch-class for: com.android.okhttp.OkUrlFactory
Running: 'android hooking watch class com.android.okhttp.Protocol':

(agent) Hooking com.android.okhttp.Protocol.get(java.lang.String)
(agent) Hooking com.android.okhttp.Protocol.valueOf(java.lang.String)
(agent) Hooking com.android.okhttp.Protocol.valueOf()
(agent) Hooking com.android.okhttp.Protocol.values()
(agent) Hooking com.android.okhttp.Protocol.toString()
(agent) Registering job 410935. Type: watch-class for: com.android.okhttp.Protocol
Running: 'android hooking watch class com.android.okhttp.Request':

(agent) Hooking com.android.okhttp.Request.-get0(com.android.okhttp.Request)
(agent) Hooking com.android.okhttp.Request.-get1(com.android.okhttp.Request)
(agent) Hooking com.android.okhttp.Request.-get2(com.android.okhttp.Request)
(agent) Hooking com.android.okhttp.Request.-get3(com.android.okhttp.Request)
(agent) Hooking com.android.okhttp.Request.-get4(com.android.okhttp.Request)
(agent) Hooking com.android.okhttp.Request.body()
(agent) Hooking com.android.okhttp.Request.cacheControl()
(agent) Hooking com.android.okhttp.Request.header(java.lang.String)
(agent) Hooking com.android.okhttp.Request.headers()
(agent) Hooking com.android.okhttp.Request.headers(java.lang.String)
(agent) Hooking com.android.okhttp.Request.httpUrl()
(agent) Hooking com.android.okhttp.Request.isHttps()
(agent) Hooking com.android.okhttp.Request.method()
(agent) Hooking com.android.okhttp.Request.newBuilder()
(agent) Hooking com.android.okhttp.Request.tag()
(agent) Hooking com.android.okhttp.Request.toString()
(agent) Hooking com.android.okhttp.Request.uri()
(agent) Hooking com.android.okhttp.Request.url()
(agent) Hooking com.android.okhttp.Request.urlString()
(agent) Registering job 837976. Type: watch-class for: com.android.okhttp.Request
Running: 'android hooking watch class com.android.okhttp.Request$Builder':

(agent) Hooking com.android.okhttp.Request$Builder.-get0(com.android.okhttp.Request$Builder)
(agent) Hooking com.android.okhttp.Request$Builder.-get1(com.android.okhttp.Request$Builder)
(agent) Hooking com.android.okhttp.Request$Builder.-get2(com.android.okhttp.Request$Builder)
(agent) Hooking com.android.okhttp.Request$Builder.-get3(com.android.okhttp.Request$Builder)
(agent) Hooking com.android.okhttp.Request$Builder.-get4(com.android.okhttp.Request$Builder)
(agent) Hooking com.android.okhttp.Request$Builder.addHeader(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.Request$Builder.build()
(agent) Hooking com.android.okhttp.Request$Builder.cacheControl(com.android.okhttp.CacheControl)
(agent) Hooking com.android.okhttp.Request$Builder.delete()
(agent) Hooking com.android.okhttp.Request$Builder.delete(com.android.okhttp.RequestBody)
(agent) Hooking com.android.okhttp.Request$Builder.get()
(agent) Hooking com.android.okhttp.Request$Builder.head()
(agent) Hooking com.android.okhttp.Request$Builder.header(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.Request$Builder.headers(com.android.okhttp.Headers)
(agent) Hooking com.android.okhttp.Request$Builder.method(java.lang.String, com.android.okhttp.RequestBody)
(agent) Hooking com.android.okhttp.Request$Builder.patch(com.android.okhttp.RequestBody)
(agent) Hooking com.android.okhttp.Request$Builder.post(com.android.okhttp.RequestBody)
(agent) Hooking com.android.okhttp.Request$Builder.put(com.android.okhttp.RequestBody)
(agent) Hooking com.android.okhttp.Request$Builder.removeHeader(java.lang.String)
(agent) Hooking com.android.okhttp.Request$Builder.tag(java.lang.Object)
(agent) Hooking com.android.okhttp.Request$Builder.url(com.android.okhttp.HttpUrl)
(agent) Hooking com.android.okhttp.Request$Builder.url(java.lang.String)
(agent) Hooking com.android.okhttp.Request$Builder.url(java.net.URL)
(agent) Registering job 068287. Type: watch-class for: com.android.okhttp.Request$Builder
Running: 'android hooking watch class com.android.okhttp.RequestBody':

(agent) Hooking com.android.okhttp.RequestBody.create(com.android.okhttp.MediaType, com.android.okhttp.okio.ByteString)
(agent) Hooking com.android.okhttp.RequestBody.create(com.android.okhttp.MediaType, java.io.File)
(agent) Hooking com.android.okhttp.RequestBody.create(com.android.okhttp.MediaType, java.lang.String)
(agent) Hooking com.android.okhttp.RequestBody.create(com.android.okhttp.MediaType, [B)
(agent) Hooking com.android.okhttp.RequestBody.create(com.android.okhttp.MediaType, [B, int, int)
(agent) Hooking com.android.okhttp.RequestBody.contentLength()
(agent) Hooking com.android.okhttp.RequestBody.contentType()
(agent) Hooking com.android.okhttp.RequestBody.writeTo(com.android.okhttp.okio.BufferedSink)
(agent) Registering job 969730. Type: watch-class for: com.android.okhttp.RequestBody
Running: 'android hooking watch class com.android.okhttp.RequestBody$2':

(agent) Hooking com.android.okhttp.RequestBody$2.contentLength()
(agent) Hooking com.android.okhttp.RequestBody$2.contentType()
(agent) Hooking com.android.okhttp.RequestBody$2.writeTo(com.android.okhttp.okio.BufferedSink)
(agent) Registering job 827072. Type: watch-class for: com.android.okhttp.RequestBody$2
Running: 'android hooking watch class com.android.okhttp.Response':

(agent) Hooking com.android.okhttp.Response.-get0(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response.-get1(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response.-get2(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response.-get3(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response.-get4(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response.-get5(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response.-get6(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response.-get7(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response.-get8(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response.-get9(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response.body()
(agent) Hooking com.android.okhttp.Response.cacheControl()
(agent) Hooking com.android.okhttp.Response.cacheResponse()
(agent) Hooking com.android.okhttp.Response.challenges()
(agent) Hooking com.android.okhttp.Response.code()
(agent) Hooking com.android.okhttp.Response.handshake()
(agent) Hooking com.android.okhttp.Response.header(java.lang.String)
(agent) Hooking com.android.okhttp.Response.header(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.Response.headers()
(agent) Hooking com.android.okhttp.Response.headers(java.lang.String)
(agent) Hooking com.android.okhttp.Response.isRedirect()
(agent) Hooking com.android.okhttp.Response.isSuccessful()
(agent) Hooking com.android.okhttp.Response.message()
(agent) Hooking com.android.okhttp.Response.networkResponse()
(agent) Hooking com.android.okhttp.Response.newBuilder()
(agent) Hooking com.android.okhttp.Response.priorResponse()
(agent) Hooking com.android.okhttp.Response.protocol()
(agent) Hooking com.android.okhttp.Response.request()
(agent) Hooking com.android.okhttp.Response.toString()
(agent) Registering job 214432. Type: watch-class for: com.android.okhttp.Response
Running: 'android hooking watch class com.android.okhttp.Response$Builder':

(agent) Hooking com.android.okhttp.Response$Builder.-get0(com.android.okhttp.Response$Builder)
(agent) Hooking com.android.okhttp.Response$Builder.-get1(com.android.okhttp.Response$Builder)
(agent) Hooking com.android.okhttp.Response$Builder.-get2(com.android.okhttp.Response$Builder)
(agent) Hooking com.android.okhttp.Response$Builder.-get3(com.android.okhttp.Response$Builder)
(agent) Hooking com.android.okhttp.Response$Builder.-get4(com.android.okhttp.Response$Builder)
(agent) Hooking com.android.okhttp.Response$Builder.-get5(com.android.okhttp.Response$Builder)
(agent) Hooking com.android.okhttp.Response$Builder.-get6(com.android.okhttp.Response$Builder)
(agent) Hooking com.android.okhttp.Response$Builder.-get7(com.android.okhttp.Response$Builder)
(agent) Hooking com.android.okhttp.Response$Builder.-get8(com.android.okhttp.Response$Builder)
(agent) Hooking com.android.okhttp.Response$Builder.-get9(com.android.okhttp.Response$Builder)
(agent) Hooking com.android.okhttp.Response$Builder.checkPriorResponse(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response$Builder.checkSupportResponse(java.lang.String, com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response$Builder.addHeader(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.Response$Builder.body(com.android.okhttp.ResponseBody)
(agent) Hooking com.android.okhttp.Response$Builder.build()
(agent) Hooking com.android.okhttp.Response$Builder.cacheResponse(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response$Builder.code(int)
(agent) Hooking com.android.okhttp.Response$Builder.handshake(com.android.okhttp.Handshake)
(agent) Hooking com.android.okhttp.Response$Builder.header(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.Response$Builder.headers(com.android.okhttp.Headers)
(agent) Hooking com.android.okhttp.Response$Builder.message(java.lang.String)
(agent) Hooking com.android.okhttp.Response$Builder.networkResponse(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response$Builder.priorResponse(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.Response$Builder.protocol(com.android.okhttp.Protocol)
(agent) Hooking com.android.okhttp.Response$Builder.removeHeader(java.lang.String)
(agent) Hooking com.android.okhttp.Response$Builder.request(com.android.okhttp.Request)
(agent) Registering job 128722. Type: watch-class for: com.android.okhttp.Response$Builder
Running: 'android hooking watch class com.android.okhttp.ResponseBody':

(agent) Hooking com.android.okhttp.ResponseBody.charset()
(agent) Hooking com.android.okhttp.ResponseBody.create(com.android.okhttp.MediaType, long, com.android.okhttp.okio.BufferedSource)
(agent) Hooking com.android.okhttp.ResponseBody.create(com.android.okhttp.MediaType, java.lang.String)
(agent) Hooking com.android.okhttp.ResponseBody.create(com.android.okhttp.MediaType, [B)
(agent) Hooking com.android.okhttp.ResponseBody.byteStream()
(agent) Hooking com.android.okhttp.ResponseBody.bytes()
(agent) Hooking com.android.okhttp.ResponseBody.charStream()
(agent) Hooking com.android.okhttp.ResponseBody.close()
(agent) Hooking com.android.okhttp.ResponseBody.contentLength()
(agent) Hooking com.android.okhttp.ResponseBody.contentType()
(agent) Hooking com.android.okhttp.ResponseBody.source()
(agent) Hooking com.android.okhttp.ResponseBody.string()
(agent) Registering job 180561. Type: watch-class for: com.android.okhttp.ResponseBody
Running: 'android hooking watch class com.android.okhttp.Route':

(agent) Hooking com.android.okhttp.Route.equals(java.lang.Object)
(agent) Hooking com.android.okhttp.Route.getAddress()
(agent) Hooking com.android.okhttp.Route.getProxy()
(agent) Hooking com.android.okhttp.Route.getSocketAddress()
(agent) Hooking com.android.okhttp.Route.hashCode()
(agent) Hooking com.android.okhttp.Route.requiresTunnel()
(agent) Registering job 479161. Type: watch-class for: com.android.okhttp.Route
Running: 'android hooking watch class com.android.okhttp.TlsVersion':

(agent) Hooking com.android.okhttp.TlsVersion.forJavaName(java.lang.String)
(agent) Hooking com.android.okhttp.TlsVersion.valueOf(java.lang.String)
(agent) Hooking com.android.okhttp.TlsVersion.valueOf()
(agent) Hooking com.android.okhttp.TlsVersion.values()
(agent) Hooking com.android.okhttp.TlsVersion.javaName()
(agent) Registering job 528569. Type: watch-class for: com.android.okhttp.TlsVersion
Running: 'android hooking watch class com.android.okhttp.internal.ConnectionSpecSelector':

(agent) Hooking com.android.okhttp.internal.ConnectionSpecSelector.isFallbackPossible(javax.net.ssl.SSLSocket)
(agent) Hooking com.android.okhttp.internal.ConnectionSpecSelector.configureSecureSocket(javax.net.ssl.SSLSocket)
(agent) Hooking com.android.okhttp.internal.ConnectionSpecSelector.connectionFailed(java.io.IOException)
(agent) Registering job 613314. Type: watch-class for: com.android.okhttp.internal.ConnectionSpecSelector
Running: 'android hooking watch class com.android.okhttp.internal.Internal':

(agent) Hooking com.android.okhttp.internal.Internal.initializeInstanceForTests()
(agent) Hooking com.android.okhttp.internal.Internal.addLenient(com.android.okhttp.Headers$Builder, java.lang.String)
(agent) Hooking com.android.okhttp.internal.Internal.addLenient(com.android.okhttp.Headers$Builder, java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.internal.Internal.apply(com.android.okhttp.ConnectionSpec, javax.net.ssl.SSLSocket, boolean)
(agent) Hooking com.android.okhttp.internal.Internal.callEngineGetStreamAllocation(com.android.okhttp.Call)
(agent) Hooking com.android.okhttp.internal.Internal.callEnqueue(com.android.okhttp.Call, com.android.okhttp.Callback, boolean)
(agent) Hooking com.android.okhttp.internal.Internal.connectionBecameIdle(com.android.okhttp.ConnectionPool, com.android.okhttp.internal.io.RealConnection)
(agent) Hooking com.android.okhttp.internal.Internal.get(com.android.okhttp.ConnectionPool, com.android.okhttp.Address, com.android.okhttp.internal.http.StreamAllocation)
(agent) Hooking com.android.okhttp.internal.Internal.getHttpUrlChecked(java.lang.String)
(agent) Hooking com.android.okhttp.internal.Internal.internalCache(com.android.okhttp.OkHttpClient)
(agent) Hooking com.android.okhttp.internal.Internal.put(com.android.okhttp.ConnectionPool, com.android.okhttp.internal.io.RealConnection)
(agent) Hooking com.android.okhttp.internal.Internal.routeDatabase(com.android.okhttp.ConnectionPool)
(agent) Hooking com.android.okhttp.internal.Internal.setCache(com.android.okhttp.OkHttpClient, com.android.okhttp.internal.InternalCache)
(agent) Registering job 340815. Type: watch-class for: com.android.okhttp.internal.Internal
Running: 'android hooking watch class com.android.okhttp.internal.OptionalMethod':

(agent) Hooking com.android.okhttp.internal.OptionalMethod.getMethod(java.lang.Class)
(agent) Hooking com.android.okhttp.internal.OptionalMethod.getPublicMethod(java.lang.Class, java.lang.String, [Ljava.lang.Class;)
(agent) Hooking com.android.okhttp.internal.OptionalMethod.invoke(java.lang.Object, [Ljava.lang.Object;)
(agent) Hooking com.android.okhttp.internal.OptionalMethod.invokeOptional(java.lang.Object, [Ljava.lang.Object;)
(agent) Hooking com.android.okhttp.internal.OptionalMethod.invokeOptionalWithoutCheckedException(java.lang.Object, [Ljava.lang.Object;)
(agent) Hooking com.android.okhttp.internal.OptionalMethod.invokeWithoutCheckedException(java.lang.Object, [Ljava.lang.Object;)
(agent) Hooking com.android.okhttp.internal.OptionalMethod.isSupported(java.lang.Object)
(agent) Registering job 072913. Type: watch-class for: com.android.okhttp.internal.OptionalMethod
Running: 'android hooking watch class com.android.okhttp.internal.RouteDatabase':

(agent) Hooking com.android.okhttp.internal.RouteDatabase.connected(com.android.okhttp.Route)
(agent) Hooking com.android.okhttp.internal.RouteDatabase.failed(com.android.okhttp.Route)
(agent) Hooking com.android.okhttp.internal.RouteDatabase.failedRoutesCount()
(agent) Hooking com.android.okhttp.internal.RouteDatabase.shouldPostpone(com.android.okhttp.Route)
(agent) Registering job 353842. Type: watch-class for: com.android.okhttp.internal.RouteDatabase
Running: 'android hooking watch class com.android.okhttp.internal.URLFilter':

(agent) Hooking com.android.okhttp.internal.URLFilter.checkURLPermitted(java.net.URL)
(agent) Registering job 043100. Type: watch-class for: com.android.okhttp.internal.URLFilter
Running: 'android hooking watch class com.android.okhttp.internal.Util':

(agent) Hooking com.android.okhttp.internal.Util.checkOffsetAndCount(long, long, long)
(agent) Hooking com.android.okhttp.internal.Util.closeAll(java.io.Closeable, java.io.Closeable)
(agent) Hooking com.android.okhttp.internal.Util.closeQuietly(java.io.Closeable)
(agent) Hooking com.android.okhttp.internal.Util.closeQuietly(java.net.ServerSocket)
(agent) Hooking com.android.okhttp.internal.Util.closeQuietly(java.net.Socket)
(agent) Hooking com.android.okhttp.internal.Util.concat([Ljava.lang.String;, java.lang.String)
(agent) Hooking com.android.okhttp.internal.Util.contains([Ljava.lang.String;, java.lang.String)
(agent) Hooking com.android.okhttp.internal.Util.discard(com.android.okhttp.okio.Source, int, java.util.concurrent.TimeUnit)
(agent) Hooking com.android.okhttp.internal.Util.equal(java.lang.Object, java.lang.Object)
(agent) Hooking com.android.okhttp.internal.Util.hostHeader(com.android.okhttp.HttpUrl, boolean)
(agent) Hooking com.android.okhttp.internal.Util.immutableList(java.util.List)
(agent) Hooking com.android.okhttp.internal.Util.immutableList([Ljava.lang.Object;)
(agent) Hooking com.android.okhttp.internal.Util.immutableMap(java.util.Map)
(agent) Hooking com.android.okhttp.internal.Util.intersect([Ljava.lang.Object;, [Ljava.lang.Object;)
(agent) Hooking com.android.okhttp.internal.Util.intersect(java.lang.Class, [Ljava.lang.Object;, [Ljava.lang.Object;)
(agent) Hooking com.android.okhttp.internal.Util.isAndroidGetsocknameError(java.lang.AssertionError)
(agent) Hooking com.android.okhttp.internal.Util.md5Hex(java.lang.String)
(agent) Hooking com.android.okhttp.internal.Util.sha1(com.android.okhttp.okio.ByteString)
(agent) Hooking com.android.okhttp.internal.Util.shaBase64(java.lang.String)
(agent) Hooking com.android.okhttp.internal.Util.skipAll(com.android.okhttp.okio.Source, int, java.util.concurrent.TimeUnit)
(agent) Hooking com.android.okhttp.internal.Util.threadFactory(java.lang.String, boolean)
(agent) Hooking com.android.okhttp.internal.Util.toHumanReadableAscii(java.lang.String)
(agent) Registering job 301949. Type: watch-class for: com.android.okhttp.internal.Util
Running: 'android hooking watch class com.android.okhttp.internal.Util$1':

(agent) Hooking com.android.okhttp.internal.Util$1.newThread(java.lang.Runnable)
(agent) Registering job 953686. Type: watch-class for: com.android.okhttp.internal.Util$1
Running: 'android hooking watch class com.android.okhttp.internal.http.AuthenticatorAdapter':

(agent) Hooking com.android.okhttp.internal.http.AuthenticatorAdapter.getConnectToInetAddress(java.net.Proxy, com.android.okhttp.HttpUrl)
(agent) Hooking com.android.okhttp.internal.http.AuthenticatorAdapter.authenticate(java.net.Proxy, com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.internal.http.AuthenticatorAdapter.authenticateProxy(java.net.Proxy, com.android.okhttp.Response)
(agent) Registering job 734236. Type: watch-class for: com.android.okhttp.internal.http.AuthenticatorAdapter
Running: 'android hooking watch class com.android.okhttp.internal.http.CacheStrategy':

(agent) Hooking com.android.okhttp.internal.http.CacheStrategy.isCacheable(com.android.okhttp.Response, com.android.okhttp.Request)
(agent) Registering job 965479. Type: watch-class for: com.android.okhttp.internal.http.CacheStrategy
Running: 'android hooking watch class com.android.okhttp.internal.http.CacheStrategy$Factory':

(agent) Hooking com.android.okhttp.internal.http.CacheStrategy$Factory.cacheResponseAge()
(agent) Hooking com.android.okhttp.internal.http.CacheStrategy$Factory.computeFreshnessLifetime()
(agent) Hooking com.android.okhttp.internal.http.CacheStrategy$Factory.getCandidate()
(agent) Hooking com.android.okhttp.internal.http.CacheStrategy$Factory.hasConditions(com.android.okhttp.Request)
(agent) Hooking com.android.okhttp.internal.http.CacheStrategy$Factory.isFreshnessLifetimeHeuristic()
(agent) Hooking com.android.okhttp.internal.http.CacheStrategy$Factory.get()
(agent) Registering job 636611. Type: watch-class for: com.android.okhttp.internal.http.CacheStrategy$Factory
Running: 'android hooking watch class com.android.okhttp.internal.http.Http1xStream':

(agent) Hooking com.android.okhttp.internal.http.Http1xStream.-get0(com.android.okhttp.internal.http.Http1xStream)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.-get1(com.android.okhttp.internal.http.Http1xStream)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.-get2(com.android.okhttp.internal.http.Http1xStream)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.-get3(com.android.okhttp.internal.http.Http1xStream)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.-set0(com.android.okhttp.internal.http.Http1xStream, int)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.-wrap0(com.android.okhttp.internal.http.Http1xStream, com.android.okhttp.okio.ForwardingTimeout)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.detachTimeout(com.android.okhttp.okio.ForwardingTimeout)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.getTransferStream(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.cancel()
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.createRequestBody(com.android.okhttp.Request, long)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.finishRequest()
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.isClosed()
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.newChunkedSink()
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.newChunkedSource(com.android.okhttp.internal.http.HttpEngine)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.newFixedLengthSink(long)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.newFixedLengthSource(long)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.newUnknownLengthSource()
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.openResponseBody(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.readHeaders()
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.readResponse()
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.readResponseHeaders()
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.setHttpEngine(com.android.okhttp.internal.http.HttpEngine)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.writeRequest(com.android.okhttp.Headers, java.lang.String)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.writeRequestBody(com.android.okhttp.internal.http.RetryableSink)
(agent) Hooking com.android.okhttp.internal.http.Http1xStream.writeRequestHeaders(com.android.okhttp.Request)
(agent) Registering job 327823. Type: watch-class for: com.android.okhttp.internal.http.Http1xStream
Running: 'android hooking watch class com.android.okhttp.internal.http.Http1xStream$AbstractSource':

(agent) Hooking com.android.okhttp.internal.http.Http1xStream$AbstractSource.endOfInput()
(agent) Hooking com.android.okhttp.internal.http.Http1xStream$AbstractSource.timeout()
(agent) Hooking com.android.okhttp.internal.http.Http1xStream$AbstractSource.unexpectedEndOfInput()
(agent) Registering job 594343. Type: watch-class for: com.android.okhttp.internal.http.Http1xStream$AbstractSource
Running: 'android hooking watch class com.android.okhttp.internal.http.Http1xStream$ChunkedSource':

(agent) Hooking com.android.okhttp.internal.http.Http1xStream$ChunkedSource.readChunkSize()
(agent) Hooking com.android.okhttp.internal.http.Http1xStream$ChunkedSource.close()
(agent) Hooking com.android.okhttp.internal.http.Http1xStream$ChunkedSource.read(com.android.okhttp.okio.Buffer, long)
(agent) Registering job 809496. Type: watch-class for: com.android.okhttp.internal.http.Http1xStream$ChunkedSource
Running: 'android hooking watch class com.android.okhttp.internal.http.Http1xStream$FixedLengthSource':

(agent) Hooking com.android.okhttp.internal.http.Http1xStream$FixedLengthSource.close()
(agent) Hooking com.android.okhttp.internal.http.Http1xStream$FixedLengthSource.read(com.android.okhttp.okio.Buffer, long)
(agent) Registering job 635706. Type: watch-class for: com.android.okhttp.internal.http.Http1xStream$FixedLengthSource
Running: 'android hooking watch class com.android.okhttp.internal.http.HttpEngine':

(agent) Hooking com.android.okhttp.internal.http.HttpEngine.-get0(com.android.okhttp.internal.http.HttpEngine)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.-set0(com.android.okhttp.internal.http.HttpEngine, com.android.okhttp.Request)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.-wrap0(com.android.okhttp.internal.http.HttpEngine)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.cacheWritingResponse(com.android.okhttp.internal.http.CacheRequest, com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.combine(com.android.okhttp.Headers, com.android.okhttp.Headers)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.connect()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.createAddress(com.android.okhttp.OkHttpClient, com.android.okhttp.Request)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.hasBody(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.maybeCache()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.networkRequest(com.android.okhttp.Request)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.readNetworkResponse()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.stripBody(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.unzip(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.validate(com.android.okhttp.Response, com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.cancel()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.close()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.followUpRequest()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.getBufferedRequestBody()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.getConnection()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.getRequest()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.getRequestBody()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.getResponse()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.hasResponse()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.permitsRequestBody(com.android.okhttp.Request)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.readResponse()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.receiveHeaders(com.android.okhttp.Headers)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.recover(com.android.okhttp.internal.http.RouteException)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.recover(java.io.IOException)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.recover(java.io.IOException, com.android.okhttp.okio.Sink)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.releaseStreamAllocation()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.sameConnection(com.android.okhttp.HttpUrl)
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.sendRequest()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine.writingRequestHeaders()
(agent) Registering job 316819. Type: watch-class for: com.android.okhttp.internal.http.HttpEngine
Running: 'android hooking watch class com.android.okhttp.internal.http.HttpEngine$1':

(agent) Hooking com.android.okhttp.internal.http.HttpEngine$1.contentLength()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine$1.contentType()
(agent) Hooking com.android.okhttp.internal.http.HttpEngine$1.source()
(agent) Registering job 429732. Type: watch-class for: com.android.okhttp.internal.http.HttpEngine$1
Running: 'android hooking watch class com.android.okhttp.internal.http.HttpMethod':

(agent) Hooking com.android.okhttp.internal.http.HttpMethod.invalidatesCache(java.lang.String)
(agent) Hooking com.android.okhttp.internal.http.HttpMethod.permitsRequestBody(java.lang.String)
(agent) Hooking com.android.okhttp.internal.http.HttpMethod.redirectsToGet(java.lang.String)
(agent) Hooking com.android.okhttp.internal.http.HttpMethod.requiresRequestBody(java.lang.String)
(agent) Registering job 731834. Type: watch-class for: com.android.okhttp.internal.http.HttpMethod
Running: 'android hooking watch class com.android.okhttp.internal.http.HttpStream':

(agent) Hooking com.android.okhttp.internal.http.HttpStream.cancel()
(agent) Hooking com.android.okhttp.internal.http.HttpStream.createRequestBody(com.android.okhttp.Request, long)
(agent) Hooking com.android.okhttp.internal.http.HttpStream.finishRequest()
(agent) Hooking com.android.okhttp.internal.http.HttpStream.openResponseBody(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.internal.http.HttpStream.readResponseHeaders()
(agent) Hooking com.android.okhttp.internal.http.HttpStream.setHttpEngine(com.android.okhttp.internal.http.HttpEngine)
(agent) Hooking com.android.okhttp.internal.http.HttpStream.writeRequestBody(com.android.okhttp.internal.http.RetryableSink)
(agent) Hooking com.android.okhttp.internal.http.HttpStream.writeRequestHeaders(com.android.okhttp.Request)
(agent) Registering job 055616. Type: watch-class for: com.android.okhttp.internal.http.HttpStream
Running: 'android hooking watch class com.android.okhttp.internal.http.OkHeaders$1':

(agent) Hooking com.android.okhttp.internal.http.OkHeaders$1.compare(java.lang.Object, java.lang.Object)
(agent) Hooking com.android.okhttp.internal.http.OkHeaders$1.compare(java.lang.String, java.lang.String)
(agent) Registering job 357083. Type: watch-class for: com.android.okhttp.internal.http.OkHeaders$1
Running: 'android hooking watch class com.android.okhttp.internal.http.RealResponseBody':

(agent) Hooking com.android.okhttp.internal.http.RealResponseBody.contentLength()
(agent) Hooking com.android.okhttp.internal.http.RealResponseBody.contentType()
(agent) Hooking com.android.okhttp.internal.http.RealResponseBody.source()
(agent) Registering job 308337. Type: watch-class for: com.android.okhttp.internal.http.RealResponseBody
Running: 'android hooking watch class com.android.okhttp.internal.http.RequestException':

(agent) Hooking com.android.okhttp.internal.http.RequestException.getCause()
(agent) Hooking com.android.okhttp.internal.http.RequestException.getCause()
(agent) Registering job 080599. Type: watch-class for: com.android.okhttp.internal.http.RequestException
Running: 'android hooking watch class com.android.okhttp.internal.http.RequestLine':

(agent) Hooking com.android.okhttp.internal.http.RequestLine.get(com.android.okhttp.Request, java.net.Proxy$Type)
(agent) Hooking com.android.okhttp.internal.http.RequestLine.includeAuthorityInRequestLine(com.android.okhttp.Request, java.net.Proxy$Type)
(agent) Hooking com.android.okhttp.internal.http.RequestLine.requestPath(com.android.okhttp.HttpUrl)
(agent) Registering job 466036. Type: watch-class for: com.android.okhttp.internal.http.RequestLine
Running: 'android hooking watch class com.android.okhttp.internal.http.RetryableSink':

(agent) Hooking com.android.okhttp.internal.http.RetryableSink.close()
(agent) Hooking com.android.okhttp.internal.http.RetryableSink.contentLength()
(agent) Hooking com.android.okhttp.internal.http.RetryableSink.flush()
(agent) Hooking com.android.okhttp.internal.http.RetryableSink.timeout()
(agent) Hooking com.android.okhttp.internal.http.RetryableSink.write(com.android.okhttp.okio.Buffer, long)
(agent) Hooking com.android.okhttp.internal.http.RetryableSink.writeToSocket(com.android.okhttp.okio.Sink)
(agent) Registering job 084987. Type: watch-class for: com.android.okhttp.internal.http.RetryableSink
Running: 'android hooking watch class com.android.okhttp.internal.http.RouteException':

(agent) Hooking com.android.okhttp.internal.http.RouteException.addSuppressedIfPossible(java.io.IOException, java.io.IOException)
(agent) Hooking com.android.okhttp.internal.http.RouteException.addConnectException(java.io.IOException)
(agent) Hooking com.android.okhttp.internal.http.RouteException.getLastConnectException()
(agent) Registering job 119136. Type: watch-class for: com.android.okhttp.internal.http.RouteException
Running: 'android hooking watch class com.android.okhttp.internal.http.RouteSelector':

(agent) Hooking com.android.okhttp.internal.http.RouteSelector.getHostString(java.net.InetSocketAddress)
(agent) Hooking com.android.okhttp.internal.http.RouteSelector.hasNextInetSocketAddress()
(agent) Hooking com.android.okhttp.internal.http.RouteSelector.hasNextPostponed()
(agent) Hooking com.android.okhttp.internal.http.RouteSelector.hasNextProxy()
(agent) Hooking com.android.okhttp.internal.http.RouteSelector.nextInetSocketAddress()
(agent) Hooking com.android.okhttp.internal.http.RouteSelector.nextPostponed()
(agent) Hooking com.android.okhttp.internal.http.RouteSelector.nextProxy()
(agent) Hooking com.android.okhttp.internal.http.RouteSelector.resetNextInetSocketAddress(java.net.Proxy)
(agent) Hooking com.android.okhttp.internal.http.RouteSelector.resetNextProxy(com.android.okhttp.HttpUrl, java.net.Proxy)
(agent) Hooking com.android.okhttp.internal.http.RouteSelector.connectFailed(com.android.okhttp.Route, java.io.IOException)
(agent) Hooking com.android.okhttp.internal.http.RouteSelector.hasNext()
(agent) Hooking com.android.okhttp.internal.http.RouteSelector.next()
(agent) Registering job 863316. Type: watch-class for: com.android.okhttp.internal.http.RouteSelector
Running: 'android hooking watch class com.android.okhttp.internal.http.StatusLine':

(agent) Hooking com.android.okhttp.internal.http.StatusLine.get(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.internal.http.StatusLine.parse(java.lang.String)
(agent) Hooking com.android.okhttp.internal.http.StatusLine.toString()
(agent) Registering job 782407. Type: watch-class for: com.android.okhttp.internal.http.StatusLine
Running: 'android hooking watch class com.android.okhttp.internal.http.StreamAllocation':

(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.connectionFailed(java.io.IOException)
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.connectionFailed()
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.deallocate(boolean, boolean, boolean)
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.findConnection(int, int, int, boolean)
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.findHealthyConnection(int, int, int, boolean, boolean)
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.isRecoverable(com.android.okhttp.internal.http.RouteException)
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.isRecoverable(java.io.IOException)
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.release(com.android.okhttp.internal.io.RealConnection)
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.release()
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.routeDatabase()
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.acquire(com.android.okhttp.internal.io.RealConnection)
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.cancel()
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.connection()
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.newStream(int, int, int, boolean, boolean)
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.noNewStreams()
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.recover(com.android.okhttp.internal.http.RouteException)
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.recover(java.io.IOException, com.android.okhttp.okio.Sink)
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.stream()
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.streamFinished(com.android.okhttp.internal.http.HttpStream)
(agent) Hooking com.android.okhttp.internal.http.StreamAllocation.toString()
(agent) Registering job 062167. Type: watch-class for: com.android.okhttp.internal.http.StreamAllocation
Running: 'android hooking watch class com.android.okhttp.internal.huc.DelegatingHttpsURLConnection':

(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.addRequestProperty(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.connect()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.disconnect()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getAllowUserInteraction()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getCipherSuite()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getConnectTimeout()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getContent()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getContent([Ljava.lang.Class;)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getContentEncoding()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getContentLength()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getContentType()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getDate()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getDefaultUseCaches()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getDoInput()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getDoOutput()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getErrorStream()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getExpiration()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getHeaderField(int)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getHeaderField(java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getHeaderFieldDate(java.lang.String, long)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getHeaderFieldInt(java.lang.String, int)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getHeaderFieldKey(int)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getHeaderFields()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getHostnameVerifier()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getIfModifiedSince()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getInputStream()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getInstanceFollowRedirects()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getLastModified()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getLocalCertificates()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getLocalPrincipal()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getOutputStream()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getPeerPrincipal()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getPermission()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getReadTimeout()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getRequestMethod()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getRequestProperties()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getRequestProperty(java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getResponseCode()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getResponseMessage()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getSSLSocketFactory()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getServerCertificates()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getURL()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.getUseCaches()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.handshake()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setAllowUserInteraction(boolean)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setChunkedStreamingMode(int)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setConnectTimeout(int)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setDefaultUseCaches(boolean)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setDoInput(boolean)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setDoOutput(boolean)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setFixedLengthStreamingMode(int)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setHostnameVerifier(javax.net.ssl.HostnameVerifier)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setIfModifiedSince(long)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setInstanceFollowRedirects(boolean)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setReadTimeout(int)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setRequestMethod(java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setRequestProperty(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setSSLSocketFactory(javax.net.ssl.SSLSocketFactory)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.setUseCaches(boolean)
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.toString()
(agent) Hooking com.android.okhttp.internal.huc.DelegatingHttpsURLConnection.usingProxy()
(agent) Registering job 884340. Type: watch-class for: com.android.okhttp.internal.huc.DelegatingHttpsURLConnection
Running: 'android hooking watch class com.android.okhttp.internal.huc.HttpURLConnectionImpl':

(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.defaultUserAgent()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.execute(boolean)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getHeaders()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getResponse()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.initHttpEngine()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.newHttpEngine(java.lang.String, com.android.okhttp.internal.http.StreamAllocation, com.android.okhttp.internal.http.RetryableSink, com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.responseSourceHeader(com.android.okhttp.Response)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.setProtocols(java.lang.String, boolean)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.addRequestProperty(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.connect()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.disconnect()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getConnectTimeout()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getErrorStream()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getHeaderField(int)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getHeaderField(java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getHeaderFieldKey(int)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getHeaderFields()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getInputStream()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getInstanceFollowRedirects()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getOutputStream()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getPermission()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getReadTimeout()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getRequestProperties()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getRequestProperty(java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getResponseCode()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.getResponseMessage()
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.setConnectTimeout(int)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.setFixedLengthStreamingMode(int)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.setFixedLengthStreamingMode(long)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.setIfModifiedSince(long)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.setInstanceFollowRedirects(boolean)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.setReadTimeout(int)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.setRequestMethod(java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.setRequestProperty(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.HttpURLConnectionImpl.usingProxy()
(agent) Registering job 674991. Type: watch-class for: com.android.okhttp.internal.huc.HttpURLConnectionImpl
Running: 'android hooking watch class com.android.okhttp.internal.huc.HttpsURLConnectionImpl':

(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.addRequestProperty(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.connect()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.disconnect()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getAllowUserInteraction()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getCipherSuite()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getConnectTimeout()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getContent()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getContent([Ljava.lang.Class;)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getContentEncoding()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getContentLength()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getContentLengthLong()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getContentType()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getDate()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getDefaultUseCaches()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getDoInput()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getDoOutput()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getErrorStream()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getExpiration()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getHeaderField(int)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getHeaderField(java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getHeaderFieldDate(java.lang.String, long)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getHeaderFieldInt(java.lang.String, int)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getHeaderFieldKey(int)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getHeaderFieldLong(java.lang.String, long)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getHeaderFields()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getHostnameVerifier()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getIfModifiedSince()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getInputStream()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getInstanceFollowRedirects()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getLastModified()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getLocalCertificates()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getLocalPrincipal()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getOutputStream()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getPeerPrincipal()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getPermission()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getReadTimeout()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getRequestMethod()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getRequestProperties()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getRequestProperty(java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getResponseCode()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getResponseMessage()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getSSLSocketFactory()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getServerCertificates()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getURL()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.getUseCaches()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.handshake()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setAllowUserInteraction(boolean)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setChunkedStreamingMode(int)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setConnectTimeout(int)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setDefaultUseCaches(boolean)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setDoInput(boolean)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setDoOutput(boolean)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setFixedLengthStreamingMode(int)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setFixedLengthStreamingMode(long)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setHostnameVerifier(javax.net.ssl.HostnameVerifier)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setIfModifiedSince(long)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setInstanceFollowRedirects(boolean)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setReadTimeout(int)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setRequestMethod(java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setRequestProperty(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setSSLSocketFactory(javax.net.ssl.SSLSocketFactory)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.setUseCaches(boolean)
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.toString()
(agent) Hooking com.android.okhttp.internal.huc.HttpsURLConnectionImpl.usingProxy()
(agent) Registering job 739071. Type: watch-class for: com.android.okhttp.internal.huc.HttpsURLConnectionImpl
Running: 'android hooking watch class com.android.okhttp.internal.tls.OkHostnameVerifier':

(agent) Hooking com.android.okhttp.internal.tls.OkHostnameVerifier.allSubjectAltNames(java.security.cert.X509Certificate)
(agent) Hooking com.android.okhttp.internal.tls.OkHostnameVerifier.getSubjectAltNames(java.security.cert.X509Certificate, int)
(agent) Hooking com.android.okhttp.internal.tls.OkHostnameVerifier.verifyAsIpAddress(java.lang.String)
(agent) Hooking com.android.okhttp.internal.tls.OkHostnameVerifier.verifyHostName(java.lang.String, java.lang.String)
(agent) Hooking com.android.okhttp.internal.tls.OkHostnameVerifier.verifyHostName(java.lang.String, java.security.cert.X509Certificate)
(agent) Hooking com.android.okhttp.internal.tls.OkHostnameVerifier.verifyIpAddress(java.lang.String, java.security.cert.X509Certificate)
(agent) Hooking com.android.okhttp.internal.tls.OkHostnameVerifier.verify(java.lang.String, java.security.cert.X509Certificate)
(agent) Hooking com.android.okhttp.internal.tls.OkHostnameVerifier.verify(java.lang.String, javax.net.ssl.SSLSession)
(agent) Registering job 807249. Type: watch-class for: com.android.okhttp.internal.tls.OkHostnameVerifier
Running: 'android hooking watch class com.android.okhttp.okio.AsyncTimeout':

(agent) Hooking com.android.okhttp.okio.AsyncTimeout.-wrap0()
(agent) Hooking com.android.okhttp.okio.AsyncTimeout.awaitTimeout()
(agent) Hooking com.android.okhttp.okio.AsyncTimeout.cancelScheduledTimeout(com.android.okhttp.okio.AsyncTimeout)
(agent) Hooking com.android.okhttp.okio.AsyncTimeout.remainingNanos(long)
(agent) Hooking com.android.okhttp.okio.AsyncTimeout.scheduleTimeout(com.android.okhttp.okio.AsyncTimeout, long, boolean)
(agent) Hooking com.android.okhttp.okio.AsyncTimeout.enter()
(agent) Hooking com.android.okhttp.okio.AsyncTimeout.exit(java.io.IOException)
(agent) Hooking com.android.okhttp.okio.AsyncTimeout.exit(boolean)
(agent) Hooking com.android.okhttp.okio.AsyncTimeout.exit()
(agent) Hooking com.android.okhttp.okio.AsyncTimeout.newTimeoutException(java.io.IOException)
(agent) Hooking com.android.okhttp.okio.AsyncTimeout.sink(com.android.okhttp.okio.Sink)
(agent) Hooking com.android.okhttp.okio.AsyncTimeout.source(com.android.okhttp.okio.Source)
(agent) Hooking com.android.okhttp.okio.AsyncTimeout.timedOut()
(agent) Registering job 550719. Type: watch-class for: com.android.okhttp.okio.AsyncTimeout
Running: 'android hooking watch class com.android.okhttp.okio.AsyncTimeout$1':

(agent) Hooking com.android.okhttp.okio.AsyncTimeout$1.close()
(agent) Hooking com.android.okhttp.okio.AsyncTimeout$1.flush()
(agent) Hooking com.android.okhttp.okio.AsyncTimeout$1.timeout()
(agent) Hooking com.android.okhttp.okio.AsyncTimeout$1.toString()
(agent) Hooking com.android.okhttp.okio.AsyncTimeout$1.write(com.android.okhttp.okio.Buffer, long)
(agent) Registering job 451625. Type: watch-class for: com.android.okhttp.okio.AsyncTimeout$1
Running: 'android hooking watch class com.android.okhttp.okio.AsyncTimeout$2':

(agent) Hooking com.android.okhttp.okio.AsyncTimeout$2.close()
(agent) Hooking com.android.okhttp.okio.AsyncTimeout$2.read(com.android.okhttp.okio.Buffer, long)
(agent) Hooking com.android.okhttp.okio.AsyncTimeout$2.timeout()
(agent) Hooking com.android.okhttp.okio.AsyncTimeout$2.toString()
(agent) Registering job 787858. Type: watch-class for: com.android.okhttp.okio.AsyncTimeout$2
Running: 'android hooking watch class com.android.okhttp.okio.AsyncTimeout$Watchdog':

(agent) Hooking com.android.okhttp.okio.AsyncTimeout$Watchdog.run()
(agent) Registering job 541661. Type: watch-class for: com.android.okhttp.okio.AsyncTimeout$Watchdog
Running: 'android hooking watch class com.android.okhttp.okio.BufferedSink':

(agent) Hooking com.android.okhttp.okio.BufferedSink.buffer()
(agent) Hooking com.android.okhttp.okio.BufferedSink.emit()
(agent) Hooking com.android.okhttp.okio.BufferedSink.emitCompleteSegments()
(agent) Hooking com.android.okhttp.okio.BufferedSink.outputStream()
(agent) Hooking com.android.okhttp.okio.BufferedSink.write(com.android.okhttp.okio.ByteString)
(agent) Hooking com.android.okhttp.okio.BufferedSink.write(com.android.okhttp.okio.Source, long)
(agent) Hooking com.android.okhttp.okio.BufferedSink.write([B)
(agent) Hooking com.android.okhttp.okio.BufferedSink.write([B, int, int)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeAll(com.android.okhttp.okio.Source)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeByte(int)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeDecimalLong(long)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeHexadecimalUnsignedLong(long)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeInt(int)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeIntLe(int)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeLong(long)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeLongLe(long)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeShort(int)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeShortLe(int)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeString(java.lang.String, int, int, java.nio.charset.Charset)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeString(java.lang.String, java.nio.charset.Charset)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeUtf8(java.lang.String)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeUtf8(java.lang.String, int, int)
(agent) Hooking com.android.okhttp.okio.BufferedSink.writeUtf8CodePoint(int)
(agent) Registering job 868670. Type: watch-class for: com.android.okhttp.okio.BufferedSink
Running: 'android hooking watch class com.android.okhttp.okio.BufferedSource':

(agent) Hooking com.android.okhttp.okio.BufferedSource.buffer()
(agent) Hooking com.android.okhttp.okio.BufferedSource.exhausted()
(agent) Hooking com.android.okhttp.okio.BufferedSource.indexOf(byte)
(agent) Hooking com.android.okhttp.okio.BufferedSource.indexOf(byte, long)
(agent) Hooking com.android.okhttp.okio.BufferedSource.indexOf(com.android.okhttp.okio.ByteString)
(agent) Hooking com.android.okhttp.okio.BufferedSource.indexOf(com.android.okhttp.okio.ByteString, long)
(agent) Hooking com.android.okhttp.okio.BufferedSource.indexOfElement(com.android.okhttp.okio.ByteString)
(agent) Hooking com.android.okhttp.okio.BufferedSource.indexOfElement(com.android.okhttp.okio.ByteString, long)
(agent) Hooking com.android.okhttp.okio.BufferedSource.inputStream()
(agent) Hooking com.android.okhttp.okio.BufferedSource.read([B)
(agent) Hooking com.android.okhttp.okio.BufferedSource.read([B, int, int)
(agent) Hooking com.android.okhttp.okio.BufferedSource.readAll(com.android.okhttp.okio.Sink)
(agent) Hooking com.android.okhttp.okio.BufferedSource.readByte()
(agent) Hooking com.android.okhttp.okio.BufferedSource.readByteArray()
(agent) Hooking com.android.okhttp.okio.BufferedSource.readByteArray(long)
(agent) Hooking com.android.okhttp.okio.BufferedSource.readByteString()
(agent) Hooking com.android.okhttp.okio.BufferedSource.readByteString(long)
(agent) Hooking com.android.okhttp.okio.BufferedSource.readDecimalLong()
(agent) Hooking com.android.okhttp.okio.BufferedSource.readFully(com.android.okhttp.okio.Buffer, long)
(agent) Hooking com.android.okhttp.okio.BufferedSource.readFully([B)
(agent) Hooking com.android.okhttp.okio.BufferedSource.readHexadecimalUnsignedLong()
(agent) Hooking com.android.okhttp.okio.BufferedSource.readInt()
(agent) Hooking com.android.okhttp.okio.BufferedSource.readIntLe()
(agent) Hooking com.android.okhttp.okio.BufferedSource.readLong()
(agent) Hooking com.android.okhttp.okio.BufferedSource.readLongLe()
(agent) Hooking com.android.okhttp.okio.BufferedSource.readShort()
(agent) Hooking com.android.okhttp.okio.BufferedSource.readShortLe()
(agent) Hooking com.android.okhttp.okio.BufferedSource.readString(long, java.nio.charset.Charset)
(agent) Hooking com.android.okhttp.okio.BufferedSource.readString(java.nio.charset.Charset)
(agent) Hooking com.android.okhttp.okio.BufferedSource.readUtf8()
(agent) Hooking com.android.okhttp.okio.BufferedSource.readUtf8(long)
(agent) Hooking com.android.okhttp.okio.BufferedSource.readUtf8CodePoint()
(agent) Hooking com.android.okhttp.okio.BufferedSource.readUtf8Line()
(agent) Hooking com.android.okhttp.okio.BufferedSource.readUtf8LineStrict()
(agent) Hooking com.android.okhttp.okio.BufferedSource.request(long)
(agent) Hooking com.android.okhttp.okio.BufferedSource.require(long)
(agent) Hooking com.android.okhttp.okio.BufferedSource.skip(long)
(agent) Registering job 927827. Type: watch-class for: com.android.okhttp.okio.BufferedSource
Running: 'android hooking watch class com.android.okhttp.okio.ForwardingTimeout':

(agent) Hooking com.android.okhttp.okio.ForwardingTimeout.clearDeadline()
(agent) Hooking com.android.okhttp.okio.ForwardingTimeout.clearTimeout()
(agent) Hooking com.android.okhttp.okio.ForwardingTimeout.deadlineNanoTime()
(agent) Hooking com.android.okhttp.okio.ForwardingTimeout.deadlineNanoTime(long)
(agent) Hooking com.android.okhttp.okio.ForwardingTimeout.delegate()
(agent) Hooking com.android.okhttp.okio.ForwardingTimeout.hasDeadline()
(agent) Hooking com.android.okhttp.okio.ForwardingTimeout.setDelegate(com.android.okhttp.okio.Timeout)
(agent) Hooking com.android.okhttp.okio.ForwardingTimeout.throwIfReached()
(agent) Hooking com.android.okhttp.okio.ForwardingTimeout.timeout(long, java.util.concurrent.TimeUnit)
(agent) Hooking com.android.okhttp.okio.ForwardingTimeout.timeoutNanos()
(agent) Registering job 238326. Type: watch-class for: com.android.okhttp.okio.ForwardingTimeout
Running: 'android hooking watch class com.android.okhttp.okio.GzipSource':

(agent) Hooking com.android.okhttp.okio.GzipSource.checkEqual(java.lang.String, int, int)
(agent) Hooking com.android.okhttp.okio.GzipSource.consumeHeader()
(agent) Hooking com.android.okhttp.okio.GzipSource.consumeTrailer()
(agent) Hooking com.android.okhttp.okio.GzipSource.updateCrc(com.android.okhttp.okio.Buffer, long, long)
(agent) Hooking com.android.okhttp.okio.GzipSource.close()
(agent) Hooking com.android.okhttp.okio.GzipSource.read(com.android.okhttp.okio.Buffer, long)
(agent) Hooking com.android.okhttp.okio.GzipSource.timeout()
(agent) Registering job 573939. Type: watch-class for: com.android.okhttp.okio.GzipSource
Running: 'android hooking watch class com.android.okhttp.okio.InflaterSource':

(agent) Hooking com.android.okhttp.okio.InflaterSource.releaseInflatedBytes()
(agent) Hooking com.android.okhttp.okio.InflaterSource.close()
(agent) Hooking com.android.okhttp.okio.InflaterSource.read(com.android.okhttp.okio.Buffer, long)
(agent) Hooking com.android.okhttp.okio.InflaterSource.refill()
(agent) Hooking com.android.okhttp.okio.InflaterSource.timeout()
(agent) Registering job 180127. Type: watch-class for: com.android.okhttp.okio.InflaterSource
Running: 'android hooking watch class com.android.okhttp.okio.Okio$1':

(agent) Hooking com.android.okhttp.okio.Okio$1.close()
(agent) Hooking com.android.okhttp.okio.Okio$1.flush()
(agent) Hooking com.android.okhttp.okio.Okio$1.timeout()
(agent) Hooking com.android.okhttp.okio.Okio$1.toString()
(agent) Hooking com.android.okhttp.okio.Okio$1.write(com.android.okhttp.okio.Buffer, long)
(agent) Registering job 872672. Type: watch-class for: com.android.okhttp.okio.Okio$1
Running: 'android hooking watch class com.android.okhttp.okio.Okio$2':

(agent) Hooking com.android.okhttp.okio.Okio$2.close()
(agent) Hooking com.android.okhttp.okio.Okio$2.read(com.android.okhttp.okio.Buffer, long)
(agent) Hooking com.android.okhttp.okio.Okio$2.timeout()
(agent) Hooking com.android.okhttp.okio.Okio$2.toString()
(agent) Registering job 254945. Type: watch-class for: com.android.okhttp.okio.Okio$2
Running: 'android hooking watch class com.android.okhttp.okio.Okio$3':

(agent) Hooking com.android.okhttp.okio.Okio$3.newTimeoutException(java.io.IOException)
(agent) Hooking com.android.okhttp.okio.Okio$3.timedOut()
(agent) Registering job 325473. Type: watch-class for: com.android.okhttp.okio.Okio$3
Running: 'android hooking watch class com.android.okhttp.okio.RealBufferedSink':

(agent) Hooking com.android.okhttp.okio.RealBufferedSink.-get0(com.android.okhttp.okio.RealBufferedSink)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.buffer()
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.close()
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.emit()
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.emitCompleteSegments()
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.flush()
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.outputStream()
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.timeout()
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.toString()
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.write(com.android.okhttp.okio.ByteString)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.write(com.android.okhttp.okio.Source, long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.write([B)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.write([B, int, int)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.write(com.android.okhttp.okio.Buffer, long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeAll(com.android.okhttp.okio.Source)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeByte(int)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeDecimalLong(long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeHexadecimalUnsignedLong(long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeInt(int)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeIntLe(int)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeLong(long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeLongLe(long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeShort(int)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeShortLe(int)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeString(java.lang.String, int, int, java.nio.charset.Charset)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeString(java.lang.String, java.nio.charset.Charset)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeUtf8(java.lang.String)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeUtf8(java.lang.String, int, int)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink.writeUtf8CodePoint(int)
(agent) Registering job 903572. Type: watch-class for: com.android.okhttp.okio.RealBufferedSink
Running: 'android hooking watch class com.android.okhttp.okio.RealBufferedSink$1':

(agent) Hooking com.android.okhttp.okio.RealBufferedSink$1.close()
(agent) Hooking com.android.okhttp.okio.RealBufferedSink$1.flush()
(agent) Hooking com.android.okhttp.okio.RealBufferedSink$1.toString()
(agent) Hooking com.android.okhttp.okio.RealBufferedSink$1.write(int)
(agent) Hooking com.android.okhttp.okio.RealBufferedSink$1.write([B, int, int)
(agent) Registering job 846177. Type: watch-class for: com.android.okhttp.okio.RealBufferedSink$1
Running: 'android hooking watch class com.android.okhttp.okio.RealBufferedSource':

(agent) Hooking com.android.okhttp.okio.RealBufferedSource.-get0(com.android.okhttp.okio.RealBufferedSource)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.rangeEquals(long, com.android.okhttp.okio.ByteString)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.buffer()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.close()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.exhausted()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.indexOf(byte)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.indexOf(byte, long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.indexOf(com.android.okhttp.okio.ByteString)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.indexOf(com.android.okhttp.okio.ByteString, long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.indexOfElement(com.android.okhttp.okio.ByteString)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.indexOfElement(com.android.okhttp.okio.ByteString, long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.inputStream()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.read([B)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.read([B, int, int)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.read(com.android.okhttp.okio.Buffer, long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readAll(com.android.okhttp.okio.Sink)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readByte()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readByteArray()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readByteArray(long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readByteString()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readByteString(long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readDecimalLong()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readFully(com.android.okhttp.okio.Buffer, long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readFully([B)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readHexadecimalUnsignedLong()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readInt()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readIntLe()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readLong()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readLongLe()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readShort()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readShortLe()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readString(long, java.nio.charset.Charset)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readString(java.nio.charset.Charset)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readUtf8()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readUtf8(long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readUtf8CodePoint()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readUtf8Line()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.readUtf8LineStrict()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.request(long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.require(long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.skip(long)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.timeout()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource.toString()
(agent) Registering job 854599. Type: watch-class for: com.android.okhttp.okio.RealBufferedSource
Running: 'android hooking watch class com.android.okhttp.okio.RealBufferedSource$1':

(agent) Hooking com.android.okhttp.okio.RealBufferedSource$1.available()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource$1.close()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource$1.read()
(agent) Hooking com.android.okhttp.okio.RealBufferedSource$1.read([B, int, int)
(agent) Hooking com.android.okhttp.okio.RealBufferedSource$1.toString()
(agent) Registering job 183749. Type: watch-class for: com.android.okhttp.okio.RealBufferedSource$1
Running: 'android hooking watch class com.android.okhttp.okio.Segment':

(agent) Hooking com.android.okhttp.okio.Segment.compact()
(agent) Hooking com.android.okhttp.okio.Segment.pop()
(agent) Hooking com.android.okhttp.okio.Segment.push(com.android.okhttp.okio.Segment)
(agent) Hooking com.android.okhttp.okio.Segment.split(int)
(agent) Hooking com.android.okhttp.okio.Segment.writeTo(com.android.okhttp.okio.Segment, int)
(agent) Registering job 680501. Type: watch-class for: com.android.okhttp.okio.Segment
Running: 'android hooking watch class com.android.okhttp.okio.Sink':

(agent) Hooking com.android.okhttp.okio.Sink.close()
(agent) Hooking com.android.okhttp.okio.Sink.flush()
(agent) Hooking com.android.okhttp.okio.Sink.timeout()
(agent) Hooking com.android.okhttp.okio.Sink.write(com.android.okhttp.okio.Buffer, long)
(agent) Registering job 318093. Type: watch-class for: com.android.okhttp.okio.Sink
Running: 'android hooking watch class com.android.okhttp.okio.Source':

(agent) Hooking com.android.okhttp.okio.Source.close()
(agent) Hooking com.android.okhttp.okio.Source.read(com.android.okhttp.okio.Buffer, long)
(agent) Hooking com.android.okhttp.okio.Source.timeout()
(agent) Registering job 157494. Type: watch-class for: com.android.okhttp.okio.Source
Running: 'android hooking watch class com.android.okhttp.okio.Timeout':

(agent) Hooking com.android.okhttp.okio.Timeout.clearDeadline()
(agent) Hooking com.android.okhttp.okio.Timeout.clearTimeout()
(agent) Hooking com.android.okhttp.okio.Timeout.deadline(long, java.util.concurrent.TimeUnit)
(agent) Hooking com.android.okhttp.okio.Timeout.deadlineNanoTime()
(agent) Hooking com.android.okhttp.okio.Timeout.deadlineNanoTime(long)
(agent) Hooking com.android.okhttp.okio.Timeout.hasDeadline()
(agent) Hooking com.android.okhttp.okio.Timeout.throwIfReached()
(agent) Hooking com.android.okhttp.okio.Timeout.timeout(long, java.util.concurrent.TimeUnit)
(agent) Hooking com.android.okhttp.okio.Timeout.timeoutNanos()
(agent) Registering job 907764. Type: watch-class for: com.android.okhttp.okio.Timeout
Running: 'android hooking watch class com.android.okhttp.okio.Timeout$1':

(agent) Hooking com.android.okhttp.okio.Timeout$1.deadlineNanoTime(long)
(agent) Hooking com.android.okhttp.okio.Timeout$1.throwIfReached()
(agent) Hooking com.android.okhttp.okio.Timeout$1.timeout(long, java.util.concurrent.TimeUnit)
(agent) Registering job 178230. Type: watch-class for: com.android.okhttp.okio.Timeout$1
Running: 'android hooking watch class':

Usage: android hooking watch class <class> (eg: com.example.test)

     _   _         _   _
 ___| |_|_|___ ___| |_|_|___ ___
| . | . | | -_|  _|  _| | . |   |
|___|___| |___|___|_| |_|___|_|_|
      |___|(object)inject(ion) v1.11.0

     Runtime Mobile Exploration
        by: @leonjza from @sensepost

[tab] for command suggestions
com.cz.babySister on (google: 8.1.0) [usb] # 


```

在将所有类都Hook上后，便可以在手机上对App进行操作

这里主要想抓到登录时的网络通信数据，所以在登录框中 输入用户名和密码后单击“确定”按钮。

如果Objection的界面中没 有任何函数被调用到，就说明实际发包函数并没有使用对应的框架， 这时应当更换包含Hook命令的文本文件重新回到第三步继续通过-c参 数执行文件中所有Hook命令进行trace；如果在Objection的界面上 看到一堆函数被调用，则说明框架类型确认成功。注意，这里虽有一 堆okhttp相关字眼，但并不是使用okhttp3网络框架完成的，而是因 为 HttpURLConnection 这 个 原 生 库 底 层 使 用 的 是 okhttp （ 与 okhttp3第三方网络框架并不是一个概念）。在确定了收发包框架的 同时，可以确定的是Objection中这些被调用的函数在App进行登录 的过程中一定是会被调用的。

```
com.cz.babySister on (google: 8.1.0) [usb] # (agent) [996967] Called com.android.okhttp.HttpHandler.openConnection(java.net.URL)
(agent) [996967] Called com.android.okhttp.HttpHandler.newOkUrlFactory(java.net.Proxy)
(agent) [996967] Called com.android.okhttp.HttpHandler.createHttpOkUrlFactory(java.net.Proxy)
(agent) [937203] Called com.android.okhttp.OkHttpClient.setConnectTimeout(long, java.util.concurrent.TimeUnit)
(agent) [937203] Called com.android.okhttp.OkHttpClient.setReadTimeout(long, java.util.concurrent.TimeUnit)
(agent) [937203] Called com.android.okhttp.OkHttpClient.setWriteTimeout(long, java.util.concurrent.TimeUnit)
(agent) [508942] Called com.android.okhttp.ConfigAwareConnectionPool.get()
(agent) [937203] Called com.android.okhttp.OkHttpClient.setConnectionPool(com.android.okhttp.ConnectionPool)
(agent) [717254] Called com.android.okhttp.ConfigAwareConnectionPool$1.onNetworkConfigurationChanged()
(agent) [717254] Called com.android.okhttp.ConfigAwareConnectionPool$1.onNetworkConfigurationChanged()
(agent) [996967] Called com.android.okhttp.HttpHandler.openConnection(java.net.URL)
(agent) [996967] Called com.android.okhttp.HttpHandler.newOkUrlFactory(java.net.Proxy)
(agent) [996967] Called com.android.okhttp.HttpHandler.createHttpOkUrlFactory(java.net.Proxy)
(agent) [937203] Called com.android.okhttp.OkHttpClient.setConnectTimeout(long, java.util.concurrent.TimeUnit)
(agent) [937203] Called com.android.okhttp.OkHttpClient.setReadTimeout(long, java.util.concurrent.TimeUnit)
(agent) [937203] Called com.android.okhttp.OkHttpClient.setWriteTimeout(long, java.util.concurrent.TimeUnit)
(agent) [508942] Called com.android.okhttp.ConfigAwareConnectionPool.get()
(agent) [937203] Called com.android.okhttp.OkHttpClient.setConnectionPool(com.android.okhttp.ConnectionPool)

```

任意选择一个被调用的函数，然后再退出objection，从新附加上APP后是以哦那个以下命令进行单一的Hook工作。



```
com.cz.babySister on (google: 8.1.0) [usb] # android hooking watch class_method com.android.okhttp.internal.http.RealResponseBody.source --dump-args --dump-backtrace --dump-return
(agent) Attempting to watch class com.android.okhttp.internal.http.RealResponseBody and method source.
(agent) Hooking com.android.okhttp.internal.http.RealResponseBody.source()
(agent) Registering job 225165. Type: watch-method for: com.android.okhttp.internal.http.RealResponseBody.source
com.cz.babySister on (google: 8.1.0) [usb] # (agent) [996967] Called com.android.okhttp.HttpHandler.openConnection(java.net.URL)
(agent) [716573] Called com.android.okhttp.HttpHandler.newOkUrlFactory(java.net.Proxy)
(agent) [716573] Backtrace:
        com.android.okhttp.HttpHandler.newOkUrlFactory(Native Method)
        com.android.okhttp.HttpHandler.openConnection(HttpHandler.java:44)
        com.android.okhttp.HttpHandler.openConnection(Native Method)
        java.net.URL.openConnection(URL.java:992)
        com.cz.babySister.c.a.a(HttpClients.java:22)
        com.cz.babySister.activity.fa.run(RegisterActivity.java:2)
        java.lang.Thread.run(Thread.java:764)


```

在Hook命令执行完成后，重新对App执行登录操作。最终定位到 App中关键的网络数据包发送的函数为com.cz.babySister.c.a.a函 数。为了印证App数据包发送的接口定位正确与否，再次对定 位到的函数进行Hook。最终在Hook得到的结果中不管是网络请求地 址还是用户名和密码都清晰可见。

```
(base) ┌──(root㉿kali)-[/home/zhy/桌面]
└─# objection -g 移动TV explore
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
com.cz.babySister on (google: 8.1.0) [usb] # android hooking watch class_method com.cz.babySister.c.a.a --dump-args --dump-backtrace --dump-return
(agent) Attempting to watch class com.cz.babySister.c.a and method a.
(agent) Hooking com.cz.babySister.c.a.a(java.io.InputStream)
(agent) Hooking com.cz.babySister.c.a.a(java.lang.String)
(agent) Hooking com.cz.babySister.c.a.a(java.lang.String, java.lang.String)
(agent) Registering job 762801. Type: watch-method for: com.cz.babySister.c.a.a
com.cz.babySister on (google: 8.1.0) [usb] # (agent) [762801] Called com.cz.babySister.c.a.a(java.lang.String, java.lang.String)
(agent) [762801] Backtrace:
        com.cz.babySister.c.a.a(Native Method)
        com.cz.babySister.activity.y.run(LoginActivity.java:2)
        java.lang.Thread.run(Thread.java:764)

(agent) [762801] Arguments com.cz.babySister.c.a.a(http://39.108.64.125/WebRoot/superMaster/Server, name=qqq&pass=wqqq)
(agent) [762801] Called com.cz.babySister.c.a.a(java.io.InputStream)
(agent) [762801] Backtrace:
        com.cz.babySister.c.a.a(Native Method)
        com.cz.babySister.c.a.a(HttpClients.java:31)
        com.cz.babySister.c.a.a(Native Method)
        com.cz.babySister.activity.y.run(LoginActivity.java:2)
        java.lang.Thread.run(Thread.java:764)

(agent) [762801] Arguments com.cz.babySister.c.a.a(buffer(com.android.okhttp.internal.http.Http1xStream$ChunkedSource@3e0de3d).inputStream())
(agent) [762801] Return Value: [{"memi1":"","pass":"@@@$$$%%$","jifen":"0","today":"2022-10-16","name":"name","vipday":"0","startviptime":"0","endviptime":"0","isvip":"false"}]
(agent) [762801] Return Value: [{"memi1":"","pass":"@@@$$$%%$","jifen":"0","today":"2022-10-16","name":"name","vipday":"0","startviptime":"0","endviptime":"0","isvip":"false"}]

```

但是该方法存在问题：

如果App通信是使用 第三方网络框架而App本身又存在着强度非常大的混淆，将App中一 些可以用来快速定位关键网络框架的关键字混淆成了无意义的字符， 那么Hook抓包定位的方式就失效了。
