---
title: 命令行获取minSdkVersion、targetSdkVersion
date: 2018-08-21 20:13:00
categories:
    - 自动化测试
tags:
    - adb
    - android
---
# 这里以chrome为例。

其中apk文件为`chrome-67-0-3396-87.apk`，对应的包名为`com.android.chrome`

## 有apk

### 方式一：minSdkVersion-可以、targetSdkVersion-可以

```shell
$ aapt list -a chrome-67-0-3396-87.apk |grep -i sdk
    A: android:compileSdkVersion(0x01010572)=(type 0x10)0x1c
    A: android:compileSdkVersionCodename(0x01010573)="9" (Raw: "9")
    E: uses-sdk (line=3)
      A: android:minSdkVersion(0x0101020c)=(type 0x10)0x10
      A: android:targetSdkVersion(0x01010270)=(type 0x10)0x1c
    E: uses-permission-sdk-23 (line=10)
    E: uses-permission-sdk-23 (line=15)
    E: uses-permission-sdk-23 (line=16)
    E: uses-permission-sdk-23 (line=17)
    E: uses-permission-sdk-23 (line=18)
        A: android:name(0x01010003)="com.samsung.android.sdk.multiwindow.enable" (Raw: "com.samsung.android.sdk.multiwindow.enable")
        A: android:name(0x01010003)="com.samsung.android.sdk.multiwindow.multiinstance.enable" (Raw: "com.samsung.android.sdk.multiwindow.multiinstance.enable")
        A: android:name(0x01010003)="com.samsung.android.sdk.multiwindow.multiinstance.launchmode" (Raw: "com.samsung.android.sdk.multiwindow.multiinstance.launchmode")
        A: android:name(0x01010003)="com.samsung.android.sdk.multiwindow.penwindow.enable" (Raw: "com.samsung.android.sdk.multiwindow.penwindow.enable")
```

> 注意，这里取出来的值是16进制的，需要转换为10进制使用。
> python 16进制转10进制： `int('0x10', 16) = 16`

### 方式二：minSdkVersion-可以、targetSdkVersion-可以

```shell
$ aapt d badging chrome-67-0-3396-87.apk |grep -i sdk
package: name='com.android.chrome' versionCode='339608700' versionName='67.0.3396.87' compileSdkVersion='28' compileSdkVersionCodename='9'
sdkVersion:'16'
targetSdkVersion:'28'
uses-permission-sdk-23: name='android.permission.ACCESS_WIFI_STATE'
uses-permission-sdk-23: name='android.permission.BLUETOOTH'
uses-permission-sdk-23: name='android.permission.BLUETOOTH_ADMIN'
uses-permission-sdk-23: name='android.permission.REORDER_TASKS'
uses-permission-sdk-23: name='android.permission.REQUEST_INSTALL_PACKAGES'
  uses-feature-sdk-23: name='android.hardware.bluetooth'
  uses-implied-feature-sdk-23: name='android.hardware.bluetooth' reason='requested android.permission.BLUETOOTH permission, requested android.permission.BLUETOOTH_ADMIN permission, and targetSdkVersion > 4'
  uses-feature-sdk-23: name='android.hardware.wifi'
  uses-implied-feature-sdk-23: name='android.hardware.wifi' reason='requested android.permission.ACCESS_WIFI_STATE permission'
```

> `minSdkVersion`对应的应该是这里的`sdkVersion`

## 没有apk（已安装在手机中）：minSdkVersion-不行、targetSdkVersion-可以

```shell
$ adb shell dumpsys package com.android.chrome|grep -i sdk
    versionCode=339608700 targetSdk=28
```

> 注意：`sdk`可能是小写、可能是大小写混合，所以`grep`命令要使用`-i`参数，指定忽略大小写