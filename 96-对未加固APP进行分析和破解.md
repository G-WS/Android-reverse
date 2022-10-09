# 对未加固APP进行分析和破解

参考《安卓Frida逆向与抓包实战》

目标是去除一个未加固的APP的升级提示弹窗zhibo.apk

首先确保手机处于USB调试状态

```sh
adb devices
```

APK文件位于我的仓库中的apk文件夹下

首先安装apk文件到测试机

```
(base) ┌──(root㉿kali)-[/home/zhy/桌面/chap5]
└─# adb install zhibo.apk 
Performing Streamed Install
Success
```

通过Jadx打开APP，搜索关键词“升级”这里在搜索关键词中只得到一个结果，十分低效，一旦App进行 了加固处理或者关键词在App中出现结果过多，那么搜索关键词的这 个方法将使得分析者消耗大量的时间在无关的代码上，正确的方法应 该是从开发者的角度思考。

![](https://github.com/G-WS/Android-reverse/blob/main/image/%E6%9C%AA%E5%8A%A0%E5%9B%BA1.png?raw=true)

App的升级提示是以一个弹窗来实现 的 。 在 Android 中 常 用 的 实 现 弹 窗 的 类 主 要 有 三 种 ： android.App.Dialog 、 android.App.AlertDialog 和 android.widget.PopupWindow。

因此这里使用Objection对调用弹窗类的函数进行快速定位。

要使用Objection进行测试，首先需要获取App的包名。这里选 择使用Jadx反编译App并打开AndroidManifest.xml文件查看App包 名

```
<manifest xmlns:android="http://schemas.android.com/apk/res/android" android:versionCode="58" android:versionName="1.3.3.5" android:installLocation="auto" package="com.hd.zhibo" platformBuildVersionCode="14" platformBuildVersionName="4.0.2-238991">
```

在手机上启动frida

```
(base) ┌──(zhy㉿kali)-[~/桌面]
└─$ su      
密码：
(base) ┌──(root㉿kali)-[/home/zhy/桌面]
└─# adb shell                  
bullhead:/ $ su
bullhead:/ # cd data/local/tmp/                                                         
bullhead:/data/local/tmp # ls
frida-server-14.2.17-android-arm64 oat             
frida-server-15.2.2-android-arm64  re.frida.server 
bullhead:/data/local/tmp # ./f
frida-server-14.2.17-android-arm64          frida-server-15.2.2-android-arm64
/frida-server-15.2.2-android-arm64                                                     <
```

```
┌──(root💀r0env)-[~/Desktop]
└─# objection -g com.hd.zhibo explore
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
com.hd.zhibo on (google: 8.1.0) [usb] # android heap search instances android.App.AlertDialog
com.hd.zhibo on (google: 8.1.0) [usb] # android heap search instances android.App.AlertDialog
com.hd.zhibo on (google: 8.1.0) [usb] # android heap search instances android.app.AlertDialog
Class instance enumeration complete for android.app.AlertDialog
com.hd.zhibo on (google: 8.1.0) [usb] # android heap search instances android.app.AlertDialog
Class instance enumeration complete for android.app.AlertDialog
com.hd.zhibo on (google: 8.1.0) [usb] # android heap search instances android.app.Dialog
Class instance enumeration complete for android.app.Dialog
com.hd.zhibo on (google: 8.1.0) [usb] # android heap search instances android.widget.PopupWindow
Class instance enumeration complete for android.widget.PopupWindow
  Hashcode  Class                       toString()
----------  --------------------------  ---------------------------------
   8378703  android.widget.PopupWindow  android.widget.PopupWindow@7fd94f

```

进一步确认弹窗类型用一个Objection的插件WallBreaker（ 代 码 仓 库 地 址 为 https://github.com/hluwa/Wallbreaker ）

。要使用Wallbreaker，仅需 将Wallbreaker下载到本地，再使用Objection加载插件的命令将其 加载即可，具体命令如下：

```
(base) ┌──(root㉿kali)-[/home/zhy/桌面]
└─# git clone https://github.com/hluwa/Wallbreaker ~/.objection/plugins/Wallbreaker

```

通过-P命令在注入应用时加载插件

```
(base) ┌──(root㉿kali)-[/home/zhy/桌面]
└─# objection -g com.hd.zhibo explore -P ~/.objection/plugins/
Using USB device `Nexus 5X`
Agent injected and responds ok!
Loaded plugin: wallbreaker

     _   _         _   _
 ___| |_|_|___ ___| |_|_|___ ___
| . | . | | -_|  _|  _| | . |   |
|___|___| |___|___|_| |_|___|_|_|
      |___|(object)inject(ion) v1.11.0

     Runtime Mobile Exploration
        by: @leonjza from @sensepost

[tab] for command suggestions
com.hd.zhibo on (google: 8.1.0) [usb] # 

```

插件中只要使用两个命令

> objectsearch命令和Objection的 heap search命令的作用是相同的；objectdump命令则是用于将特 定实例中的具体内容打印出来。

以android.App.AlertDialog为例，先用objectsearch命令搜索 相关的实例；在搜索到实例后，再使用打印出来的十六进制值执行 objectdump命令。在第一步执行完objectsearch命令后就可以发现 内存中存在一个android.App.AlertDialog实例，在搜索到实例后再 使用这一步得到的十六进制的handle：0x2582去执行objectdump 命令，便可将0x2582所代表实例的内容打印出来

在打印结果中，注意在“=>”符号后的都是相应变量对应的值。

```
(base) ┌──(root㉿kali)-[/home/zhy/桌面]
└─# objection -g com.hd.zhibo explore -P ~/.objection/plugins/
Using USB device `Nexus 5X`
Agent injected and responds ok!
Loaded plugin: wallbreaker

     _   _         _   _
 ___| |_|_|___ ___| |_|_|___ ___
| . | . | | -_|  _|  _| | . |   |
|___|___| |___|___|_| |_|___|_|_|
      |___|(object)inject(ion) v1.11.0

     Runtime Mobile Exploration
        by: @leonjza from @sensepost

[tab] for command suggestions
com.hd.zhibo on (google: 8.1.0) [usb] # plugin wallbreaker objectsearch android.app.AlertDialog
com.hd.zhibo on (google: 8.1.0) [usb] # plugin wallbreaker objectsearch android.widget.PopupWindow
[0x217a]: android.widget.PopupWindow@303bde5
com.hd.zhibo on (google: 8.1.0) [usb] # 
[0x217a]: android.widget.PopupWindow@303bde5
com.hd.zhibo on (google: 8.1.0) [usb] # plugin wallbreaker objectdump 0x217a

package android.widget

class PopupWindow {

        /* static fields */
        static I[] ABOVE_ANCHOR_STATE_SET; => [0x46a2]: 16842922
        static int ANIMATION_STYLE_DEFAULT; => -1
        static int DEFAULT_ANCHORED_GRAVITY; => 8388659
        static int INPUT_METHOD_FROM_FOCUSABLE; => 0
        static int INPUT_METHOD_NEEDED; => 1
        static int INPUT_METHOD_NOT_NEEDED; => 2

        /* instance fields */
        boolean mAboveAnchor; => false
        Drawable mAboveAnchorBackgroundDrawable; => null
        boolean mAllowScrollingAnchorParent; => true
        WeakReference mAnchor; => null
        WeakReference mAnchorRoot; => null
        int mAnchorXoff; => 0
        int mAnchorYoff; => 0
        int mAnchoredGravity; => 0
        int mAnimationStyle; => -1
        boolean mAttachedInDecor; => false
        boolean mAttachedInDecorSet; => true
        Drawable mBackground; => [0x466a]: android.graphics.drawable.ColorDrawable@91d0ba
        View mBackgroundView; => null
        Drawable mBelowAnchorBackgroundDrawable; => null
        boolean mClipToScreen; => false
        boolean mClippingEnabled; => true
        View mContentView; => [0x463a]: com.zhibo.widget.linearlayout{58abb6b V.E...... ......I. 0,0-0,0}
        Context mContext; => [0x460a]: com.zhibo.media.channel_main@ee96c8
        PopupWindow$PopupDecorView mDecorView; => null
        float mElevation; => 0
        Transition mEnterTransition; => null
        Rect mEpicenterBounds; => null
        Transition mExitTransition; => null
        boolean mFocusable; => true
        int mGravity; => 0
        int mHeight; => -2
        int mHeightMode; => 0
        boolean mIgnoreCheekPress; => false
        int mInputMethodMode; => 0
        boolean mIsAnchorRootAttached; => false
        boolean mIsDropdown; => false
        boolean mIsShowing; => false
        boolean mIsTransitioningToDismiss; => false
        int mLastHeight; => 0
        int mLastWidth; => 0
        boolean mLayoutInScreen; => false
        boolean mLayoutInsetDecor; => false
        boolean mNotTouchModal; => false
        View$OnAttachStateChangeListener mOnAnchorDetachedListener; => [0x45da]: android.widget.PopupWindow$1@e375361
        View$OnAttachStateChangeListener mOnAnchorRootDetachedListener; => [0x45e2]: android.widget.PopupWindow$2@6fc3e86
        PopupWindow$OnDismissListener mOnDismissListener; => null
        View$OnLayoutChangeListener mOnLayoutChangeListener; => [0x459a]: android.widget.-$Lambda$ISuHLqeK-K4pmesAfzlFglc3xF4@617a347
        ViewTreeObserver$OnScrollChangedListener mOnScrollChangedListener; => [0x456a]: android.widget.-$Lambda$ISuHLqeK-K4pmesAfzlFglc3xF4$1@bba5774
        boolean mOutsideTouchable; => false
        boolean mOverlapAnchor; => false
        WeakReference mParentRootView; => null
        boolean mPopupViewInitialLayoutDirectionInherited; => false
        int mSoftInputMode; => 1
        int mSplitTouchEnabled; => -1
        Rect mTempRect; => [0x453a]: Rect(0, 0 - 0, 0)
        I[] mTmpAppLocation; => [0x4542]: 0,0
        I[] mTmpDrawingLocation; => [0x452a]: 0,0
        I[] mTmpScreenLocation; => [0x451a]: 0,0
        View$OnTouchListener mTouchInterceptor; => null
        boolean mTouchable; => true
        int mWidth; => 495
        int mWidthMode; => 0
        int mWindowLayoutType; => 1000
        WindowManager mWindowManager; => [0x44da]: android.view.WindowManagerImpl@cb4c49d

        /* constructor methods */
        android.widget.PopupWindow();
        android.widget.PopupWindow(int, int);
        android.widget.PopupWindow(Context);
        android.widget.PopupWindow(Context, AttributeSet);
        android.widget.PopupWindow(Context, AttributeSet, int);
        android.widget.PopupWindow(Context, AttributeSet, int, int);
        android.widget.PopupWindow(View);
        android.widget.PopupWindow(View, int, int);
        android.widget.PopupWindow(View, int, int, boolean);

        /* static methods */
        static I[] -get0();
        static boolean -get1(PopupWindow);
        static WeakReference -get2(PopupWindow);
        static View$OnTouchListener -get3(PopupWindow);
        static boolean -set0(PopupWindow, boolean);
        static void -wrap0(PopupWindow);
        static void -wrap1(PopupWindow, View, ViewGroup, View);

        /* instance methods */
        void alignToAnchor();
        int computeAnimationResource();
        int computeFlags(int);
        int computeGravity();
        PopupWindow$PopupBackgroundView createBackgroundView(View);
        PopupWindow$PopupDecorView createDecorView(View);
        void dismissImmediate(View, ViewGroup, View);
        View getAppRootView(View);
        Transition getTransition(int);
        void invokePopup(WindowManager$LayoutParams);
        boolean positionInDisplayHorizontal(WindowManager$LayoutParams, int, int, int, int, int, boolean);
        boolean positionInDisplayVertical(WindowManager$LayoutParams, int, int, int, int, int, boolean);
        void preparePopup(WindowManager$LayoutParams);
        void setLayoutDirectionFromAnchor();
        boolean tryFitHorizontal(WindowManager$LayoutParams, int, int, int, int, int, int, int, boolean);
        boolean tryFitVertical(WindowManager$LayoutParams, int, int, int, int, int, int, int, boolean);
        void update(View, boolean, int, int, int, int);
        void update();
        void update(int, int);
        void update(int, int, int, int);
        void update(int, int, int, int, boolean);
        void update(View, int, int);
        void update(View, int, int, int, int);
        void update(View, WindowManager$LayoutParams);
        void -android_widget_PopupWindow-mthref-0();
        void attachToAnchor(View, int, int, int);
        WindowManager$LayoutParams createPopupLayoutParams(IBinder);
        void detachFromAnchor();
        void dismiss();
        boolean findDropDownPosition(View, WindowManager$LayoutParams, int, int, int, int, int, boolean);
        boolean getAllowScrollingAnchorParent();
        View getAnchor();
        int getAnimationStyle();
        Drawable getBackground();
        View getContentView();
        WindowManager$LayoutParams getDecorViewLayoutParams();
        float getElevation();
        Transition getEnterTransition();
        Transition getExitTransition();
        int getHeight();
        int getInputMethodMode();
        int getMaxAvailableHeight(View);
        int getMaxAvailableHeight(View, int);
        int getMaxAvailableHeight(View, int, boolean);
        PopupWindow$OnDismissListener getOnDismissListener();
        boolean getOverlapAnchor();
        int getSoftInputMode();
        Rect getTransitionEpicenter();
        int getWidth();
        int getWindowLayoutType();
        boolean hasContentView();
        boolean hasDecorView();
        boolean isAboveAnchor();
        boolean isAttachedInDecor();
        boolean isClippingEnabled();
        boolean isFocusable();
        boolean isLayoutInScreenEnabled();
        boolean isLayoutInsetDecor();
        boolean isOutsideTouchable();
        boolean isShowing();
        boolean isSplitTouchEnabled();
        boolean isTouchable();
        boolean isTransitioningToDismiss();
        void lambda$-android_widget_PopupWindow_9628(View, int, int, int, int, int, int, int, int);
        void setAllowScrollingAnchorParent(boolean);
        void setAnimationStyle(int);
        void setAttachedInDecor(boolean);
        void setBackgroundDrawable(Drawable);
        void setClipToScreenEnabled(boolean);
        void setClippingEnabled(boolean);
        void setContentView(View);
        void setDropDown(boolean);
        void setElevation(float);
        void setEnterTransition(Transition);
        void setEpicenterBounds(Rect);
        void setExitTransition(Transition);
        void setFocusable(boolean);
        void setHeight(int);
        void setIgnoreCheekPress();
        void setInputMethodMode(int);
        void setLayoutInScreenEnabled(boolean);
        void setLayoutInsetDecor(boolean);
        void setOnDismissListener(PopupWindow$OnDismissListener);
        void setOutsideTouchable(boolean);
        void setOverlapAnchor(boolean);
        void setShowing(boolean);
        void setSoftInputMode(int);
        void setSplitTouchEnabled(boolean);
        void setTouchInterceptor(View$OnTouchListener);
        void setTouchModal(boolean);
        void setTouchable(boolean);
        void setTransitioningToDismiss(boolean);
        void setWidth(int);
        void setWindowLayoutMode(int, int);
        void setWindowLayoutType(int);
        void showAsDropDown(View);
        void showAsDropDown(View, int, int);
        void showAsDropDown(View, int, int, int);
        void showAtLocation(IBinder, int, int, int);
        void showAtLocation(View, int, int, int);
        void updateAboveAnchor(boolean);


```

**AlertDialog中布存在的原因是手机卡在初始界面没有弹出弹窗**目前没有什么解决办法，可能是APK存在问题

由于Hook在函数被调用后再触发就没有任何作用了，而样本App 的升级提示弹窗在App刚进入主页面就弹出了，因此必须保证样本刚 启动函数便被Hook上了。Objection作为一个成熟的工具也提供了这 样 的 功 能 ， 只 需 在 Objection 注 入 App 时 加 上 参 数 --startupcommand或者-s，并在参数后加上要执行的命令即可，Objection的 注入命令如下：

```
(base) ┌──(root㉿kali)-[/home/zhy/桌面]
└─# objection -g com.hd.zhibo explore -s "android hooking watch class android.app.AlertDialog"
Using USB device `Nexus 5X`
Agent injected and responds ok!
Running a startup command... android hooking watch class android.app.AlertDialog
(agent) Hooking android.app.AlertDialog.-get0(android.app.AlertDialog)
(agent) Hooking android.app.AlertDialog.resolveDialogTheme(android.content.Context, int)
(agent) Hooking android.app.AlertDialog.getButton(int)
(agent) Hooking android.app.AlertDialog.getListView()
(agent) Hooking android.app.AlertDialog.onCreate(android.os.Bundle)
(agent) Hooking android.app.AlertDialog.onKeyDown(int, android.view.KeyEvent)
(agent) Hooking android.app.AlertDialog.onKeyUp(int, android.view.KeyEvent)
(agent) Hooking android.app.AlertDialog.setButton(int, java.lang.CharSequence, android.content.DialogInterface$OnClickListener)
(agent) Hooking android.app.AlertDialog.setButton(int, java.lang.CharSequence, android.os.Message)
(agent) Hooking android.app.AlertDialog.setButton(java.lang.CharSequence, android.content.DialogInterface$OnClickListener)
(agent) Hooking android.app.AlertDialog.setButton(java.lang.CharSequence, android.os.Message)
(agent) Hooking android.app.AlertDialog.setButton2(java.lang.CharSequence, android.content.DialogInterface$OnClickListener)
(agent) Hooking android.app.AlertDialog.setButton2(java.lang.CharSequence, android.os.Message)
(agent) Hooking android.app.AlertDialog.setButton3(java.lang.CharSequence, android.content.DialogInterface$OnClickListener)
(agent) Hooking android.app.AlertDialog.setButton3(java.lang.CharSequence, android.os.Message)
(agent) Hooking android.app.AlertDialog.setButtonPanelLayoutHint(int)
(agent) Hooking android.app.AlertDialog.setCustomTitle(android.view.View)
(agent) Hooking android.app.AlertDialog.setIcon(int)
(agent) Hooking android.app.AlertDialog.setIcon(android.graphics.drawable.Drawable)
(agent) Hooking android.app.AlertDialog.setIconAttribute(int)
(agent) Hooking android.app.AlertDialog.setInverseBackgroundForced(boolean)
(agent) Hooking android.app.AlertDialog.setMessage(java.lang.CharSequence)
(agent) Hooking android.app.AlertDialog.setTitle(java.lang.CharSequence)
(agent) Hooking android.app.AlertDialog.setView(android.view.View)
(agent) Hooking android.app.AlertDialog.setView(android.view.View, int, int, int, int)
(agent) Registering job 391415. Type: watch-class for: android.app.AlertDialog

     _   _         _   _
 ___| |_|_|___ ___| |_|_|___ ___
| . | . | | -_|  _|  _| | . |   |
|___|___| |___|___|_| |_|___|_|_|
      |___|(object)inject(ion) v1.11.0

     Runtime Mobile Exploration
        by: @leonjza from @sensepost

[tab] for command suggestions
com.hd.zhibo on (google: 8.1.0) [usb] # 


```

调用栈得到样本App为了弹窗所创建的函数为 com.zhibo.media.channel_main. update_show()。使用Jadx观察 相应的函数内容

Jadx中升级是否提示弹窗取决于外层的if判断条件是否全部满足，所以想让升级提示弹窗不在弹出，就需要修改对应的判断语句并重新打包。

```java
public void update_show(Bundle bundle) {
        if (bundle == null || !bundle.containsKey("ver") || !bundle.containsKey("info") || !bundle.containsKey("path")) {
            return;
        }
        new AlertDialog.Builder(this).setTitle("发现新版本 " + bundle.getString("ver") + " 是否升级").setMessage(bundle.getString("info")).setPositiveButton("立刻升级", new o(this, bundle)).show();
    }
}
```

通过apktools将apk文件进行反编译，具体命令以及结果显示如下

```
(base) ┌──(root㉿kali)-[/home/zhy/桌面/chap5]
└─# apktool d zhibo.apk        
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
I: Using Apktool 2.6.1 on zhibo.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /root/.local/share/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...

```

update_show()的smali文件如下：

可以通过ctrl f快速搜索。

```
method public update_show(Landroid/os/Bundle;)V
    .locals 3

    if-eqz p1, :cond_0

    const-string v0, "ver"

    invoke-virtual {p1, v0}, Landroid/os/Bundle;->containsKey(Ljava/lang/String;)Z

    move-result v0

    if-eqz v0, :cond_0

    const-string v0, "info"

    invoke-virtual {p1, v0}, Landroid/os/Bundle;->containsKey(Ljava/lang/String;)Z

    move-result v0

    if-eqz v0, :cond_0

    const-string v0, "path"

    invoke-virtual {p1, v0}, Landroid/os/Bundle;->containsKey(Ljava/lang/String;)Z

    move-result v0

    if-eqz v0, :cond_0

    new-instance v0, Landroid/app/AlertDialog$Builder;

    invoke-direct {v0, p0}, Landroid/app/AlertDialog$Builder;-><init>(Landroid/content/Context;)V

    new-instance v1, Ljava/lang/StringBuilder;

    const-string v2, "\u53d1\u73b0\u65b0\u7248\u672c "

    invoke-direct {v1, v2}, Ljava/lang/StringBuilder;-><init>(Ljava/lang/String;)V

    const-string v2, "ver"

    invoke-virtual {p1, v2}, Landroid/os/Bundle;->getString(Ljava/lang/String;)Ljava/lang/String;

    move-result-object v2

    invoke-virtual {v1, v2}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    move-result-object v1

    const-string v2, " \u662f\u5426\u5347\u7ea7"

    invoke-virtual {v1, v2}, Ljava/lang/StringBuilder;->append(Ljava/lang/String;)Ljava/lang/StringBuilder;

    move-result-object v1

    invoke-virtual {v1}, Ljava/lang/StringBuilder;->toString()Ljava/lang/String;

    move-result-object v1

    invoke-virtual {v0, v1}, Landroid/app/AlertDialog$Builder;->setTitle(Ljava/lang/CharSequence;)Landroid/app/AlertDialog$Builder;

    move-result-object v0

    const-string v1, "info"

    invoke-virtual {p1, v1}, Landroid/os/Bundle;->getString(Ljava/lang/String;)Ljava/lang/String;

    move-result-object v1

    invoke-virtual {v0, v1}, Landroid/app/AlertDialog$Builder;->setMessage(Ljava/lang/CharSequence;)Landroid/app/AlertDialog$Builder;

    move-result-object v0

    const-string v1, "\u7acb\u523b\u5347\u7ea7"

    new-instance v2, Lcom/zhibo/media/o;

    invoke-direct {v2, p0, p1}, Lcom/zhibo/media/o;-><init>(Lcom/zhibo/media/channel_main;Landroid/os/Bundle;)V

    invoke-virtual {v0, v1, v2}, Landroid/app/AlertDialog$Builder;->setPositiveButton(Ljava/lang/CharSequence;Landroid/content/DialogInterface$OnClickListener;)Landroid/app/AlertDialog$Builder;

    move-result-object v0

    invoke-virtual {v0}, Landroid/app/AlertDialog$Builder;->show()Landroid/app/AlertDialog;

    :cond_0
    return-void
.end method
```

结合反编出的java代码和smali代码进行分析，我们会发现只需要将if判断语句中的任意一个逻辑置反即可阻止最终升级提示窗口的弹出。笔 者选择修改if-eqz p1, :cond_0（如果p1寄存器中的内容为0就跳转 到:cond_0处）。这里将if-eqz改为if-nez（如果p1寄存器中的内容不 为0就跳转到:cond_0处）

再将APK文件重新打包

```
(base) ┌──(root㉿kali)-[/home/zhy/桌面/chap5]
└─# apktool b zhibo    
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
I: Using Apktool 2.6.1
I: Checking whether sources has changed...
I: Smaling smali folder into classes.dex...
I: Checking whether resources has changed...
I: Building resources...
I: Copying libs... (/lib)
I: Building apk file...
I: Copying unknown files/dir...
I: Built apk...
                     
```

重打包后的App会保存在dist目录下。此时二次打包的应用并不 能直接安装到手机上，这是因为在Android上运行的App都是需要签 名的。

为此需要使用jarsigner或者其他Android认可的签名工具生成一个签名文件，并使用生成的签名文件对打包好的App进行签名，具体命令和结果如下：

```
(base) ┌──(root㉿kali)-[/home/…/桌面/chap5/zhibo/dist]
└─# keytool -genkey -alias abc.keystore -keyalg RSA -validity 2000 -keystore abc.key
Picked up _JAVA_OPTIONS: -Dawt.useSystemAAFontSettings=on -Dswing.aatext=true
输入密钥库口令:  
再次输入新口令: 
您的名字与姓氏是什么?
  [Unknown]:  zhy
您的组织单位名称是什么?
  [Unknown]:  xdu
您的组织名称是什么?
  [Unknown]:  xdu
您所在的城市或区域名称是什么?
  [Unknown]:  xian
您所在的省/市/自治区名称是什么?
  [Unknown]:  shanxi
该单位的双字母国家/地区代码是什么?
  [Unknown]:  86
CN=zhy, OU=xdu, O=xdu, L=xian, ST=shanxi, C=86是否正确?
  [否]:  
您的名字与姓氏是什么?
  [zhy]:  zhy
您的组织单位名称是什么?
  [xdu]:  xdu
您的组织名称是什么?
  [xdu]:  xdu
您所在的城市或区域名称是什么?
  [xian]:  xian
您所在的省/市/自治区名称是什么?
  [shanxi]:  shanxi
该单位的双字母国家/地区代码是什么?
  [86]:  86
CN=zhy, OU=xdu, O=xdu, L=xian, ST=shanxi, C=86是否正确?
  [否]:  y
电脑上没有jarsigner直接用AndroidKiller打包
```

