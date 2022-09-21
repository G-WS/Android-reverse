# 反编译APK文件

## 工具：

反编译工具

静态分析工具

动态调试工具

集成分析环境

## 1、开发Android程序

Android项目：

assets->资源文件

bin->apk程序

libs->.jar

src->存放Java文件

res->布局文件、图片文件、values文件、字符串资源等

AndroidManifest.xml配置文件

上述的android项目结构较老旧，以新版本为主





APK包结构：

asset：资源目录文件夹|c++游戏引擎和LUA脚本，Unity3D开发的资源文件一般放在这个目录下面

lib：so库存放的位置|so库文件由NDK文件编译

META-INF：用于存放签名文件

res文件夹：资源目录文件|除了音频视频，一般用于Java开发的资源文件都放在这个文件夹下面

AndroidManifest.xml：基础配置清单

class.dex：Java曾主要分析文件，它里面全是可执行代码，相当于Windows的PE文件

resources.arcs：存放一些工程中编译的配置文件，以及索引res目录的一些文件。



如何安装APP

到对应文件目录下，打开cmd

adb install 电脑上APK的路径

也可以直接拖进去安装

## 2、分析Android程序

将APK文件扩展名修改为.zip后，可以获得一个压缩包，解压缩后获得的是未经过反编译的APK文件压缩包，其中

Android安装包文件目录

res->资源文件（布局文件，图片，菜单）

META-INF->签名文件夹

resources.arsc->包含一些编译过程中遇到的一些信息，有关资源文件的一些内容

classes.dex->可执行文件

AndroidManifesst.xml->配置文件
