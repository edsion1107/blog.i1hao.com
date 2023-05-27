---
title: 如何通过命令行读取android app的数据库
date: 2018-01-16 15:25:10
update: 2018-01-16 15:25:10
categories:
    - 自动化测试
tags:
    - adb
    - android
    - database

---
我们测试的时候有可能需要通过读取数据库来确认问题，但是某些设备上无法获取root权限（尤其是android 5.0以上），这里简单整理一些小技巧。
<!-- more -->

# 测试app

[待测 apk 下载](http://wordpress-1251008864.file.myqcloud.com/adb_databases.zip)

# 查看app所使用的数据库

首先想到的就是`dumpsys`命令，执行一下`adb shell dumpsys package com.heima.ormlitedemo`，结果如下：

```bash
Activity Resolver Table:
  Non-Data Actions:
      android.intent.action.MAIN:
        1c28858b com.heima.ormlitedemo/.MainActivity
Key Set Manager:
  [com.heima.ormlitedemo]
      Signing KeySets: 3332
Packages:
  Package [com.heima.ormlitedemo] (296f3941):
    userId=10226 gids=[1028, 1015]
    pkg=Package{2a736668 com.heima.ormlitedemo}
    codePath=/data/app/com.heima.ormlitedemo-2
    resourcePath=/data/app/com.heima.ormlitedemo-2
    legacyNativeLibraryDir=/data/app/com.heima.ormlitedemo-2/lib
    primaryCpuAbi=null
    secondaryCpuAbi=null
    dexoptNeeded=false
    versionCode=1 targetSdk=25
    versionName=1.0
    splits=[base]
    applicationInfo=ApplicationInfo{18c67a81 com.heima.ormlitedemo}
    flags=[ DEBUGGABLE HAS_CODE ALLOW_CLEAR_USER_DATA ALLOW_BACKUP ]
    dataDir=/data/data/com.heima.ormlitedemo
    supportsScreens=[small, medium, large, xlarge, resizeable, anyDensity]
    timeStamp=2018-01-08 14:57:48
    firstInstallTime=2018-01-08 14:57:10
    lastUpdateTime=2018-01-08 14:57:48
    signatures=PackageSignatures{15705ed4 [69ba726]}
    permissionsFixed=true haveGids=true installStatus=1
    pkgFlags=[ DEBUGGABLE HAS_CODE ALLOW_CLEAR_USER_DATA ALLOW_BACKUP ]
    User 0:  installed=true hidden=false stopped=false notLaunched=false enabled=0
    grantedPermissions:
      android.permission.READ_EXTERNAL_STORAGE
      android.permission.WRITE_EXTERNAL_STORAGE
```

发现一些app相关的信息都有了，但就是没有database路径和文件名。要知道无root权限访问`/data/data`路径，总是提示`Permission denied`。虽然可以在一个root过的设备上先把路径记录下来，但毕竟不是正途，应该还有更“科学”的办法的。
后来果然不负重望，在使用`dumpsys meminfo`命令时居然让我找到了一个小花招：

```bash
>>adb shell dumpsys meminfo com.heima.ormlitedemo

Applications Memory Usage (kB):
Uptime: 22075649 Realtime: 22075649

** MEMINFO in pid 20762 [com.heima.ormlitedemo] **
                   Pss  Private  Private  Swapped     Heap     Heap     Heap
                 Total    Dirty    Clean    Dirty     Size    Alloc     Free
                ------   ------   ------   ------   ------   ------   ------
  Native Heap     7459     7396        0        0    11828    11803       24
  Dalvik Heap     2661     2400        0        0    21928    13609     8319
 Dalvik Other      460      460        0        0
        Stack      232      232        0        0
       Ashmem        2        0        0        0
      Gfx dev      896      284        0        0
    Other dev        5        0        4        0
     .so mmap     2401      184      336        0
    .apk mmap      126        0       16        0
    .ttf mmap       70        0       40        0
    .dex mmap     3036        0     3032        0
    .oat mmap     1186        0      208        0
    .art mmap     1212      824      156        0
   Other mmap      158        4       84        0
   EGL mtrack    39012    39012        0        0
      Unknown      209      208        0        0
        TOTAL    59125    51004     3876        0    33756    25412     8343

 Objects
               Views:       26         ViewRootImpl:        2
         AppContexts:        3           Activities:        1
              Assets:        5        AssetManagers:        5
       Local Binders:       19        Proxy Binders:       19
       Parcel memory:        2         Parcel count:       11
    Death Recipients:        1      OpenSSL Sockets:        0

 SQL
         MEMORY_USED:      624
  PAGECACHE_OVERFLOW:      128          MALLOC_SIZE:       62

 DATABASES
      pgsz     dbsz   Lookaside(b)          cache  Dbname
         4       16             32         3/20/4  /data/data/com.heima.ormlitedemo/databases/user.db
         4       16             32         3/20/4  /data/data/com.heima.ormlitedemo/databases/user.db
         4       16             32         3/20/4  /data/data/com.heima.ormlitedemo/databases/user.db
         4       16             32         3/20/4  /data/data/com.heima.ormlitedemo/databases/user.db
         4       16             32         3/20/4  /data/data/com.heima.ormlitedemo/databases/user.db
         4       16             37         3/35/6  /data/data/com.heima.ormlitedemo/databases/user.db
```

可以看到末尾是有打开数据库的内存占用的，这里可以显示数据库的绝对路径。
**但是这个办法有个限制：数据库必须要使用过，即必须打开app并手动触发一次数据库的增删改查。**

# run-as

首先，使用run-as命令需要被测app为可调试（打包时设置debuggable=true，也通过pkgFlags或flags是否存在DEBUGGABLE来判断），或手机为全局可调试（ro.debuggable=1，一般手机出厂后都是0）。
先上命令：

```bash
adb exec-out run-as com.heima.ormlitedemo cat /data/data/com.heima.ormlitedemo/databases/user.db > user.db
adb exec-out run-as com.heima.ormlitedemo cat databases/user.db > user.db
```

这两条命令，功能是一样的，唯一的区别是注意数据库文件的路径可以使用绝对路径（第一个）或相对路径（第二个）。第二个要注意的地方就是，重定向符号`>`后面的文件路径，不是设备上的，而是你的PC。
其实另外还有两种办法，一个是用`tar`命令代替`cat`，这样可以直接压缩整个文件夹；另外一个办法，在`adb shell`的交互式命令行下，先通过`run-as`改变当前用户，再用`chmod`修改文件权限再拷贝出来。但是这两个办法第一个我尝试后没有成功，第二种方式步骤太繁琐，所以都被我放弃了。

# backup

  第二个方案是使用`adb backup`将数据备份，然后借用工具解密。
先执行命令`adb backup -f backup.ab -noapk com.heima.ormlitedemo`，然后手机端会弹出提示，让你设置密码，为了简单这里直接设置为空即可。其中`-f`用来指定备份文件（扩展名为.ab），`-noapk`指定不备份apk本身，最后一个参数是要备份app的包名。
这样备份以后，通过[Android Backup Utilities](https://sourceforge.net/projects/android-backup-utilities/)这样一个工具，即可解密得到数据库和其它文件了。
[Android Backup Utilities备份下载](http://wordpress-1251008864.file.myqcloud.com/Android_Backup_Utilities_20171015.zip)
用法：

```bash
java -jar "Android Backup Utilities/Android Backup Extractor/android-backup-extractor-20171005-bin/abe.jar" unpack backup.ab backup.tar
```

这样把`.ab`文件解密为tar，然后再解压出来，找到数据库文件即可。
相对上一个方案，这个办法对app没有限制，也不要求root权限，唯一的隐患就是某些厂商“优化”了backup功能。