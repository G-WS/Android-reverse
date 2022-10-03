# Objection入门

## 0x01 Objection介绍

Frida时提供的各种API基础之上可以实现无数的具体功能，Objection时一个将各种常用功能整合进工具中提供我们直接在命令行使用的利器，Objection甚至可以不写一行代码就能进行APP的逆向分析。

Objection集成的功能主要支持Android和iOS两大移动平台。在 对Android的支持中，Objection可以快速完成诸如内存搜索、类和 模块搜索、方法Hook以及打印参数、返回值、调用栈等常用功能， 是一个非常方便的逆向必备工具和内存漫游神器。

Objection主要有三大组成部分。

 第一部分是指Objection重打包的相关组件。Objection可以将 Frida运行时所需要的frida-gadget.so重打包进App中，从而完成 Frida的无root调试。 

第二部分是指Objection本身。Objection是一个Python的pypi 包，可以和包含frida-gadget.so这个so文件的App进行交互，运行 Frida的Hook脚本，并分析Hook的结果。

第三部分是指Objection从TypeScript项目编译而成的一个 agent.js文件。该文件在App运行过程中插了Frida运行库，使得 Objection支持的所有功能成为可能。

Objection依托Frida完成了对应用的注入以及对函 数的Hook模板，使用时只需要将具体的类填充进去即可完成相应的 Hook测试，是一个非常好用的逆向工具。



## 0x02 Objection安装与使用

### 1、Objection的运行条件

（1）、python的版本必须大于3.4

可以通过以下命令确认python的版本：

```sh
python -V
```

（2）、Python包的管理软件pip的版本大于9.0。可通过pip查看版本的命令来确认版本：

```shell
pip --version
```

### 2、安装

使用pip命令安装Objection

官网的建议时直接执行以下命令

```sh
pip3 install -U objection
```

但是由于Frida的版本更新过快，不同版本的Frida支持的版本库可能会不同，所以尽量选择Objection在Frida相应版本发布后更新的版本，且发布版本实践应当尽量与Frida相应版本靠近。可以通过pypi官网查看Objection不同版本的发布时间，同时使用GitHub Frida的仓库查看相应的Frida发布时间进行对比确认。

我使用的时Frida的 15.2.2是2022年 Jul 22，最近的Objection版本时2021年的Apr 6。

那么先尝试一下能不能用。

### 3、使用

Objection默认通过USB连接设备，这里不必和 Frida的命令行一样通过-U参数指定USB模式连接，同时主要通过-g 参数指定注入的进程并通过explore命令进入REPL模式。在进入 REPL模式后便可以使用Objection进行Hook的常用命令，这也是本 章中所要介绍的重点。

Objection支持通过-N参数来指定网络中 的设备并通过-h参数和-p参数来指定对应设备的IP和端口以进行连 接，从而完成对网络设备的注入与Hook。除此之外还可以通过 patchapk命令将frida-gadget.so打包进App等。

以Android系统的基本应用“设置”为例来介绍Objection 的REPL模式常用命令。首先，在确认手机使用USB线连接上手机 后，运行相应版本的frida-server，运行“设置”应用以通过frida-ps 找到对应的App及其包名，具体操作如下：

```
frida-ps -U|grep setting 
```

但是在我的测试机中没有反应，通过

```
frida-ps -U
```

命令发现设置应用的进程名就是设置，通过objection注入设置应用，注入成功后便进入了Objection的REPL界面，具体操作命令和结果如下：

```sh
objection -g 设置 explore
```



```sh
(base) ┌──(root㉿kali)-[/home/zhy/桌面]
└─# objection -g 设置 explore
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
com.android.settings on (google: 8.1.0) [usb] # 

```

在学习Objection的REPL界面命令之前，首先要了解空格键的作 用。在Objection REPL界面中，当不知道命令时通过按空格键就会 提示可用的命令，在出现提示后通过上下选择键及回车键便可以输入 命令

### 4、命令学习

#### 4.1 help命令

不知道当前命令的效果是什么时，在当前命令 前加help（比如help env）再回车之后就会出现当前命令的解释信息

#### 4.2 jobs命令

作业系统很好用，用于查看和管理当前所执行 Hook的任务，建议一定要掌握，可以同时运行多项Hook作业。

#### 4.3 Frida命令

Frida命令。查看Frida相关信息

Frida版本过高会导致打印相关信息报错。可以考虑对Frida进行降版本处理。

#### 4.4 内存漫游相关命令

Objection可以快速便捷地打印出内存 中的各种类的相关信息，这对App快速定位有着无可比拟的优势，下 面介绍几个常用命令。

①可以使用以下命令列出内存中的所有类：

```sh
 android hooking list classes
```

```sh
com.android.settings on (google: 8.1.0) [usb] # android hooking list classes
[B
[C
[D
[F
[I
[J
[Landroid.animation.Animator;
[Landroid.animation.Keyframe$FloatKeyframe;
[Landroid.animation.Keyframe$IntKeyframe;
[Landroid.animation.PropertyValuesHolder;
[Landroid.app.FragmentState;
[Landroid.app.LoaderManagerImpl;
[Landroid.bluetooth.BluetoothCodecConfig;
[Landroid.content.pm.ActivityInfo;
[Landroid.content.pm.ConfigurationInfo;
[Landroid.content.pm.FeatureGroupInfo;
[Landroid.content.pm.FeatureInfo;
[Landroid.content.pm.InstrumentationInfo;
[Landroid.content.pm.PathPermission;
[Landroid.content.pm.PermissionInfo;
........
........
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

Found 5986 classes
```

②可以使用以下命令在内存中所有已加载的类中搜索包含特定关 键词的类

```sh
android hooking search classes 关键词
```

```sh
com.android.settings on (google: 8.1.0) [usb] # android hooking search classes display
Note that Java classes are only loaded when they are used, so if the expected class has not been found, it might not have been loaded yet.
[Landroid.icu.text.DisplayContext$Type;
[Landroid.icu.text.DisplayContext;
[Landroid.view.Display$Mode;
android.hardware.display.DisplayManager
android.hardware.display.DisplayManager$DisplayListener
android.hardware.display.DisplayManagerGlobal
android.hardware.display.DisplayManagerGlobal$DisplayListenerDelegate
android.hardware.display.DisplayManagerGlobal$DisplayManagerCallback
android.hardware.display.IDisplayManager
android.hardware.display.IDisplayManager$Stub
android.hardware.display.IDisplayManager$Stub$Proxy
android.hardware.display.IDisplayManagerCallback
android.hardware.display.IDisplayManagerCallback$Stub
android.hardware.display.WifiDisplay$1
android.hardware.display.WifiDisplaySessionInfo$1
android.hardware.display.WifiDisplayStatus$1
android.icu.impl.CurrencyData$CurrencyDisplayInfo
android.icu.impl.CurrencyData$CurrencyDisplayInfoProvider
android.icu.impl.ICUCurrencyDisplayInfoProvider
android.icu.impl.ICUCurrencyDisplayInfoProvider$ICUCurrencyDisplayInfo
android.icu.impl.ICUCurrencyDisplayInfoProvider$ICUCurrencyDisplayInfo$SpacingInfoSink
android.icu.text.CurrencyDisplayNames
android.icu.text.DisplayContext
android.icu.text.DisplayContext$Type
android.media.MediaRouter$WifiDisplayStatusChangedReceiver
android.media.RemoteDisplay
android.opengl.EGLDisplay
android.support.v14.preference.PreferenceFragment$OnPreferenceDisplayDialogCallback
android.support.v7.preference.PreferenceManager$OnDisplayPreferenceDialogListener
android.util.DisplayMetrics
android.view.Choreographer$FrameDisplayEventReceiver
android.view.Display
android.view.Display$HdrCapabilities
android.view.Display$HdrCapabilities$1
android.view.Display$Mode
android.view.Display$Mode$1
android.view.DisplayAdjustments
android.view.DisplayEventReceiver
android.view.DisplayInfo
android.view.DisplayInfo$1
android.view.DisplayListCanvas
android.view.SurfaceControl$PhysicalDisplayInfo
com.android.internal.app.NightDisplayController
com.android.internal.app.NightDisplayController$1
com.android.internal.app.NightDisplayController$Callback
com.android.internal.hardware.AmbientDisplayConfiguration
com.android.settings.DisplaySettings
com.android.settings.DisplaySettings$1
com.android.settings.Settings$AmbientDisplayPickupSuggestionActivity
com.android.settings.Settings$AmbientDisplaySuggestionActivity
com.android.settings.Settings$DisplaySettingsActivity
com.android.settings.Settings$NightDisplaySettingsActivity
com.android.settings.Settings$NightDisplaySuggestionActivity
com.android.settings.Settings$WifiDisplaySettingsActivity
com.android.settings.dashboard.conditional.NightDisplayCondition
com.android.settings.display.DensityPreference
com.android.settings.display.NightDisplaySettings
com.android.settings.wfd.WifiDisplaySettings
com.google.android.gles_jni.EGLDisplayImpl
javax.microedition.khronos.egl.EGLDisplay

Found 60 classes

```

③可以使用以下命令从内存中搜索所有包含关键词key的方法：

```
android hooking search methods <key>
```

```sh
com.android.settings on (google: 8.1.0) [usb] # android hooking search methods display
Note that Java classes are only loaded when they are used, so if the expected class has not been found, it might not have been loaded yet.
Warning, searching all classes may take some time and in some cases, crash the target application.
Continue? [y/N]: y
Found 5986 classes, searching methods (this may take some time)...
android.app.ActionBar.getDisplayOptions
android.app.ActionBar.setDefaultDisplayHomeAsUpEnabled
android.app.ActionBar.setDisplayHomeAsUpEnabled
android.app.ActionBar.setDisplayOptions
android.app.ActionBar.setDisplayOptions
android.app.ActionBar.setDisplayShowCustomEnabled
android.app.ActionBar.setDisplayShowHomeEnabled
android.app.ActionBar.setDisplayShowTitleEnabled
android.app.ActionBar.setDisplayUseLogoEnabled
android.app.Activity.dispatchMovedToDisplay
android.app.Activity.onMovedToDisplay
android.app.ActivityOptions.getLaunchDisplayId
android.app.ActivityOptions.setLaunchDisplayId
android.app.ActivityThread$ApplicationThread.scheduleActivityMovedToDisplay
.......
.......
```

④搜索到我们感兴趣的类后，可以使用以下命令查看关心的类的 所有方法：

```sh
android hooking list class_methods
```

```sh
com.android.settings on (google: 8.1.0) [usb] # android hooking list class_methods android.hardware.display.Di
splayManager
private android.view.Display android.hardware.display.DisplayManager.getOrCreateDisplayLocked(int,boolean)
private void android.hardware.display.DisplayManager.addAllDisplaysLocked(java.util.ArrayList<android.view.Display>,int[])
private void android.hardware.display.DisplayManager.addPresentationDisplaysLocked(java.util.ArrayList<android.view.Display>,int[],int)
public android.graphics.Point android.hardware.display.DisplayManager.getStableDisplaySize()
public android.hardware.display.VirtualDisplay android.hardware.display.DisplayManager.createVirtualDisplay(android.media.projection.MediaProjection,java.lang.String,int,int,int,android.view.Surface,int,android.hardware.display.VirtualDisplay$Callback,android.os.Handler,java.lang.String)
public android.hardware.display.VirtualDisplay android.hardware.display.DisplayManager.createVirtualDisplay(java.lang.String,int,int,int,android.view.Surface,int)
public android.hardware.display.VirtualDisplay android.hardware.display.DisplayManager.createVirtualDisplay(java.lang.String,int,int,int,android.view.Surface,int,android.hardware.display.VirtualDisplay$Callback,android.os.Handler)
public android.hardware.display.WifiDisplayStatus android.hardware.display.DisplayManager.getWifiDisplayStatus()
public android.view.Display android.hardware.display.DisplayManager.getDisplay(int)
public android.view.Display[] android.hardware.display.DisplayManager.getDisplays()
public android.view.Display[] android.hardware.display.DisplayManager.getDisplays(java.lang.String)
public void android.hardware.display.DisplayManager.connectWifiDisplay(java.lang.String)
public void android.hardware.display.DisplayManager.disconnectWifiDisplay()
public void android.hardware.display.DisplayManager.forgetWifiDisplay(java.lang.String)
public void android.hardware.display.DisplayManager.pauseWifiDisplay()
public void android.hardware.display.DisplayManager.registerDisplayListener(android.hardware.display.DisplayManager$DisplayListener,android.os.Handler)
public void android.hardware.display.DisplayManager.renameWifiDisplay(java.lang.String,java.lang.String)
public void android.hardware.display.DisplayManager.resumeWifiDisplay()
public void android.hardware.display.DisplayManager.startWifiDisplayScan()
public void android.hardware.display.DisplayManager.stopWifiDisplayScan()
public void android.hardware.display.DisplayManager.unregisterDisplayListener(android.hardware.display.DisplayManager$DisplayListener)

Found 21 method(s)
```

⑤以上介绍的都是最基础的一些Java类相关的内容。在Android 中，正如笔者在第2章中介绍的，四大组件的相关内容是非常值得关 注的，Objection在这方面也提供了支持，可以通过以下命令列出进 程所有的activity（活动）。

```
android hooking list activities
```

```sh
com.android.settings on (google: 8.1.0) [usb] # android hooking list activities
com.android.settings.ActivityPicker
com.android.settings.AirplaneModeVoiceActivity
com.android.settings.AllowBindAppWidgetActivity
com.android.settings.AppWidgetPickActivity
com.android.settings.BandMode
com.android.settings.ConfirmDeviceCredentialActivity
com.android.settings.CreateShortcut
com.android.settings.CredentialStorage
com.android.settings.CryptKeeper$FadeToBlack
com.android.settings.CryptKeeperConfirm$Blank
com.android.settings.DeviceAdminAdd
com.android.settings.DeviceAdminSettings
com.android.settings.Display
com.android.settings.DisplaySettings
com.android.settings.EncryptionInterstitial
com.android.settings.FallbackHome
com.android.settings.HelpTrampoline
com.android.settings.LanguageSettings
com.android.settings.ManageApplications
com.android.settings.MonitoringCertInfoActivity
com.android.settings.RadioInfo
com.android.settings.RegulatoryInfoDisplayActivity
com.android.settings.RemoteBugreportActivity
com.android.settings.RunningServices
com.android.settings.SecuritySettings
com.android.settings.SetFullBackupPassword
com.android.settings.SetProfileOwner
com.android.settings.Settings
com.android.settings.Settings
com.android.settings.Settings$AccessibilityDaltonizerSettingsActivity
com.android.settings.Settings$AccessibilitySettingsActivity
com.android.settings.Settings$AccountSyncSettingsActivity
com.android.settings.Settings$AdvancedAppsActivity
com.android.settings.Settings$AllApplicationsActivity
com.android.settings.Settings$AmbientDisplayPickupSuggestionActivity
com.android.settings.Settings$AmbientDisplaySuggestionActivity
com.android.settings.Settings$AndroidBeamSettingsActivity
com.android.settings.Settings$ApnEditorActivity
com.android.settings.Settings$ApnSettingsActivity
com.android.settings.Settings$AppAndNotificationDashboardActivity
com.android.settings.Settings$AppDrawOverlaySettingsActivity
com.android.settings.Settings$AppMemoryUsageActivity
com.android.settings.Settings$AppNotificationSettingsActivity
com.android.settings.Settings$AppPictureInPictureSettingsActivity
com.android.settings.Settings$AppWriteSettingsActivity
com.android.settings.Settings$AssistGestureSettingsActivity
com.android.settings.Settings$AutomaticStorageManagerSettingsActivity
com.android.settings.Settings$AvailableVirtualKeyboardActivity
com.android.settings.Settings$BatterySaverSettingsActivity
com.android.settings.Settings$BluetoothSettingsActivity
com.android.settings.Settings$CaptioningSettingsActivity
com.android.settings.Settings$ChannelNotificationSettingsActivity
com.android.settings.Settings$ChooseAccountActivity
com.android.settings.Settings$ConfigureNotificationSettingsActivity
com.android.settings.Settings$ConfigureWifiSettingsActivity
com.android.settings.Settings$ConnectedDeviceDashboardActivity
com.android.settings.Settings$CryptKeeperSettingsActivity
com.android.settings.Settings$DataUsageSummaryActivity
com.android.settings.Settings$DateTimeSettingsActivity
com.android.settings.Settings$DevelopmentSettingsActivity
com.android.settings.Settings$DeviceAdminSettingsActivity
com.android.settings.Settings$DeviceInfoSettingsActivity
com.android.settings.Settings$DisplaySettingsActivity
com.android.settings.Settings$DoubleTapPowerSuggestionActivity
com.android.settings.Settings$DoubleTwistSuggestionActivity
com.android.settings.Settings$DreamSettingsActivity
com.android.settings.Settings$EnterprisePrivacySettingsActivity
com.android.settings.Settings$FactoryResetActivity
com.android.settings.Settings$FingerprintEnrollSuggestionActivity
com.android.settings.Settings$HighPowerApplicationsActivity
com.android.settings.Settings$IccLockSettingsActivity
com.android.settings.Settings$ImeiInformationActivity
com.android.settings.Settings$KeyboardLayoutPickerActivity
com.android.settings.Settings$LanguageAndInputSettingsActivity
com.android.settings.Settings$LegacySupportActivity
com.android.settings.Settings$LocalePickerActivity
com.android.settings.Settings$LocationSettingsActivity
com.android.settings.Settings$ManageAppExternalSourcesActivity
com.android.settings.Settings$ManageApplicationsActivity
com.android.settings.Settings$ManageAssistActivity
com.android.settings.Settings$ManageDomainUrlsActivity
com.android.settings.Settings$ManageExternalSourcesActivity
com.android.settings.Settings$ManagedProfileSettingsActivity
com.android.settings.Settings$MemorySettingsActivity
com.android.settings.Settings$MobileDataUsageListActivity
com.android.settings.Settings$NetworkDashboardActivity
com.android.settings.Settings$NotificationAccessSettingsActivity
com.android.settings.Settings$NotificationAppListActivity
com.android.settings.Settings$NotificationStationActivity
com.android.settings.Settings$OverlaySettingsActivity
com.android.settings.Settings$PaymentSettingsActivity
com.android.settings.Settings$PhysicalKeyboardActivity
com.android.settings.Settings$PictureInPictureSettingsActivity
com.android.settings.Settings$PowerUsageSummaryActivity
com.android.settings.Settings$PrintJobSettingsActivity
com.android.settings.Settings$PrintSettingsActivity
com.android.settings.Settings$PrivacySettingsActivity
com.android.settings.Settings$PrivateVolumeForgetActivity
com.android.settings.Settings$PrivateVolumeSettingsActivity
com.android.settings.Settings$PublicVolumeSettingsActivity
com.android.settings.Settings$RunningServicesActivity
com.android.settings.Settings$SavedAccessPointsSettingsActivity
com.android.settings.Settings$ScreenLockSuggestionActivity
com.android.settings.Settings$SecuritySettingsActivity
com.android.settings.Settings$SimStatusActivity
com.android.settings.Settings$SoundSettingsActivity
com.android.settings.Settings$SpecialAccessSettingsActivity
com.android.settings.Settings$SpellCheckersSettingsActivity
com.android.settings.Settings$StatusActivity
com.android.settings.Settings$StorageDashboardActivity
com.android.settings.Settings$StorageUseActivity
com.android.settings.Settings$SwipeToNotificationSuggestionActivity
com.android.settings.Settings$SystemDashboardActivity
com.android.settings.Settings$TestingSettingsActivity
com.android.settings.Settings$TetherSettingsActivity
com.android.settings.Settings$TextToSpeechSettingsActivity
com.android.settings.Settings$TrustedCredentialsSettingsActivity
com.android.settings.Settings$UsageAccessSettingsActivity
com.android.settings.Settings$UserAndAccountDashboardActivity
com.android.settings.Settings$UserDictionarySettingsActivity
com.android.settings.Settings$UserSettingsActivity
com.android.settings.Settings$VpnSettingsActivity
com.android.settings.Settings$VrListenersSettingsActivity
com.android.settings.Settings$WallpaperSettingsActivity
com.android.settings.Settings$WebViewAppPickerActivity
com.android.settings.Settings$WifiAPITestActivity
com.android.settings.Settings$WifiCallingSettingsActivity
com.android.settings.Settings$WifiCallingSuggestionActivity
com.android.settings.Settings$WifiDisplaySettingsActivity
com.android.settings.Settings$WifiInfoActivity
com.android.settings.Settings$WifiP2pSettingsActivity
com.android.settings.Settings$WifiSettingsActivity
com.android.settings.Settings$WriteSettingsActivity
com.android.settings.Settings$ZenAccessSettingsActivity
com.android.settings.Settings$ZenModeEventRuleSettingsActivity
com.android.settings.Settings$ZenModeExternalRuleSettingsActivity
com.android.settings.Settings$ZenModePrioritySettingsActivity
com.android.settings.Settings$ZenModeScheduleRuleSettingsActivity
com.android.settings.Settings$ZenModeSettingsActivity
com.android.settings.Settings$ZenModeVisualInterruptionSettingsActivity
com.android.settings.SettingsLicenseActivity
com.android.settings.SetupEncryptionInterstitial
com.android.settings.ShowAdminSupportDetailsDialog
com.android.settings.SmsDefaultDialog
com.android.settings.SoundSettings
com.android.settings.SubSettings
com.android.settings.TetherProvisioningActivity
com.android.settings.TetherSettings
com.android.settings.UsageStatsActivity
com.android.settings.UsbSettings
com.android.settings.UserDictionarySettings
com.android.settings.WebViewImplementation
com.android.settings.accessibility.AccessibilitySettingsForSetupWizardActivity
com.android.settings.accounts.AddAccountSettings
com.android.settings.applications.InstalledAppDetails
com.android.settings.applications.InstalledAppDetailsTop
com.android.settings.applications.ManageApplications
com.android.settings.applications.StorageUse
com.android.settings.applications.autofill.AutofillPickerActivity
com.android.settings.applications.autofill.AutofillPickerTrampolineActivity
com.android.settings.backup.BackupSettingsActivity
com.android.settings.bluetooth.BluetoothPairingDialog
com.android.settings.bluetooth.BluetoothPermissionActivity
com.android.settings.bluetooth.BluetoothSettings
com.android.settings.bluetooth.DevicePickerActivity
com.android.settings.bluetooth.RequestPermissionActivity
com.android.settings.bluetooth.RequestPermissionHelperActivity
com.android.settings.datausage.AppDataUsageActivity
com.android.settings.development.AppPicker
com.android.settings.development.DevelopmentSettingsDisabledActivity
com.android.settings.deviceinfo.StorageWizardFormatConfirm
com.android.settings.deviceinfo.StorageWizardFormatProgress
com.android.settings.deviceinfo.StorageWizardInit
com.android.settings.deviceinfo.StorageWizardMigrate
com.android.settings.deviceinfo.StorageWizardMigrateConfirm
com.android.settings.deviceinfo.StorageWizardMigrateProgress
com.android.settings.deviceinfo.StorageWizardMoveConfirm
com.android.settings.deviceinfo.StorageWizardMoveProgress
com.android.settings.deviceinfo.StorageWizardReady
com.android.settings.deviceinfo.UsbModeChooserActivity
com.android.settings.fingerprint.FingerprintEnrollEnrolling
com.android.settings.fingerprint.FingerprintEnrollFindSensor
com.android.settings.fingerprint.FingerprintEnrollFinish
com.android.settings.fingerprint.FingerprintEnrollIntroduction
com.android.settings.fingerprint.FingerprintSettings
com.android.settings.fingerprint.FingerprintSuggestionActivity
com.android.settings.fingerprint.SetupFingerprintEnrollEnrolling
com.android.settings.fingerprint.SetupFingerprintEnrollFindSensor
com.android.settings.fingerprint.SetupFingerprintEnrollFinish
com.android.settings.fingerprint.SetupFingerprintEnrollIntroduction
com.android.settings.fuelgauge.BatterySaverModeVoiceActivity
com.android.settings.fuelgauge.PowerUsageSummary
com.android.settings.fuelgauge.RequestIgnoreBatteryOptimizations
com.android.settings.inputmethod.InputMethodAndSubtypeEnablerActivity
com.android.settings.inputmethod.UserDictionaryAddWordActivity
com.android.settings.nfc.HowItWorks
com.android.settings.nfc.PaymentDefaultDialog
com.android.settings.notification.NotificationAccessConfirmationActivity
com.android.settings.notification.RedactionInterstitial
com.android.settings.notification.RedactionSettingsStandalone
com.android.settings.notification.ZenModeVoiceActivity
com.android.settings.password.ChooseLockGeneric
com.android.settings.password.ChooseLockGeneric$InternalActivity
com.android.settings.password.ChooseLockPassword
com.android.settings.password.ChooseLockPattern
com.android.settings.password.ConfirmDeviceCredentialActivity
com.android.settings.password.ConfirmDeviceCredentialActivity$InternalActivity
com.android.settings.password.ConfirmLockPassword
com.android.settings.password.ConfirmLockPassword$InternalActivity
com.android.settings.password.ConfirmLockPattern
com.android.settings.password.ConfirmLockPattern$InternalActivity
com.android.settings.password.SetNewPasswordActivity
com.android.settings.password.SetupChooseLockGeneric
com.android.settings.password.SetupChooseLockPassword
com.android.settings.password.SetupChooseLockPattern
com.android.settings.qstile.DevelopmentTileConfigActivity
com.android.settings.search.SearchActivity
com.android.settings.sim.SimDialogActivity
com.android.settings.sim.SimPreferenceDialog
com.android.settings.support.NewDeviceIntroSuggestionActivity
com.android.settings.support.SupportDashboardActivity
com.android.settings.wallpaper.WallpaperSuggestionActivity
com.android.settings.wifi.RequestToggleWiFiActivity
com.android.settings.wifi.WifiConfigInfo
com.android.settings.wifi.WifiDialogActivity
com.android.settings.wifi.WifiNoInternetDialog
com.android.settings.wifi.WifiPickerActivity
com.android.settings.wifi.WifiScanModeActivity
com.android.settings.wifi.WifiSettings
com.android.settings.wifi.WifiStatusTest
com.google.android.libraries.hats20.SurveyPromptActivity
com.google.android.settings.backup.BackupSuggestionActivity
com.google.android.settings.external.ExternalSettingsTrampoline
com.google.android.settings.gestures.AssistGestureSuggestion
com.google.android.settings.gestures.assist.AssistGestureTrainingEnrollingActivity
com.google.android.settings.gestures.assist.AssistGestureTrainingFinishedActivity
com.google.android.settings.gestures.assist.AssistGestureTrainingIntroActivity
com.google.android.settings.gestures.assist.bubble.AssistGestureBubbleActivity

Found 238 classes
```

⑥可以通过以下命令列出进程所有的service。

```
android hooking list services
```

```sh
com.android.settings on (google: 8.1.0) [usb] # android hooking list services
android.bluetooth.BluetoothA2dp$2
com.android.settings.SettingsDumpService
com.android.settings.TetherService
com.android.settings.bluetooth.BluetoothPairingService

Found 4 classes
```

需要列出其他两个组件的信息时，只要将对应的地方更换为 receivers和providers即可

```
com.android.settings on (google: 8.1.0) [usb] # android hooking list receivers
com.android.settings.RestrictedSettingsFragment$1
com.android.settings.SettingsActivity$1
com.android.settings.SettingsInitialize
com.android.settings.TestingSettingsBroadcastReceiver
com.android.settings.bluetooth.BluetoothPairingRequest
com.android.settings.bluetooth.BluetoothPermissionRequest
com.android.settings.development.DevelopmentSettings$1
com.android.settings.development.DevelopmentSettings$2
com.android.settings.deviceinfo.StorageUnmountReceiver
com.android.settings.sim.SimSelectNotification
com.android.settings.users.ProfileUpdateReceiver
com.android.settingslib.bluetooth.BluetoothDiscoverableTimeoutReceiver
com.android.settingslib.drawer.SettingsDrawerActivity$PackageReceiver
com.google.android.settings.phenotype.PhenotypeBroadcastReceiver

Found 14 classes
```

providers组件目前没有发现支持

ps：猜测可能没有支持吗？

#### 4.5 Hook相关命令

作为Frida的核心功能，Hook总是不能绕 过的。同样地，Objection作为Frida优秀的开发工具，Hook相关的 命令是一定要实现的。事实上，Objection在这方面的表现确实令人 称赞。

通过以下命令对指定的方法进行Hook。

```sh
android hooking watch class_method <methodName>
```

这里选择Java中File类的构造函数进行Hook，结果如下：

在上述命令中，加上了 --dump-args 、 --dumpbacktrace、--dump-return三个参数，分别用于打印函数的参数、 调用栈以及返回值。这三个参数对逆向分析的帮助是非常大的：有些 函数的明文和密文非常有可能放在参数和返回值中，而打印调用栈可 以让分析者快速进行调用链的溯源。

```sh
com.android.settings on (google: 8.1.0) [usb] # android hooking watch class_method java.io.File.$init --dump-a
rgs --dump-backtrace --dump-return
```

另外需要注意的是，此时虽然只确定了Hook构造函数，但是默 认会Hook对应方法的所有重载。同时，在输出的最后一行显示 Registering job 7s9a29pxmt4，这表示这个Hook被作为一个“作 业”添加到Objection的作业系统中了，此时运行job list命令可以查 看到这个“作业”的相关信息，如图4-10所示。可以发现这里的Job ID对应的是7s9a29pxmt4，同时Hooks对应的6正是Hook的函数的 数量。

```sh
com.android.settings on (google: 8.1.0) [usb] # android hooking watch class_method java.io.File.$init --dump-a
rgs --dump-backtrace --dump-return
(agent) Attempting to watch class java.io.File and method $init.
(agent) Hooking java.io.File.$init(java.io.File, java.lang.String)
(agent) Hooking java.io.File.$init(java.lang.String)
(agent) Hooking java.io.File.$init(java.lang.String, int)
(agent) Hooking java.io.File.$init(java.lang.String, java.io.File)
(agent) Hooking java.io.File.$init(java.lang.String, java.lang.String)
(agent) Hooking java.io.File.$init(java.net.URI)
(agent) Registering job 042074. Type: watch-method for: java.io.File.$init
```

```sh
com.android.settings on (google: 8.1.0) [usb] # jobs list
Job ID  Hooks  Type
------  -----  ------------------------------------
042074      6  watch-method for: java.io.File.$init
```

在“设置”应用中的任意位置进行点击时，会发现 java.io.File.File(java.io.File, java.lang.String)这一个函数被调用 了。在Backtrace之后打印的调用栈中，可以清楚地看到这个构造函数的调用来源。

调用栈的顺序是从下至上的， 根 据 Arguments 那一行会发现打开的文件路径是/data/user_de/0/com.android.settings/shared_prefs，文件 名 为 development.xml 。 虽 然 Return Value后打印的返回值为none，表明这个函数没有返回值，但是也是真实地打印了返回值。

测试结束后，可以根据“作业”的ID来删除“作业” ，取消对这 些函数的Hook

```
 jobs kill <id>
```

除了可以直接Hook一个函数之外，Objection还可以通过执行如 下命令实现对指定类classname中所有函数的Hook（这里的所有函 数并不包括构造函数的Hook）。

```
android hooking watch class <classname>
```

同样以java.io.File类为例：

```
android hooking watch class java.io.File
```

```
com.android.settings on (google: 8.1.0) [usb] # android hooking watch class java.io.File
(agent) Hooking java.io.File.createTempFile(java.lang.String, java.lang.String)
(agent) Hooking java.io.File.createTempFile(java.lang.String, java.lang.String, java.io.File)
(agent) Hooking java.io.File.listRoots()
(agent) Hooking java.io.File.readObject(java.io.ObjectInputStream)
(agent) Hooking java.io.File.slashify(java.lang.String, boolean)
(agent) Hooking java.io.File.writeObject(java.io.ObjectOutputStream)
(agent) Hooking java.io.File.canExecute()
(agent) Hooking java.io.File.canRead()
(agent) Hooking java.io.File.canWrite()
(agent) Hooking java.io.File.compareTo(java.io.File)
(agent) Hooking java.io.File.compareTo(java.lang.Object)
(agent) Hooking java.io.File.createNewFile()
(agent) Hooking java.io.File.delete()
(agent) Hooking java.io.File.deleteOnExit()
(agent) Hooking java.io.File.equals(java.lang.Object)
(agent) Hooking java.io.File.exists()
(agent) Hooking java.io.File.getAbsoluteFile()
(agent) Hooking java.io.File.getAbsolutePath()
(agent) Hooking java.io.File.getCanonicalFile()
(agent) Hooking java.io.File.getCanonicalPath()
(agent) Hooking java.io.File.getFreeSpace()
(agent) Hooking java.io.File.getName()
(agent) Hooking java.io.File.getParent()
(agent) Hooking java.io.File.getParentFile()
(agent) Hooking java.io.File.getPath()
(agent) Hooking java.io.File.getPrefixLength()
(agent) Hooking java.io.File.getTotalSpace()
(agent) Hooking java.io.File.getUsableSpace()
(agent) Hooking java.io.File.hashCode()
(agent) Hooking java.io.File.isAbsolute()
(agent) Hooking java.io.File.isDirectory()
(agent) Hooking java.io.File.isFile()
(agent) Hooking java.io.File.isHidden()
(agent) Hooking java.io.File.isInvalid()
(agent) Hooking java.io.File.lastModified()
(agent) Hooking java.io.File.length()
(agent) Hooking java.io.File.list()
(agent) Hooking java.io.File.list(java.io.FilenameFilter)
(agent) Hooking java.io.File.listFiles()
(agent) Hooking java.io.File.listFiles(java.io.FileFilter)
(agent) Hooking java.io.File.listFiles(java.io.FilenameFilter)
(agent) Hooking java.io.File.mkdir()
(agent) Hooking java.io.File.mkdirs()
(agent) Hooking java.io.File.renameTo(java.io.File)
(agent) Hooking java.io.File.setExecutable(boolean)
(agent) Hooking java.io.File.setExecutable(boolean, boolean)
(agent) Hooking java.io.File.setLastModified(long)
(agent) Hooking java.io.File.setReadOnly()
(agent) Hooking java.io.File.setReadable(boolean)
(agent) Hooking java.io.File.setReadable(boolean, boolean)
(agent) Hooking java.io.File.setWritable(boolean)
(agent) Hooking java.io.File.setWritable(boolean, boolean)
(agent) Hooking java.io.File.toPath()
(agent) Hooking java.io.File.toString()
(agent) Hooking java.io.File.toURI()
(agent) Hooking java.io.File.toURL()
(agent) Registering job 306157. Type: watch-class for: java.io.File
```

此时执行以下命令`jobs list`可以发现一共Hook了56个函数，输出结果如下：

```
com.android.settings on (google: 8.1.0) [usb] # jobs list
Job ID  Hooks  Type
------  -----  -----------------------------
306157     56  watch-class for: java.io.File
```

这里的调用顺序（自上 而下）和之前的调用栈的打印是不同的。

#### 4.6主动调用android heap相关命令。

最后介绍Frida的一 大特色——主动调用在Objection中的使用。

①基于最简单的Java.choose的实现，在Frida脚本中，对实例的 搜索在Objection中是使用以下命令实现的：

```
android heap search instances 
```



```
android heap search instances java.io.File
```

②在Objection中调用实例方法的方式有两种。第一种是使用以 下命令调用实例方法：

```
android heap execute <Handle> <methodname>
```

```
这里的实例方法指的是没有参数的实例方法。下面演示一下使用
Handle值为0x3606所对应的实例来执行File的getPath方法
```

如果要执行带参数的函数，则需要先执行以下命令：

android heap evaluate 

在进入一个迷你编辑器环境后，输入想要执行的脚本内容，确认 编辑完成，然后按Esc键退出编辑器，最后按回车键，即会开始执行 这行脚本并输出结果。这里的脚本内容和在编辑器中直接编写的脚本 内容是一样的（使用File类的canWrite()函数和setWritable()函数进 行测试）

在 这 个 脚 本 中 ， Objection 设 定 clazz 用 于 代 表 Handle 值 为 0x3606所对应的实例，同时函数canWrite()用于返回这个实例所打 开的文件是否可写。setWritable()函数用于修改对应文件是否可写的 属性。

heap evaluate既可以执行有参函数，也可以执行无参函数。