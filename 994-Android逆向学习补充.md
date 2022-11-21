# Android逆向学习补充

## 调试方法

### 源程序修改

一种比较老旧是调试方法，使用apktool的-d选项

- java -jar apktool.jar d -d 目标.apk -o 结果存放目录
- 修改Android.mainfest文件，在application节点中添加 `android:debuggable="true"`
- 在入口点的类的onCreate方法中添加 `invoke-static{},Landroid/os/Debug;->waitForDebugger()V`
- 反编译修改过的APK文件 `java -jar apktool.jar b -d 代码目录 -o 目标APK名字`
- 手动对apk文件进行签名 `java -jar signapk.jar testkey.x509.pem testkey.pk8 未签名apk名 签名APK名`
- 在Android Studio中选择编译好的文件目录，导入代码，在相应位置下好断点（waitdebugger下一行设置好断点）
- 设置远程调试选项，Run->Debug Configurations-> Remote Java Application,Host填写为localhost，端口为Debug开发的端口8700
- 打开apk文件，知道看到wait for debugger
