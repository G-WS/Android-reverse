# iqiyi广告去除

主要分为两个广告：

- 一个是视频广告
- 一个是开屏广告

## 首先查壳

![image-20230414094036802](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230414094036802.png)

可以看到该软件没有被加固

## 开屏广告去除

### 定位

#### 广告定位

首先打开Jadx，查看`AndroidMainfest.xml`文件，查找main可以找到入口活动 `com.qiyi.video.speaker.activity.WelcomeActivity`

![image-20230414105323559](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230414105323559.png)

#### frida

将iqiyi通过`adb install`命令发送至测试机nexus 5X上。然后通过`adb shell`命令进入测试机，在 `data/local/tmp`目录下打开firda服务端。

在测试机上开启爱奇艺，在攻击机上使用 `frida-ps -U`查看进程名结果如下

![image-20230414105448378](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230414105448378.png)

接着对该进程名使用objection进行注入

![image-20230414105539366](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230414105539366.png)

由于我们定位到了广告页的类，直接hook该类，查看调用栈：

首先确定该类![image-20230414110036619](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230414110036619.png)

然后直接Hook调用栈

![image-20230414110723294](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230414110723294.png)

![image-20230414110736442](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230414110736442.png)

由于这是应用开启的一瞬间的调用，我们需要眼疾手快的打开应用的一瞬间打开objection，并完成对该类调用栈的Hook，在Hook之后我们可以观察到`access$200`该方法时在调用`showHomePage`方法之前。是一个很可疑的方法，我们回到代码

#### 代码分析

基本上可以确定的是`showHomePage`这个方法是我们主页的方法，回到jadx进行代码分析

```java
    private void checkPermission() {
        if (lpt2.br(InitHelper.getInstance().checkInitPermission(this))) {
            jumpToMain();
            return;
        }
        List<String> checkInitPermission = InitHelper.getInstance().checkInitPermission(this);
        androidx.core.app.aux.a(this, (String[]) checkInitPermission.toArray(new String[checkInitPermission.size()]), 1);
    }

    private void jumpToMain() {
        Log.e("gzy", "size:" + SpeakerApplication.getInstance().getCurrentActivitySize());
        if (!org.qiyi.speaker.o.con.bLa()) {
            org.qiyi.speaker.o.con.a(this, this.mLisenceCallback);
        } else if (GuideController.INSTANCE.needShowSplashGuide()) {
            showGuidePage();
        } else {
            launchMain(false);
        }
    }

    private void launchAppGuide() {
        org.qiyi.basecore.j.lpt2.buP().xz(R.id.event_privacy_terms_granted);
        aux auxVar = new aux();
        this.mLifecycleObservable = auxVar;
        auxVar.a(com.qiyi.video.g.b.aux.aXv().getWALifecycleObserver(new WALifecycleOwnImpl()));
        this.mLifecycleObservable.bPN();
    }

    /* JADX INFO: Access modifiers changed from: private */
    public void launchMain(final boolean z) {
        if (SpeakerApplication.getInstance().getCurrentActivitySize() != 1) {
            showHomePage(z);
            return;
        }
        com.qiyi.video.g.con.aXh().registerSplashCallback(new ISplashCallback() { // from class: com.qiyi.video.speaker.activity.WelcomeActivity.2
            @Override // org.qiyi.video.module.api.ISplashCallback
            public void onAdAnimationStarted() {
            }

            @Override // org.qiyi.video.module.api.ISplashCallback
            public void onAdCountdown(int i) {
            }

            @Override // org.qiyi.video.module.api.ISplashCallback
            public void onAdOpenDetailVideo() {
            }

            @Override // org.qiyi.video.module.api.ISplashCallback
            public void onAdStarted(String str) {
            }

            @Override // org.qiyi.video.module.api.ISplashCallback
            public void onSplashFinished(int i) {
                WelcomeActivity.this.showHomePage(z);
                JobManagerUtils.a(new Runnable() { // from class: com.qiyi.video.speaker.activity.WelcomeActivity.2.1
                    @Override // java.lang.Runnable
                    public void run() {
                        com.qiyi.video.qysplashscreen.ad.aux.aUv().aUE();
                        ((ISplashScreenApi) ModuleManager.getModule(IModuleConstants.MODULE_NAME_SPLASH_SCREEN, ISplashScreenApi.class)).requestAdAndDownload();
                    }
                }, 500, PageAutoScrollUtils.HANDLER_SWITCH_NEXT_TIPS_DELAY, "splashAD_requestad", WelcomeActivity.TAG);
            }
        });
        launchAppGuide();
    }
```

从代码中提取关键部分，我们可以看到，其对于直接进入主页的判断是当活动的数量不等于1，或者说大于等于2的时候直接调用showHomePage进入主页，否则下载广告并进行展示，我们直接对这里的条件判断进行修改。

```
 if (SpeakerApplication.getInstance().getCurrentActivitySize() != 1) {
            showHomePage(z);
            return;
        }
```

#### 动手

通过AndroidKiller，我们找到了WelcomeActivity.smali的文件在如下路径

![image-20230414123252972](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230414123252972.png)

通过baksmali将该classes5.dex文件反编译成smali，然后将打开搜索到launchmain，并对其进行修改，将条件改成永真

![image-20230414123502960](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230414123502960.png)

![image-20230414123524421](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230414123524421.png)

![image-20230414123554687](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230414123554687.png)

然后重新生成签名，安装到测试机中

测试后，成功去除了广告。

下一阶段，去除视频中的广告。

## 视频120秒广告去除

在打开应用随便找一个视频点击的时候会弹出一个`会员关闭此广告`，首先在Jadx里面全局搜索，勾选所有能勾选的给我返回了这样的两个搜索结果。

![image-20230417153312961](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417153312961.png)

可以看得出，第一个就是我们的目标，顺着资源文件找找调用 `player_ad_skip`，全局搜索这个字段

![image-20230417153500733](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417153500733.png)

我们可以看到其反射的id`0x7f1102fd`

![image-20230417155034152](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417155034152.png)

这是通过反射搜索到的搜索结果。我们通过开发者助手打开对应界面可以获得如下的信息



![image-20230417155124983](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417155124983.png)

![image-20230417155151903](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417155151903.png)

可以说是锁定了具体的Activity `com.qiyi.video.speaker.activity.PlayerActivity`

对该类进行搜索并进行Hook，可以看到该类所运行的一些函数

![image-20230417160616289](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417160616289.png)

下面是对应的函数调用栈

![image-20230417160738379](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417160738379.png)

这里并没有我们需要的内容，重新整理思路，既然我的目的是跳过视频广告，我可以去看看广告时间的判断，重新在开发者助手这里检查时间数字

![image-20230417161454495](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417161454495.png)

![image-20230417161548989](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417161548989.png)

回到Jadx，对该资源文件进行搜索 `R.id.account_ads_time_pre_ad`

![image-20230417161757005](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417161757005.png)

可以得到三处调用该函数的地方

对其进行Hook

**aux**![image-20230417162451233](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417162451233.png)

![image-20230417162516792](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417162516792.png)

**com1**

![image-20230417162619653](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417162619653.png)

**com8:**

![image-20230417162717374](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417162717374.png)

我们通过这三个类的Hook之后返回的函数调用栈，基本可以确认的一点就是这个调用栈主要在运行 `com.iqiyi.video.adview.roll.aux`类中的一些函数

对调用其的代码进行分析

```Java
 private void Vz() {
 	...
    this.bPB = (TextView) findViewById(R.id.account_ads_time_pre_ad);
    ...
 }

```

对于调用到该TextView的函数进行分析，在该类对bPB进行搜索：

我将具体的代码筛选了出来，可以看到下面的代码中，bPB一共被调用了三次。两次是在 `Xi()`函数，还有一次调用是在 `Xc()`函数中

> 这是chat对于这段程序的分析
>
> 分别对这两个类进行分析，首先分析`Xi()`方法：
>
> 这段代码主要是用来控制广告播放时的UI界面，包括一些按钮、文本的可见性和可用性等。
>
> 代码首先设置一个View的可见性为0，表示让这个View在界面上显示出来，接着获取当前广告播放状态以及广告UI策略，并打印一些日志。
>
> 然后，根据广告传送类型和广告配置等因素，设置一个按钮的背景图片和可见性。如果广告UI策略为1，则设置一些View的可见性为0；如果策略为2，则设置一些View的可见性为0并移除一些控件的触摸事件监听器；如果策略为3，则设置一些View的可见性为0，并根据是否静音设置广告静音；否则，设置另一些View的可见性和广告静音状态，并执行一个方法。
>
> 最后，如果广告传送类型不是6，则设置一个文本框的文本为广告的播放时长，并将其可见性设为0。
>
> 这段代码中的 `getAdDuration()` 方法的作用是获取广告的播放时长。
>
> 具体地，在最后一行代码中，调用了 `getAdDuration()` 方法，并将返回值转换为字符串，然后将其作为一个文本框的文本进行显示。因此，我们可以得出结论，`getAdDuration()` 方法的作用是获取广告的播放时长，并将其显示在UI界面上。

我对这两个函数进行了Hook，查看一些具体的调用去结合分析

![image-20230417165705476](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417165705476.png)

下面是对该类120秒广告的一个方法的完整的调用栈，我们对该函数栈进行分析

```sh
com.qiyi.video.speaker on (google: 8.1.0) [usb] # (agent) [516332] Called com.iqiyi.video.adview.roll.aux.Vz()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Sd()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Sd()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Sd()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xq()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.findViewById(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Vs()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xs()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.isMute()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.setAdMute(boolean, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.changeVideoSize(boolean, boolean, int, int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XD()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.WY()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xk()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dt(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.WZ()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.s(android.view.ViewGroup)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.qyplayersdk.cupid.g.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.setVideoResourceMode(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Vs()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xs()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.b(com.iqiyi.video.qyplayersdk.cupid.com5$aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xe()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XJ()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xg()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jI(java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jw(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dC(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dB(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dx(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dz(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dA(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dv(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dy(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dw(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(int, java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xi()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.VB()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.isMute()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.setAdMute(boolean, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xk()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XJ()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XD()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xg()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jI(java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jw(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dC(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dB(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dx(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dz(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dA(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dv(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dy(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dw(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(int, java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.WV()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.o(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.n(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xi()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.VB()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.isMute()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.setAdMute(boolean, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xk()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.n(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.o(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.p(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.n(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.onSurfaceChanged(int, int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XJ()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XD()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xg()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.x(java.lang.String, java.lang.String, java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jI(java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jw(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dC(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dB(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dx(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dz(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dA(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dv(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dy(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dw(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.x(java.lang.String, java.lang.String, java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(int, java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.WV()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.o(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.n(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xi()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.VB()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.isMute()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.setAdMute(boolean, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xk()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.onSurfaceChanged(int, int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XJ()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XD()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xg()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jI(java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jw(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dC(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dB(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dx(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dz(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dA(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dv(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dy(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dw(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(int, java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.WV()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.o(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.n(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xi()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.VB()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.isMute()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.setAdMute(boolean, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xk()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.n(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.o(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.p(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.onSurfaceChanged(int, int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XJ()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XD()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xg()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jI(java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jw(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dC(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dB(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dx(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dz(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dA(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dv(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dy(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dw(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(int, java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.WV()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.o(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.n(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xi()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.VB()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.isMute()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.setAdMute(boolean, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xk()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.n(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.o(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.p(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.v(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.onSurfaceChanged(int, int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XJ()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XD()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xg()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jI(java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jw(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dC(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dB(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dx(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dz(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dA(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dv(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dy(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dw(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(int, java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.WV()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.o(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.n(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xi()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.VB()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.isMute()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.setAdMute(boolean, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xk()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.onSurfaceChanged(int, int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XJ()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XD()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xg()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jI(java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jw(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dC(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dB(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dx(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dz(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dA(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dv(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dy(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dw(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(int, java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.WV()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.o(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.n(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xi()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.VB()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.isMute()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.setAdMute(boolean, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xk()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.n(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.o(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.p(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.l(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.onSurfaceChanged(int, int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XJ()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XD()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xg()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jI(java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jw(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dC(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dB(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dx(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dz(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dA(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dv(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dy(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dw(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(int, java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.WV()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.o(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.n(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xi()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.VB()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.isMute()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.setAdMute(boolean, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xk()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.onSurfaceChanged(int, int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XJ()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XD()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xg()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.x(java.lang.String, java.lang.String, java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jI(java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jw(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dC(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dB(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dx(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dz(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dA(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dv(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dy(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dw(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Wm()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xt()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.x(java.lang.String, java.lang.String, java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.m(int, java.lang.String)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.WV()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.o(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.n(com.iqiyi.video.qyplayersdk.cupid.data.model.CupidAD)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xi()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.VB()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.isMute()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.setAdMute(boolean, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xk()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(com.iqiyi.video.adview.roll.aux, android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.a(android.widget.LinearLayout)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.onSurfaceChanged(int, int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.UE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xb()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.onPreAdEnd()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.setAdMute(boolean, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XD()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.UE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xb()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.onPreAdEnd()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.setAdMute(boolean, boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XD()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.onSurfaceChanged(int, int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.onPause()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.dr(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.UE()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xb()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.du(boolean)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.onVideoChanged()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.release()

```

可以看到，循环最多的是这样的一个调用栈，一直在循环：

```shell
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.Xc()
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.jv(int)
(agent) [516332] Called com.iqiyi.video.adview.roll.aux.XE()
```

![image-20230417170729120](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417170729120.png)

```java
 private void Xi() {
        this.bSp.setVisibility(0);
        BaseState currentState = this.mAdInvoker.getCurrentState();
        int adUIStrategy = this.mAdInvoker.getAdUIStrategy();
        com.iqiyi.video.qyplayersdk.g.aux.i("PLAY_SDK_AD_ROLL", "{GPhoneRollAdView}", " show ad UI, current state = ", currentState, ", adUiStrategy: ", Integer.valueOf(adUIStrategy));
        VB();
        this.bPy.setBackgroundResource(currentState.isOnPaused() ? R.drawable.qiyi_sdk_play_ads_player : R.drawable.qiyi_sdk_play_ads_pause);
        int i = this.mDeliverType;
        boolean z = i == 3 || i == 7 || i == 4;
        QYPlayerADConfig adConfig = this.mAdInvoker.getAdConfig();
        int i2 = 8;
        this.bPy.setVisibility(z || adConfig == null || adConfig.isShowPause() ? 0 : 8);
        if (adUIStrategy == 1) {
            this.bPA.setVisibility(8);
            this.bPy.setVisibility(8);
            this.bPF.setVisibility(8);
            this.bPz.setVisibility(8);
        } else if (adUIStrategy == 2) {
            this.bPA.setVisibility(8);
            this.bPy.setVisibility(8);
            this.bPz.setVisibility(8);
            this.bSv.setVisibility(8);
            this.bSq.setVisibility(8);
            this.bSq.setOnTouchListener(null);
        } else if (adUIStrategy == 3) {
            this.bPA.setVisibility(8);
            this.bPF.setVisibility(8);
            boolean isMute = isMute();
            this.bPL = isMute;
            setAdMute(isMute, false);
        } else {
            this.bPF.setVisibility(0);
            TextView textView = this.bPA;
            if (!this.mIsLand) {
                i2 = 0;
            }
            textView.setVisibility(i2);
            boolean isMute2 = isMute();
            this.bPL = isMute2;
            setAdMute(isMute2, false);
            Xk();
        }
        if (this.mDeliverType != 6) {
            this.bPB.setVisibility(0);
        }
        this.bPB.setText(String.valueOf(this.mAdInvoker.getAdDuration()));
    }
```

```java
private void Xi() {
    ...
    // 获取当前广告播放器的状态
    BaseState currentState = this.mAdInvoker.getCurrentState();    
    // 获取了广告播放器的UI策略
    int adUIStrategy = this.mAdInvoker.getAdUIStrategy();      
    // 打印日志
    com.iqiyi.video.qyplayersdk.g.aux.i("PLAY_SDK_AD_ROLL", "{GPhoneRollAdView}", " show ad UI, current state = ", currentState, ", adUiStrategy: ", Integer.valueOf(adUIStrategy));
    // 设置视图的背景，根据当前广告播放器的状态来选择不同的背景资源
    this.bPy.setBackgroundResource(currentState.isOnPaused() ? R.drawable.qiyi_sdk_play_ads_player : R.drawable.qiyi_sdk_play_ads_pause);
    // 获取了当前广告的交付类型
    int i = this.mDeliverType;
    boolean z = i == 3 || i == 7 || i == 4;
    // 获取广告播放器配置
    QYPlayerADConfig adConfig = this.mAdInvoker.getAdConfig();
    int i2 = 8;
    // 根据UI策略的不同值，来设置一些视图的可见性或执行一些方法，8不可见，0可见
    if (adUIStrategy == 1) {
        this.bPA.setVisibility(8);
        this.bPy.setVisibility(8);
        this.bPF.setVisibility(8);
        this.bPz.setVisibility(8);
    } else if (adUIStrategy == 2) {
        this.bPA.setVisibility(8);
        this.bPy.setVisibility(8);
        this.bPz.setVisibility(8);
        this.bSv.setVisibility(8);
        this.bSq.setVisibility(8);
        this.bSq.setOnTouchListener(null);
    } else if (adUIStrategy == 3) {
        this.bPA.setVisibility(8);
        this.bPF.setVisibility(8);
        boolean isMute = isMute();      // 检查广告是否处于静音状态
        this.bPL = isMute;
        setAdMute(isMute, false);
    } else {
        this.bPF.setVisibility(0);
        TextView textView = this.bPA;
        if (!this.mIsLand) {
            i2 = 0;
        }
        textView.setVisibility(i2);
        boolean isMute2 = isMute();
        this.bPL = isMute2;
        setAdMute(isMute2, false);
        Xk();
    }
    if (this.mDeliverType != 6) {
        this.bPB.setVisibility(0);     // 设置时间视图可显
    }
    this.bPB.setText(String.valueOf(this.mAdInvoker.getAdDuration()));     // 给时间视图赋值
}
```

接着我们分析`Xc()`：

> 下面这是chat给出的分析
>
> 这段代码主要是用于处理广告计时相关的逻辑。具体来说，它完成了以下操作：
>
> 1. 通过调用 `getAdDuration()` 方法获取广告的播放时长，然后将其转换成字符串。
> 2. 如果广告控件 `bOR` 为 null，则直接返回。
> 3. 如果当前 `bTp` 为 0，则将 `bTp` 赋值为广告播放时长。
> 4. 调用 `jv()` 方法，用于更新广告播放的进度条。
> 5. 如果当前的广告类型为 `3` 或 `7`，则通过调用 `dz()` 方法来控制广告倒计时的显示。
> 6. 如果当前的广告类型为 `2`，则通过调用 `Xl()` 方法来控制广告倒计时的显示。
> 7. 如果当前的广告类型为 `6` 或者 `bSZ` 为 `true`，则调用 `jy()` 方法更新垂直方向的滚动广告的播放进度。
> 8. 如果当前的 `bTa` 为 `true`，则调用 `jy()` 方法更新水平方向的滚动广告的播放进度。
> 9. 最后，如果广告控件 `bSX` 不为 null，则调用 `jy()` 方法更新垂直方向的滚动广告的播放进度。

```java
public void Xc() {
        com8 com8Var;
        con conVar;
        int adDuration = this.mAdInvoker.getAdDuration();
        String str = adDuration + "";
        com.iqiyi.video.qyplayersdk.g.aux.i("PLAY_SDK_AD_ROLL", "{GPhoneRollAdView}", " adDuration:", str);
        if (this.bOR == null) {
            return;
        }
        if (this.bTp == 0) {
            this.bTp = adDuration;
        }
        jv(adDuration);
        if (XE()) {
            XH();
        }
        TextView textView = this.bPB;
        if (textView != null) {
            textView.setText(str);
        }
        int i = this.mDeliverType;
        if (i == 3 || i == 7) {
            if (adDuration < 1) {
                dz(false);
            } else {
                this.bSA.setText(str);
            }
        }
        if (this.mDeliverType == 2) {
            int Xp = Xp();
            if (Xp < 1) {
                Xl();
            } else {
                this.bSG.setText(this.mContext.getString(R.string.trueview_accountime, Integer.valueOf(Xp)));
            }
        }
        if ((this.bSZ || this.mDeliverType == 6) && (com8Var = this.bSS) != null) {
            com8Var.jy(adDuration);
        }
        if (this.bTa && (conVar = this.bSR) != null) {
            conVar.jy(adDuration);
        }
        com.iqiyi.video.adview.roll.vertical.nul nulVar = this.bSX;
        if (nulVar == null) {
            return;
        }
        nulVar.jy(adDuration);
    }
```

这是静态分析结合之前的调用栈分析后程序的目的

```java
public void Xc() {
    // 获取广告播放时长
    int adDuration = this.mAdInvoker.getAdDuration();   
    String str = adDuration + "";
    ...
 
    jv(adDuration);    // 判断能不能跳过广告
    if (XE()) {
        XH();
    }
    TextView textView = this.bPB;    // 设置剩余时间
    if (textView != null) {
        textView.setText(str);    // 显示非VIP持续时间
    }
    int i = this.mDeliverType;
    if (i == 3 || i == 7) {    // 如果交付类型是3或7 (VIP广告)，广告持续时间小于1，调用dz(false)
        if (adDuration < 1) {
            dz(false);    
        } else {
            this.bSA.setText(str);     // 显示VIP持续时间
        }
    }
    if (this.mDeliverType == 2) {     // 允许跳过的广告
        int Xp = Xp();    // 广告可跳过的剩余时间
        if (Xp < 1) {    // 允许跳过
            Xl();    // 显示跳过按钮
        } else {
            this.bSG.setText(this.mContext.getString(R.string.trueview_accountime, Integer.valueOf(Xp)));
        }
    }
    // 省流：根据不同的交付类型，为不同类型的广告进行时间配置与视图是否可显操作
    ...
}
```

根据程序调用栈对jv和xe进行分析

```java
// 处理广告的交互时间限制逻辑
private void jv(int i) {
    // 判断是否为触摸广告，是否支持点击跳转，并且是否已经被点击过
    if (!this.bOR.isTouchAd() || this.bOR.getClickThroughType() != 0 || this.bTn) {
        return;   // 是，直接返回
    }
    // 获取广告的预览信息
    PreAD creativeObject = this.bOR.getCreativeObject();
    // getInterTouchTime()是广告中点击交互的时间间隔，返回 10，表示用户需要等待至少 10 秒之后才能进行一次点击交互。小于0，说明可以点击。
    // 后面一个条件是指当前时间加上最早允许交互的时间点，如果超过广告总时长，则不允许交互，比如总时长120秒，getInterTouchTime() 返回 40，当前时间为100秒，大于总时长，不允许交互。
    if (creativeObject.getInterTouchTime() <= -1 || i + creativeObject.getInterTouchTime() > this.bTp) {
        return;
    }
    // 重置广告界面，继续播放
    this.bSq.reset();
    Wu();
}
```

```java
// 判断当前广告是创意广告
private boolean XE() {
    CupidAD<PreAD> cupidAD = this.bOR;
    if (cupidAD == null || cupidAD.getCreativeObject() == null) {
        return false;
    }
    return this.bOR.getDeliverType() == 10 || this.bOR.getDeliverType() == 11;
}
```

xc中有调用xp方法，对该方法进行分析

```java
// 计算广告可跳过的剩余时间
private int Xp() {
    if (this.bOR.getDeliverType() != 2) {
        return 0;
    }
    return (this.bOR.getSkippableTime() / 1000) - ((this.bOR.getDuration() / 1000) - this.mAdInvoker.getAdDuration());
}
```

这些地方都是对于布局文件的修改，并不存在跳过广告的判断，还需要继续去寻找，由于这两个类都调用到了一个`getAdDuration()`的方法

这个方法在Jadx中进行跟踪，发现属于包`package com.iqiyi.video.qyplayersdk.player;`里面的com6是一个接口类

![image-20230417172126527](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417172126527.png)

在这个里面没有什么用例的收获，还得继续找，在`QYMediaPlayerProxy`类中，在此找到了这个`getAdDuration()`的方法

在这个包的aux中找到了对应的接口方法的具体内容

```java
public int getAdDuration() {
        com.iqiyi.video.qyplayersdk.core.com1 com1Var = this.mPlayerCore;
        if (com1Var == null) {
            return 0;
        }
        return com1Var.getAdsTimeLength();
    }
```

我们观察到该方法返回了一个`getAdsTimeLength();`方法，我们通过用例查找定位到该方法。

```java
// 位于 com.iqiyi.video.qyplayersdk.core.QYBigCorePlayer 类中
public int getAdsTimeLength() {
    com8 com8Var = this.pumaPlayer;
    if (com8Var != null) {
        return Math.round(com8Var.GetADCountDown() / 1000.0f);   // 转成整数
    }
    return 0;
}
```

我们根据该方法中的`getADCountDown()`方法进行全局搜索

```java
// com.mcto.player.nativemediaplayer.NativeMediaPlayer 类中
public int GetADCountDown() {
    int GetADCountDown;
    if (IsCalledInPlayerThread()) {      // 判断是否在播放器线程中调用
        return this.native_media_player_bridge.GetADCountDown();    // 获取广告持续时间
    }
    synchronized (this) {
        if (!this.native_player_valid) {     // 判断播放器是否合法
            throw new MctoPlayerInvalidException(puma_state_error_msg);
        }
        GetADCountDown = this.native_media_player_bridge.GetADCountDown();
    }
    return GetADCountDown;
}
```

该防范又调用了一个`GetADCountDown()`方法，我们对该方法进行跳转

```java
 @Override // com.mcto.player.mctoplayer.IMctoPlayer
public int GetADCountDown() {
        // 调用了一个指定ID为43的方法，该方法返回一个JSON格式的字符串，其中包含有关广告信息的数据
    String InvokeMethod = InvokeMethod(43, "{}");
    if (InvokeMethod.isEmpty()) {  // 返回的字符串为空，则表示当前没有广告，方法返回0。
        return 0;
    }
    try {
                // 返回的字符串不为空，则将其转换为JSONObject对象，并获取其中名为ad_count_down的值
        return new JSONObject(InvokeMethod).getInt("ad_count_down");
    } catch (JSONException unused) {
        return 0;
    }
}
```

找到了这样的一个结果，看样子是需要native层的函数了

![image-20230417173809117](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417173809117.png)

看样子是被调用了很多

通过Hook之后可以发现![image-20230417195842855](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417195842855.png)

将所有的方法修改一遍不是很现实，我们需要再去寻找判断是否显示广告的地方。

**接下来去分析**`QYMediaPlayerProxy`对该类进行Hook

![image-20230417200401524](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417200401524.png)

```shell
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$000(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy, org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$001()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$002(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy, com.iqiyi.video.qyplayersdk.model.PlayerInfo)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$003(com.iqiyi.video.qyplayersdk.model.PlayerInfo, java.util.List)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$100(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy, long, com.iqiyi.video.qyplayersdk.model.PlayerInfo, java.lang.String, boolean, boolean, com.iqiyi.video.qyplayersdk.model.QYPlayerRecordConfig)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$1000(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy, org.iqiyi.video.mode.PlayData, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$1100(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy, com.iqiyi.video.qyplayersdk.model.PlayerInfo)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$1200(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy, int, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$1300(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy, org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$1400(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy, org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$200(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$300(java.lang.String, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$400(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy, org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$500(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$600(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy, com.iqiyi.video.qyplayersdk.model.PlayerInfo)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$700(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy, org.iqiyi.video.mode.PlayData, com.iqiyi.video.qyplayersdk.model.PlayerInfo)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$800(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.access$900(com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy, int, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.checkDrmError(org.iqiyi.video.data.PlayerError)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.checkDrmErrorV2(org.iqiyi.video.data.com7)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.checkIsNeedReplayback(boolean)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.checkNullPointException(java.lang.Object, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.checkPlayerCoreNullPointException()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.checkRcIfRcStrategyNeeded(org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.constructBitRateInfo()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.doVPlayAfterPlay(org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.doVPlayBeforePlay(org.iqiyi.video.mode.PlayData, boolean)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.doVPlayFullBeforePlay(org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getBigCoreBitRateInfo(boolean)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getCurrentAudioMode()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getErrorCodeVersion()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getLocalBitRateInfo()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getSystemBitRateInfo()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getVPlayPingbackParamsMap(org.iqiyi.video.mode.PlayData, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.ignoreNetworkInterceptByUA()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.isIgnoreFetchLastTimeSave()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.isNeedNetworkInterceptor(com.iqiyi.video.qyplayersdk.model.PlayerInfo)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.isNeedRequestVPlaySystemCore()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.isPreloadSuccessfully(org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.isPreloadSuccessfully()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.judgeSigtValid(java.lang.String, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.mergeWaterMarkInfo(com.iqiyi.video.qyplayersdk.model.PlayerInfo)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.notifyPlayerInfoChanged()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onFetchVPlayConditionFail(int, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onFetchVPlayConditionSuccess(com.iqiyi.video.qyplayersdk.model.PlayerInfo)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onFetchVPlayInfoFail(int, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onFetchVPlayInfoSuccess(com.iqiyi.video.qyplayersdk.model.PlayerInfo)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onPrepareNextVideoStart()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onPreviousVideoCompletion()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.performBigCorePlayback(org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.performBigCorePlayback(org.iqiyi.video.mode.PlayData, com.iqiyi.video.qyplayersdk.model.PlayerInfo)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.performBigCorePlayback(org.iqiyi.video.mode.PlayData, com.iqiyi.video.qyplayersdk.model.PlayerInfo, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.performSystemCorePlayback(org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.prepareBigCorePlayback(org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.prepareSystemCorePlayback(org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.requestVplayInfo(org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.resetData()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.savePlayerRecord(long, com.iqiyi.video.qyplayersdk.model.PlayerInfo, java.lang.String, boolean, boolean, com.iqiyi.video.qyplayersdk.model.QYPlayerRecordConfig)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.savePlayerRecordAsyn()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.sendVPlayRequestPingback(boolean, org.iqiyi.video.mode.PlayData, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.skipTitleIfUserSetted()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.updateEPGLiveData2PlayerInfo(com.iqiyi.video.qyplayersdk.player.data.model.EPGLiveData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.updatePlayDataRate(org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.updateTvIdToPlayerInfo(java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.updateWaterMarkInfo(com.iqiyi.video.qyplayersdk.model.PlayerInfo)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.updateWaterMarkInfo$___twin___(com.iqiyi.video.qyplayersdk.model.PlayerInfo)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.addCustomView(int, android.view.View, android.widget.RelativeLayout$LayoutParams)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.addEmbeddedView(android.view.View, android.widget.RelativeLayout$LayoutParams)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.capturePicture()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.changeAudioTrack(com.iqiyi.video.qyplayersdk.player.data.model.AudioTrack)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.changeBigCoreBitRate(org.iqiyi.video.mode.PlayerRate)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.changeSubtite(com.iqiyi.video.qyplayersdk.player.data.model.Subtitle)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.changeVideoSpeed(int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.clearSubTitle()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.createSurfaceAndWaterMark()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.currentIsAutoRate()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.dispatchOnSurfaceChanged(int, int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.dispatchOnSurfaceCreate(int, int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.doPreload(org.iqiyi.video.mode.PlayData, com.iqiyi.video.qyplayersdk.model.QYPlayerConfig, com.iqiyi.video.qyplayersdk.vplay.IVPlay$IVPlayCallback)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.doReplayLive(org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.dynamicReplaceWaterMarkResoure([Landroid.graphics.drawable.Drawable;, [Landroid.graphics.drawable.Drawable;)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getADConfig()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getAdDuration()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getAdUIStrategy()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getBufferLength()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getCurrentAudioTrack()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getCurrentPosition()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getCurrentVideoWidthHeight()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getCurrentVvId()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getDolbyTrialWatchingEndTime()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getDuration()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getEPGServerTime()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getFullScrrenSurfaceLayoutParameter()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getMovieJson()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getMovieJsonEntity()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getMultiViewUrl()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getNullableAudioTrackInfo()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getNullableBitRateInfo(boolean)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getNullableBuyInfo()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getNullableCurrentWaterMarkInfo()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getNullableSubtitleInfo()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getOnlyYouJson()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getPlayData()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getPlayerInfo()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getRenderAndPanoramaInfo()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getRenderView()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getScaleType()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getSigt()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getSurfaceHeight()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getSurfaceWidth()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getTitleTailInfo()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getVPlayInterceptor()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.getVideoInfo()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.ignoreThisTime(org.iqiyi.video.mode.TrialWatchingData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.init()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.initMovieJson()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.invokeQYPlayerAdCommand(int, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.invokeQYPlayerCommand(int, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.isCloseSubTitleShow()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.isHdcpLimit()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.isSupportAudioMode()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.isVRSrouce()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.login()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.notifyAdViewInvisible()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.notifyAdViewVisible()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.notifyConfigChanged(com.iqiyi.video.qyplayersdk.model.QYPlayerADConfig)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.notifyConfigChanged(com.iqiyi.video.qyplayersdk.model.QYPlayerControlConfig)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.notifyConfigChanged(com.iqiyi.video.qyplayersdk.model.QYPlayerDownloadConfig)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.notifyConfigChanged(com.iqiyi.video.qyplayersdk.model.QYPlayerRecordConfig)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.notifyPreAdDownloadStatus(java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onActivityPause()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onActivityResume(boolean)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onAdCallback(int, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onAudioTrackChange(boolean, com.iqiyi.video.qyplayersdk.player.data.model.AudioTrack)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onBitRateChange(boolean, org.iqiyi.video.mode.PlayerRate)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onError(org.iqiyi.video.data.PlayerError)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onErrorV2(org.iqiyi.video.data.com7)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onForceUseSoftWare()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onMovieStart()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onPlayDoblyEndCallback(java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onPreloadFeedDelete(java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onPreloadFeedHit(java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onPreloadSuccess()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onPrepared()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onQimoUnlockLayerShow(java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onTrialWatchingEnd()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onTrialWatchingStart(org.iqiyi.video.mode.TrialWatchingData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onUnlockErrorCallback()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.onVideoProgressChanged(long)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.parseEpisodeMessage(java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.pause()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.playback(org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.playerCupidAdStateChange(com.iqiyi.video.qyplayersdk.cupid.data.model.com6)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.postEvent(int, int, android.os.Bundle)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.rePlaybackCauseBySystemCoreChangeBitRate(org.iqiyi.video.mode.PlayerRate)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.rePreloadNextVideo()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.release()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.releaseCoreCallback()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.releasePlayerCore()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.requestContentBuyInfo(org.iqiyi.video.playernetwork.httprequest.IPlayerRequestCallBack)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.requestShowVipLayer(com.iqiyi.video.qyplayersdk.model.PlayerInfo)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.resetBufferInterceptor()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.resetPlayerInfo()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.retrieveStatistics(int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.retrieveStatistics2(java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.savePlayerRecordSync(long)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.seekTo(long)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setAdMute(boolean, boolean)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setBigcoreVPlayInterceptor(com.iqiyi.video.qyplayersdk.f.nul)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setCodecType(java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setCustomWaterMarkMargin(int, int, int, int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setFixedSize(int, int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setFullScreenTopBottomMargin(android.util.Pair)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setIWaterMarkController(com.iqiyi.videoview.player.com2)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setLiveMessage(int, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setLiveTrialWatchingLeftTime(long)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setMultiResId(java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setMute(int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setMuteWithIntercept(int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setNextMovieInfo(org.iqiyi.video.mode.PlayData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setParentView(android.view.ViewGroup, android.content.Context)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setPreLoadConfig(com.iqiyi.video.qyplayersdk.h.prn)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setPreloadFunction(com.iqiyi.video.qyplayersdk.player.com7, com.iqiyi.video.qyplayersdk.h.prn)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setShowVideoOriginSize4WaterMark(int, int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setStatisticsBizInjector(com.iqiyi.video.qyplayersdk.module.a.con)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setSystemCoreVPlayInterceptor(com.iqiyi.video.qyplayersdk.f.nul)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setTrialWatchingData(org.iqiyi.video.mode.TrialWatchingData)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setVVCollector(com.iqiyi.video.qyplayersdk.module.a.f.con)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setVideoSize(int, int, int, int, boolean, int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setVideoViewOffset(java.lang.Integer, java.lang.Integer)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.setVolume(int, int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.showDebugInfo(android.view.ViewGroup, com.iqiyi.video.qyplayersdk.e.nul, com.iqiyi.video.qyplayersdk.player.com9)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.showOrHideAdView(int, boolean)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.showOrHideWatermark(boolean)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.showSubTitle(java.lang.String, int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.skipSlide(boolean, boolean)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.start()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.startLoad()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.startNextMovie()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.stopLoad()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.stopPlayback()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.switchAudioMode(int, int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.switchToPip(boolean)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.switchToPip(boolean, int, int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.unregisterCtrlConfigObserver()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.updateAlbumInfoAndVideoInfo(com.iqiyi.video.qyplayersdk.model.PlayerAlbumInfo, com.iqiyi.video.qyplayersdk.model.PlayerVideoInfo)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.updateLiveSubtitleInfo()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.updatePlayCostStatistics()
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.updatePlayerCore(com.iqiyi.video.qyplayersdk.core.com1)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.updateQosData(java.lang.String, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.updateStartStatistics(java.lang.String, java.lang.Long)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.updateStatistics(int, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.updateStatistics2(java.lang.String, java.lang.String)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.updateViewPointAdLocation(int)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.useSameSurfaceTexture(boolean)
(agent) Hooking com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.videoSizeChanged(int, int)
(agent) Registering job 667631. Type: watch-class for: com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy

```

直接对该类进行Hook可以发现调用栈，在login函数之后有几个比较可疑的函数，可能与广告有关系。

![image-20230417201255373](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417201255373.png)

```sh
(agent) [667631] Called com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.prepareBigCorePlayback(org.iqiyi.video.mode.PlayData)
(agent) [667631] Called com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.doVPlayAfterPlay(org.iqiyi.video.mode.PlayData)
(agent) [667631] Called com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.performBigCorePlayback(org.iqiyi.video.mode.PlayData)
(agent) [667631] Called com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.performBigCorePlayback(org.iqiyi.video.mode.PlayData, com.iqiyi.video.qyplayersdk.model.PlayerInfo, java.lang.String)
```

首先是几个重要方法

```java
// setVVCollector()：设置VVCollector，收集播放器的VV统计信息。
// video view (VV)，意思为视频播放次数，根据广告播放次数，统计盈利。
public void setVVCollector(com.iqiyi.video.qyplayersdk.module.a.f.con conVar) {
    com.iqiyi.video.qyplayersdk.module.a.aux auxVar = this.mStatistics;
    if (auxVar != null) {
        auxVar.setVVCollector(conVar);
    }
}
 
// init(): 初始化播放器界面
// 获取了mControlConfig中的一些配置信息，例如编解码类型、是否自动跳过片头片尾、色盲模式等，然后调用prn.aux构造方法创建一个prn对象，并设置这些配置信息，最后通过a()方法将prn对象和mPassportAdapter对象一起传入a方法中，完成播放器的初始化。
public void init() {
    this.mPlayerCore.a(new prn.aux(this.mControlConfig.getCodecType())
            .eH(this.mControlConfig.isAutoSkipTitle())
            .eI(this.mControlConfig.isAutoSkipTrailer())
            .kR(this.mControlConfig.getColorBlindnessType())
            .lX(this.mControlConfig.getExtendInfo())
            .lY(this.mControlConfig.getExtraDecoderInfo())
            .aie(), com.iqiyi.video.qyplayersdk.core.data.aux.a(this.mPassportAdapter));
}
 
// 检查 RC 策略是否需要执行
// RC 策略是指在不同的地理位置或网络环境下，根据不同的版权限制或合作协议，播放不同的内容或提供不同的服务。
public PlayData checkRcIfRcStrategyNeeded(PlayData playData) {
    if (playData == null) {
        com.iqiyi.video.qyplayersdk.g.aux.d(TAG, "QYMediaPlayerProxy checkRcIfRcStrategyNeeded source == null!");
        return playData;
    }
    int rCCheckPolicy = playData.getRCCheckPolicy();
    com.iqiyi.video.qyplayersdk.g.aux.d(TAG, "QYMediaPlayerProxy checkRcIfRcStrategyNeeded strategy == " + rCCheckPolicy);
    if (this.mPlayerRecordAdapter == null) {
        this.mPlayerRecordAdapter = new PlayerRecordAdapter();
    }
        // 根据 RCCheckPolicy (即 RC 策略) 的值。
        // 如果值为 2，直接返回 playData；如果值为 1 或 0，，则调用 PlayerRecordAdapter 的 retrievePlayerRecord 方法，获取播放记录，
    return rCCheckPolicy == 2 ? playData : (rCCheckPolicy == 1 || rCCheckPolicy == 0) ?
        com.iqiyi.video.qyplayersdk.player.data.b.con.a(playData, this.mPlayerRecordAdapter.retrievePlayerRecord(playData)) : playData;
}
 
// 获取登录用户信息
void login() {
    IPassportAdapter iPassportAdapter;
        // mPlayerCore 是播放器核心，mPassportAdapter 是用户身份验证适配器。
    if (this.mPlayerCore == null || (iPassportAdapter = this.mPassportAdapter) == null) {
        return;
    }
        // 判断是不是VIP用户，并获取相应用户信息
    this.mPlayerCore.login(com.iqiyi.video.qyplayersdk.core.data.aux.a(iPassportAdapter));
}
```

在`switch`语句中，根据策略选择执行不同的操作。如果策略是1，则调用`performBigCorePlayback()`方法进行大核心播放。如果策略是2，则调用`doVPlayBeforePlay()`方法，并将`z`变量设置为`true`。如果策略是3，则调用`doVPlayFullBeforePlay()`方法。如果策略是4，则调用`doVPlayAfterPlay()`方法。如果策略是5，则打印日志信息。如果策略是6，则调用`doVPlayBeforePlay()`方法，并将`z`变量设置为`false`。

最后，该方法调用了`org.qiyi.android.coreplayer.d.com7.endSection()`方法，结束性能测量区块。

```java
  private void prepareBigCorePlayback(PlayData playData) {//playData应该是播放器的数据
        boolean z;
        org.qiyi.android.coreplayer.d.com7.beginSection("QYMediaPlayerProxy.prepareBigCorePlayback");
        com.iqiyi.video.qyplayersdk.h.con conVar = this.mPreload;
        if (conVar != null) {
            conVar.aoj();			//应该是预加载器
        }
        int a2 = com.iqiyi.video.qyplayersdk.player.data.b.nul.a(playData, this.mContext, this.mControlConfig);//播放策略
        com.iqiyi.video.qyplayersdk.g.aux.e("PLAY_SDK", "vplay strategy : " + a2);
        switch (a2) {
            case 1:
                performBigCorePlayback(playData);
                break;
            case 2:
                z = true;
                doVPlayBeforePlay(playData, z);
                break;
            case 3:
                doVPlayFullBeforePlay(playData);
                break;
            case 4:
                doVPlayAfterPlay(playData);
                break;
            case 5:
                if (com.iqiyi.video.qyplayersdk.g.aux.isDebug()) {
                    throw new RuntimeException("address & tvid & ctype are null");
                }
                com.iqiyi.video.qyplayersdk.g.aux.e("PLAY_SDK", "address & tvid & ctype are null");
                break;
            case 6:
                z = false;
                doVPlayBeforePlay(playData, z);
                break;
        }
        org.qiyi.android.coreplayer.d.com7.endSection();
    }
```

首先我们对`performBigCorePlayback(playData)`方法进行分析

三个重载方法，做了两次中转。该方法需要进行重点分析

```java
   private void performBigCorePlayback(PlayData playData) {
        performBigCorePlayback(playData, this.mPlayerInfo, "");
        this.mCurrentState.reset();
    }
   public void performBigCorePlayback(PlayData playData, PlayerInfo playerInfo) {
        performBigCorePlayback(playData, playerInfo, "");
    }
    //可以看到该方法做了两个中转

    private void performBigCorePlayback(PlayData playData, PlayerInfo playerInfo, String str) {
        int i;
        com.iqiyi.video.qyplayersdk.f.con conVar = this.mDoPlayInterceptor;
        if (conVar != null && conVar.e(playerInfo)) { //在该方法中，首先会检查一个“DoPlayInterceptor”的拦截器，如果它拦截了，则会调用“amX”方法进行处理。
            com.iqiyi.video.qyplayersdk.g.aux.d("PLAY_SDK", TAG, "DoPlayInterceptor is intercept!");
            lpt5 lpt5Var = this.mInvokerQYMediaPlayer;
            if (lpt5Var == null) {
                return;
            }
            lpt5Var.amX();
            //会检查播放器信息是否为null，如果是null则什么也不做
        } else if (this.mPlayerInfo == null) {
            //重点
        } else {
            org.qiyi.android.coreplayer.d.com7.beginSection("QYMediaPlayerProxy.performBigCorePlayback");
            // 通过判断播放数据（playData）是否为空以及是否存在播放地址，空则i = 0。
            if (com.iqiyi.video.qyplayersdk.player.data.b.nul.A(playerInfo) || playData == null) {
                i = 0;
            } else {
                // 如果有地址，根据该数据生成CupidVvId，并将该ID与广告相关的Ad对象（mAd）绑定。
                        // 所以这里就是去后台获取广告的id
                com.iqiyi.video.qyplayersdk.cupid.data.model.com9 a2 = com.iqiyi.video.qyplayersdk.cupid.util.con.a(playData, playerInfo, false, this.mPlayerRecordAdapter, 0);
                a2.eV(isIgnoreFetchLastTimeSave());
                int generateCupidVvId = CupidAdUtils.generateCupidVvId(a2, playData.getPlayScene());
                com.iqiyi.video.qyplayersdk.cupid.com4 com4Var = this.mAd;
                if (com4Var != null) {
                    com4Var.la(generateCupidVvId); // 更新当前的广告ID
                }
                org.qiyi.android.coreplayer.d.aux.boe();
                i = generateCupidVvId;
            }
            // a3 存储广告信息
            com.iqiyi.video.qyplayersdk.core.data.model.com1 a3 = com.iqiyi.video.qyplayersdk.core.data.a.aux.a(this.mSigt, i, playData, playerInfo, str, this.mControlConfig);
            com.iqiyi.video.qyplayersdk.g.aux.d("PLAY_SDK", TAG, " performBigCorePlayback QYPlayerMovie=", a3);
            this.mPlayerInfo = new PlayerInfo.Builder().copyFrom(playerInfo).extraInfo(new PlayerExtraInfo.Builder().copyFrom(playerInfo.getExtraInfo()).sigt(a3.getSigt()).build()).build();
            // 通知播放器信息已更改（在这里是指开始播放广告）
            notifyPlayerInfoChanged();
            // 判断是否断网
            if (!isNeedNetworkInterceptor(playerInfo)) {
                if (playData == null || (TextUtils.isEmpty(playData.getPlayAddress()) && (TextUtils.isEmpty(playData.getTvId()) || "0".equals(playData.getTvId())))) {
                    PlayerExceptionTools.report(0, 0.1f, "1", com.iqiyi.video.qyplayersdk.player.data.b.con.i(playData));
                }
                com.iqiyi.video.qyplayersdk.core.com1 com1Var = this.mPlayerCore;
                if (com1Var != null) {
                    com1Var.setVideoPath(a3);
                    this.mPlayerCore.ahF();
                }
            }
            org.qiyi.android.coreplayer.d.com7.endSection();
        }
    }
```

```java
// 停止视频
public void amX() {
    d dVar = this.mQYMediaPlayer;
    if (dVar != null) {
        dVar.stopPlayback();
    }
}
 
// 判断是否获取到视频
public static boolean A(PlayerInfo playerInfo) {
    return z(playerInfo) || y(playerInfo);
}
 
// 获取PlayerExtraInfo对象的播放地址和播放地址类型
public static boolean z(PlayerInfo playerInfo) {
    if (playerInfo == null || playerInfo.getExtraInfo() == null) {
        return false;
    }
    PlayerExtraInfo extraInfo = playerInfo.getExtraInfo();
    String playAddress = extraInfo.getPlayAddress();
    int playAddressType = extraInfo.getPlayAddressType();
    if (TextUtils.isEmpty(playAddress)) {
        return false;
    }
    return playAddressType == 9 || playAddressType == 4 || playAddressType == 8;
}
 
// 判断是否有视频和专辑ID
public static boolean y(PlayerInfo playerInfo) {
    String s = s(playerInfo);     // 专辑ID
    String u = u(playerInfo);     // 视频ID
    if ((TextUtils.isEmpty(s) || TextUtils.equals(s, "0")) && !((!TextUtils.isEmpty(u) && !TextUtils.equals(u, "0")) || playerInfo == null || playerInfo.getExtraInfo() == null)) {
        // 获取PlayerExtraInfo对象的播放地址和播放地址类型
                PlayerExtraInfo extraInfo = playerInfo.getExtraInfo();
        return !TextUtils.isEmpty(extraInfo.getPlayAddress()) && extraInfo.getPlayAddressType() == 6;
    }
    return false;
}
 
// 获取专辑ID
public static String s(PlayerInfo playerInfo) {
    String id;
    return (playerInfo == null || playerInfo.getAlbumInfo() == null || (id = playerInfo.getAlbumInfo().getId()) == null) ? "" : id;
}
 
// 获取视频ID
public static String u(PlayerInfo playerInfo) {
    String id;
    return (playerInfo == null || playerInfo.getVideoInfo() == null || (id = playerInfo.getVideoInfo().getId()) == null) ? "" : id;
}
 
// 一个广告控制器方法，用于更新当前的CupidvvId
public void la(int i) {
        // col=0，则说明当前没有活跃的vvId，打印日志信息表示要更新当前的vvId
    if (this.col.getAndIncrement() == 0) {
        com.iqiyi.video.qyplayersdk.g.aux.i("PLAY_SDK_AD_MAIN", "{AdsController}", " update current cupid vvId. current doesn't has active vvId.");
    } else {
        com.iqiyi.video.qyplayersdk.g.aux.i("PLAY_SDK_AD_MAIN", "{AdsController}", " update current cupid vvId. but current has active vvId.");
        // 将旧的vvId赋值给coh变量
                this.coh = this.coi;
    }
        // 将当前新的ID赋给coi
    this.coi = i;
    lc(i);
    com5.aux auxVar = this.mQYAdPresenter;
    if (auxVar != null) {
        auxVar.lh(i);    // 为暂停播放函数与继续播放函数传递广告ID
    }
}
 
/*
        该方法用于注册广告委托和委托JSON，以展示广告
        通过 qYPlayerADConfig3.checkRegister 方法判断是否需要注册广告
        通过 Cupid.registerObjectAppDelegate 方法注册代理
        广告类型包括：
        中插广告（SlotType.SLOT_TYPE_BRIEF_ROLL）、
        viewpoint广告（SlotType.SLOT_TYPE_VIEWPOINT）、
        页面广告（SlotType.SLOT_TYPE_PAGE）等等
*/
private void lc(final int i) {
    com.iqiyi.video.qyplayersdk.g.aux.d("PLAY_SDK_AD_CORE", "{AdsController}", "; registerCupidJsonDelegate vvId:", Integer.valueOf(i), "");
 
        org.qiyi.android.coreplayer.d.aux.wr(com.qiyi.baselib.utils.d.nul.fJ(org.iqiyi.video.mode.com3.enn) ? 2 : 1);
        ...
        QYPlayerADConfig qYPlayerADConfig5 = this.cog;
      if (qYPlayerADConfig5.checkRegister(256, qYPlayerADConfig5.getAddAdPolicy())) {
          QYPlayerADConfig qYPlayerADConfig6 = this.cog;
          if (!qYPlayerADConfig6.checkRegister(256, qYPlayerADConfig6.getRemoveAdPolicy())) {
              Cupid.registerJsonDelegate(i, SlotType.SLOT_TYPE_VIEWPOINT.value(), this.cof);
          }
      }
        ...
}
```

我们可以发现这个函数就是判断是否显示广告界面的函数，可以猜测只有当是VIP账户时，播放数据（playData）才为空，才会使 i = 0（广告ID为0）。

```java
// 视频播放结束后，继续获取视频的相关信息。
public void doVPlayAfterPlay(final PlayData playData) {
    performBigCorePlayback(playData);
    lpt6 lpt6Var = this.mTaskExecutor;
    if (lpt6Var != null) {
        lpt6Var.q(new Runnable() { // from class: com.iqiyi.video.qyplayersdk.player.QYMediaPlayerProxy.1
            @Override // java.lang.Runnable
            public void run() {
                QYMediaPlayerProxy.this.requestVplayInfo(playData);
            }
        });
    }
}
 
// 在获取视频源前获取一些与视频相关的信息
private void doVPlayBeforePlay(PlayData playData, boolean z) {
    VPlayParam a2 = com.iqiyi.video.qyplayersdk.player.data.b.con.a(playData, VPlayHelper.CONTENT_TYPE_PLAY_CONDITION, this.mPassportAdapter);
    this.mVPlayHelper.cancel();
        // 请求 VPlay 信息
    this.mVPlayHelper.requestVPlay(this.mContext, a2, new aux(this, playData, this.mSigt, z), this.mBigcoreVplayInterceptor);
    sendVPlayRequestPingback(true, playData, this.mSigt);
    com.iqiyi.video.qyplayersdk.b.com3.b(playData);
    com.iqiyi.video.qyplayersdk.g.aux.d("PLAY_SDK", TAG, " doVPlayBeforePlay needRequestFull=", Boolean.valueOf(z));
}
 
// 判断是否需要网络拦截
private boolean isNeedNetworkInterceptor(PlayerInfo playerInfo) {
        // 是否需要忽略用户代理的拦截
    if (ignoreNetworkInterceptByUA()) {
        com.iqiyi.video.qyplayersdk.g.aux.d("PLAY_SDK", TAG, "ignoreNetworkInterceptByUA ");
        return false;
    }
 
        // 判断当前是否处于离线状态，并且要播放的视频是在线视频
    boolean gW = org.iqiyi.video.l.aux.gW(this.mContext);
    boolean D = com.iqiyi.video.qyplayersdk.player.data.b.nul.D(playerInfo);
    if (gW && D) {
                // 获取当前的错误码版本号，根据不同的版本号来执行不同的逻辑
        int errorCodeVersion = getErrorCodeVersion();
        com.iqiyi.video.qyplayersdk.g.aux.d("PLAY_SDK", TAG, "isNeedNetworkInterceptor isOffNetWork = ", Boolean.valueOf(gW), " isOnLineVideo = ", Boolean.valueOf(D), " errorCodeVer = " + errorCodeVersion);
 
                if (errorCodeVersion == 1) {
                        // 自定义错误码为900400的播放器错误
            this.mInvokerQYMediaPlayer.onError(PlayerError.createCustomError(900400, "current network is offline, but you want to play online video"));
            return true;    // 进行网络拦截
 
        } else if (errorCodeVersion == 2) { 
                        // 返回错误码和错误信息
            org.iqiyi.video.data.com7 bbQ = org.iqiyi.video.data.com7.bbQ();
            bbQ.xC(String.valueOf(900400));
            bbQ.setDesc("current network is offline, but you want to play online video");
            this.mInvokerQYMediaPlayer.onErrorV2(bbQ);
            return true;
        }
    }
    return false;     // 不需要进行网络拦截
}
```

从这里得到处理的方法

```java
else {
            org.qiyi.android.coreplayer.d.com7.beginSection("QYMediaPlayerProxy.performBigCorePlayback");
            // 通过判断播放数据（playData）是否为空以及是否存在播放地址，空则i = 0。
            if (com.iqiyi.video.qyplayersdk.player.data.b.nul.A(playerInfo) || playData == null) {
                i = 0;
            } else {
                // 如果有地址，根据该数据生成CupidVvId，并将该ID与广告相关的Ad对象（mAd）绑定。
                        // 所以这里就是去后台获取广告的id
                com.iqiyi.video.qyplayersdk.cupid.data.model.com9 a2 = com.iqiyi.video.qyplayersdk.cupid.util.con.a(playData, playerInfo, false, this.mPlayerRecordAdapter, 0);
                a2.eV(isIgnoreFetchLastTimeSave());
                int generateCupidVvId = CupidAdUtils.generateCupidVvId(a2, playData.getPlayScene());
                com.iqiyi.video.qyplayersdk.cupid.com4 com4Var = this.mAd;
                if (com4Var != null) {
                    com4Var.la(generateCupidVvId); // 更新当前的广告ID
                }
                org.qiyi.android.coreplayer.d.aux.boe();
                i = generateCupidVvId;
            }
```

可以猜测只有当是VIP账户时，播放数据（playData）才为空，才会使 i = 0（广告ID为0）。

所以我们需要对其条件进行修改

![image-20230417211250002](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417211250002.png)

将这里的eqz修改为nez即可，只要两个条件修改一个即可，因为两者是或的区别，我们只需要将一个逻辑取反

重新打包编译

![image-20230417212417188](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417212417188.png)

通过adb重新传到测试机上并进行测试，一个好的结果，广告被去除了。

![image-20230417212314941](C:\Users\33551\AppData\Roaming\Typora\typora-user-images\image-20230417212314941.png)