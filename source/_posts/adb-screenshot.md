---
title: 提高Android设备命令行截图的效率
date: 2018-01-15 15:25:10
update: 2018-08-29 15:53:10
categories:
    - 自动化测试
tags:
    - adb
    - android
description: 想在PC端为android设备截图，通过adb执行shell命令无疑是一种最常用的手段，但是这个命令在执行的时候会存在效率低的问题。本文试验了直接使用`adb exec-out`命令的方法，能够显著提高截图的速度。
---

目前市面兼顾了易用性和兼容性的截图工具，最好用的当属`screencap`（支持android 4.0+，无需root）了。基本用法如下：

```bash
usage: screencap [-hp] [-d display-id] [FILENAME]
    -h: this message
    -p: save the file as a png.
    -d: specify the display id to capture, default 0.
    If FILENAME ends with .png it will be saved as a png.
    If FILENAME is not given, the results will be printed to stdout.
```

所以一般用法是：

```bash
    adb shell screencap -p /sdcard/xxx.png
    adb pull /sdcard/xxx.png .
```

~~其实还有第二种用法：`adb shell screencap -p | sed 's/\r$//' > screen.png`。但是又涉及到sed命令，Windows不支持、手机里可能被阉割，所以通用性不强。~~

现在发现有个`adb exec-out`命令可以简化这一过程：

```bash
    adb exec-out screencap -p > xxx.png
```

解释：
"adb exec-out" gives binary output.（“adb exec-out”提供二进制数据。）这样比前面的方案，少了一轮读写文件，效率自然高一些。

但是，还是有一些需要注意的地方：

1. macOS（bash）下测试正常，在Windows的powershell下失效（图片文件打不开），但cmd下正常。所以win下面可以写成批处理（.bat文件），然后鼠标双击运行。
2. 不能调整文件格式。虽然可以修改`xxx.png`为`xxx.jpg`，但这只是一种掩耳盗铃的手段。使用16进制编辑器查看该文件头，其实还是png。
