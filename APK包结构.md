# APK包结构

## asset

资源目录文件夹：C++游戏引擎和LUA脚本，Unity3D开发的资源文件一般放在这个目录下

## lib

so库存放的位置：so库由NDK文件编译

## META-INF

用于存放签名文件

## res文件夹

资源目录文件：除了音频视频，一般用于Java开发的资源文件都放在这个文件夹下

## AndroidManifest.xml

基础配置清单

## class.dex

JAVA层主要分析文件，它里面全是可执行代码，相当于Windows的PE文件

## resources.arcs

存放一些工程中编译的配置文件，以及索引res目录的一些文件