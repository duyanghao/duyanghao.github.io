---
layout: post
title: Unity打包Apk
date: 2017-2-6 19:10:31
category: 技术
tags: Unity
excerpt: Unity打包Apk步骤……
---

本文总结Unity打包Apk步骤

# 步骤

### 安装JDK和JRE

* 1、安装[Jdk](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

![](/public/img/unity_apk/jdk.png)

* 2、配置环境变量

假设你将jdk安装到：D:\Java_dir\Java\jdk1.8.0_121，同时将jre安装到：D:\Java_dir\Java\jre_dir

step 1：配置Windows用户环境变量`CLASSPATH`=;D:\Java_dir\Java\jdk1.8.0_121\lib\tools.jar;D:\Java_dir\Java\jdk1.8.0_121\lib\dt.jar

step 2：配置Windows用户环境变量`Path`=;D:\Java_dir\Java\jdk1.8.0_121\bin

* 3、验证安装

windows+R 打开运行 输入cmd 输入 java -version , javac , java查看是否有输出，若有输出则安装成功，否则安装失败 

### 安装SDK

* 1、安装[SDK](https://developer.android.com/studio/index.html?hl=zh-cn) （Android Studio包含SDK）
 
![](/public/img/unity_apk/sdk.png)


* 2、配置SDK安装目录

在安装过程中将SDK安装目录与Android Studio分开，假设安装到如下目录：D:\sdk_dir

### 打包APK

* 1、添加build场景

菜单栏->File->Build Settings 

![](/public/img/unity_apk/scene.png)

* 2、配置SDK和JDK路径

菜单栏->Edit->Preferences->External tools，选择SDK和JDK安装路径，如下：

![](/public/img/unity_apk/path.png)

* 3、配置icon

step 1：菜单栏->File->Build Settings->

step 2：点击Play Settings->Icon

step 3：`Enable Android Banner` Select logo图片，如下：

![](/public/img/unity_apk/icon.png)

* 4、打包APK

step 1：菜单栏->File->Build Settings 

step 2：Switch platform 切换到 Android平台

step 3：点击Play Settings->Other Settings,配置如下：

![](/public/img/unity_apk/build.png)

**注意：**

1、`Bundle Identifier`字段内容`com.Company.Productname`中的`Company`替换为任何其他字段

2、`Minimum API Level`字段内容为发布的apk文件的运行环境

3、**Api Compatibility Level**字段设置为`.NET 2.0`

# 参考：

* http://blog.csdn.net/u013553804/article/details/51170997
* http://www.ceeger.com/forum/read.php?tid=5918&ds=1
* http://www.jianshu.com/p/3c67fbfbb67c
* http://www.cnblogs.com/nsky/p/4594371.html
* http://www.cnblogs.com/wanglufly/p/4086788.html
