---
title: 通过命令行"adb shell am start"启动app的一个坑
date: 2018-08-29 15:52:00
update: 2018-08-29 15:53:10
categories:
    - 自动化测试
tags:
    - adb
    - android
    - am
    - dumpsys
description: 最近写自动化脚本的时候，遇到一个问题：（为了便于理解，这里拿QQ音乐举例，实际是一个内部项目不方便透露），打开QQ音乐，选择一首歌曲-播放，在播放器界面按home键回到桌面，再点击桌面快捷方式返回QQ音乐，现象：手动操作返回的是播放器界面（即回到桌面之前的界面），但是通过代码执行，返回的却是QQ音乐的首页。本文就造成这一问题的原因和分析过程做个记录，便于以后回顾。

---

# 书接上文

这里用的自动化框架是[openatx/uiautomator2](https://github.com/openatx/uiautomator2)，测试框架是：`pytest` + `allure`。先上一下关键代码：

```python
import pytest
import allure
from .actions import home, player, others   # 一些常用操作步骤的封装，参考了一些PageObject的思想
# type hints
from uiautomator2 import UIAutomatorServer


@allure.story("播放器界面")
@allure.title("回到桌面再返回，依然保持在播放器界面")
def test_player_alive(device: UIAutomatorServer, package_name: str, app_start_success: bool):
    assert app_start_success    # 这里是pytest的fixture，封装了启动app和判断是否成功的逻辑

    home.click_play_button(device, package_name)    # 首页点击播放按钮，随机播放一首歌曲
    home.open_player(device, package_name)      # 打开播放器界面
    assert player.is_playing_page(device, package_name)   # 断言是否成功进入播放器界面
    others.back_to_launcher(device, package_name)   # 按home键回到桌面
    assert device.current_app().get('package') != package_name  # 断言当前package_name与被测app不符（确认回到了桌面）
    with allure.step("重新进入APP"):    # allure标记步骤
        device.app_start(package_name)  # 再次启动app
        allure.attach(device.screenshot(format='raw'), "依然保持在播放器界面", allure.attachment_type.JPG)  # allure报告中添加截图
    assert player.is_playing_page(device, package_name)     # 断言确实回到了播放器界面

```

## 出了这个问题以后，查看了下uiautomator2中`app_start()`的代码，发现就是通过android的`am`命令实现的。

这就很奇怪了，因为在写成脚本之前，我已经手动通过命令行验证了使用`am`命令是可行的，完整的命令是：`adb shell am start -n com.tencent.qqmusic/.activity.AppStarterActivity`，执行结果：

```shell
$ am start -n com.tencent.qqmusic/com.tencent.qqmusic.activity.AppStarterActivity
Starting: Intent { cmp=com.tencent.qqmusic/.activity.AppStarterActivity }
Warning: Activity not started, its current task has been brought to the front
```

难道uiautomator2的封装有问题，或者用了什么amazing的方法我没看懂？随意干脆在代码里直接把`app_start()`替换成了`shell("am start -n com.tencent.qqmusic/.activity.AppStarterActivity")`。
但是执行一下，问题依旧。不过这个时候又有了新的发现：
手动执行的时候多了一个warning：`Warning: Activity not started, its current task has been brought to the front`，在脚本中执行shell命令却没有这个warning。实在想不通了，给作者提了个issues。

## 在等待回复这段时间，整理了下思路，换了个角度去思考

既然在android系统里桌面也是一个app，播放器是一个app，那么冷启动（app未启动，从桌面进入）/热启动（app启动后挂后台，从桌面进入）过程其实就是从一个app拉起另外一个app。所以撸起袖子继续请教谷歌大神，发现一个app拉起另外一个app，都是通过构造一个`Intent`，在这个过程中会传入ACTION、CATEGORY等各种参数。

### 所以，有没有办法记录所有 Intent ？

查了一圈没有找到答案，但是根据经验`dumpsys`这个命令记录的东西真是太多了，但是不是没人详细介绍这个命令，就是开发写的源码分析，对于我来讲从android源码角度去分析还是有点难了。没办法，一条一条试吧……功夫不负有心人，当尝试`dumpsys activity recents`这条命令时，终于发现端倪了：

```shell
$ dumpsys activity recents
ACTIVITY MANAGER RECENT TASKS (dumpsys activity recents)
  Recent tasks:
  * Recent #0: TaskRecord{5fb7ac0 #5217 A=android.task.qqmusic U=0 StackId=1 sz=1}
    userId=0 effectiveUid=u0a140 mCallingUid=u0a161 mUserSetupComplete=true mCallingPackage=ch.deletescape.lawnchair.plah
    affinity=android.task.qqmusic
    intent={act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10200000 cmp=com.tencent.qqmusic/.activity.AppStarterActivity}
    realActivity=com.tencent.qqmusic/.activity.AppStarterActivity
    autoRemoveRecents=false isPersistable=true numFullscreen=1 taskType=0 mTaskToReturnTo=1
    rootWasReset=true mNeverRelinquishIdentity=true mReuseTask=false mLockTaskAuth=LOCK_TASK_AUTH_PINNABLE
    Activities=[ActivityRecord{7339e65 u0 com.tencent.qqmusic/.activity.AppStarterActivity t5217}]
    askedCompatMode=false inRecents=true isAvailable=true
    lastThumbnail=android.graphics.Bitmap@774cf23 lastThumbnailFile=/data/system_ce/0/recent_images/5217_task_thumbnail.png
    stackId=1
    hasBeenVisible=true mResizeMode=RESIZE_MODE_RESIZEABLE mSupportsPictureInPicture=false isResizeable=true supportsSplitScreen=true firstActiveTime=1535538967561 lastActiveTime=1535541667612 (inactive for 77s)
  * Recent #1: TaskRecord{eb21815 #5189 I=ch.deletescape.lawnchair.plah/ch.deletescape.lawnchair.Launcher U=0 StackId=0 sz=1}
    userId=0 effectiveUid=u0a161 mCallingUid=1000 mUserSetupComplete=true mCallingPackage=com.android.systemui
    intent={act=android.intent.action.MAIN cat=[android.intent.category.HOME] flg=0x10a00000 cmp=ch.deletescape.lawnchair.plah/ch.deletescape.lawnchair.Launcher}
    realActivity=ch.deletescape.lawnchair.plah/ch.deletescape.lawnchair.Launcher
    autoRemoveRecents=false isPersistable=false numFullscreen=1 taskType=1 mTaskToReturnTo=0
    rootWasReset=true mNeverRelinquishIdentity=true mReuseTask=false mLockTaskAuth=LOCK_TASK_AUTH_PINNABLE
    Activities=[ActivityRecord{209110f u0 ch.deletescape.lawnchair.plah/ch.deletescape.lawnchair.Launcher t5189}]
    askedCompatMode=false inRecents=true isAvailable=true
    lastThumbnail=null lastThumbnailFile=/data/system_ce/0/recent_images/5189_task_thumbnail.png
    stackId=0
    hasBeenVisible=true mResizeMode=RESIZE_MODE_UNRESIZEABLE mSupportsPictureInPicture=false isResizeable=false supportsSplitScreen=false firstActiveTime=1535525128804 lastActiveTime=1535541667598 (inactive for 77s)
...
```

按字面意思理解，这条命令查看的是最近的activity，这里通过切换多任务、尝试各个app之间互相拉起后查看结果，也验证了这一点，且第一个task（#0）是最新的一个activity，这里面有Intent等各种信息。
所以，就用这种办法分析试试。这里尝试手动桌面返回app和脚本自动化，再把两次的操作结果保存下来作对比，有了新的发现：

- 出错的时候（脚本），Intent是：`intent={flg=0x10000000 cmp=com.tencent.qqmusic/.activity.AppStarterActivity}`
- 正常的时候（手动），Intent是：`intent={act=android.intent.action.MAIN cat=[android.intent.category.LAUNCHER] flg=0x10200000 cmp=com.tencent.qqmusic/.activity.AppStarterActivity}`

`act`和`cat`这两个参数十分可疑！查看一下`am -h`，发现这两个参数是`ACTION`和`CATEGORY`，赶紧加上试试！

所以完整shell命令修改为：`adb shell am start -a android.intent.action.MAIN -c android.intent.category.LAUNCHER -n com.tencent.qqmusic/.activity.AppStarterActivity`。

手动执行一下，没问题！那再回到代码，把桌面回到QQ音乐的代码改成：`shell("am start -a android.intent.action.MAIN -c android.intent.category.LAUNCHER -n com.tencent.qqmusic/.activity.AppStarterActivity")`。这次该解决了吧！执行一遍，又报错了！

## 喵？？？

继续调试吧……这次“狠”一点，干脆在第一行代码就加上断点。不过，这次灵机一动，在断点这里停下时，手动执行了一下`dumpsys activity recents`，终于找到问题了！原来启动的时候没有`ACTION`和`CATEGORY`！

这时候不禁恍然大悟，原来我把启动app封装在了fixture里，然而fixture里依然使用的是方法`app_start()`，而这个方法是不带参数的！所以其实问题出在app首次启动不带参数。

既然找到根本原因了，修改一下代码，反手给作者一记PR（pull request），折腾了小半天终于解决！

> 这里故意忽略了`FLAG`这个参数，因为我怕解释错了。我的理解是，像`FLAG`、`EXTRA_KEY`、`DATA_URI`这类参数，传递的都是附加信息/数据，通常来讲，开发实现时不会通过这个作为判断"跳转"来源的依据。
> 举个不恰当的例子：这就像用selenium在做自动化，driver指定用什么浏览器测试，`driver.get(URL)`执行打开URL，一般不会去用URL（数据）判断用什么浏览器去测试。因为“数据”是外部传递过来的，它的内容是不可预期的；“行为”是系统规定的，只有几个选项可用。

另外，
> 最后还是再强调一下，文中只是用QQ音乐举例而已，实际用QQ音乐是无法复现这个问题的！

## 总结

面试造火箭，入职拧螺丝。

我印象最深的就是一般的面试题都有四大组件啊、android系统架构这些东西，基本大家就是背一下而已，工作中觉得用不上。真正出了问题，能查到的资料大部分又都是开发视角，对于测试来讲非常不友好。

就拿这个`dumpsys`命令来说吧，大部分开发不会深入去研究，因为这不是业务必须的。即使某些情况下需要，直接阅读源码就能快速理解。但是测试面临这个问题就很难办了，一是代码能力不一定够，二是某些工具几乎没有文档。而且因为android开源，厂商可以对系统进行定制，所以这个工具在不同的厂商、不同的系统版本、甚至不同的ROM上，表现并不一致（亲眼见过有个定制ROM把所有命令行工具都加上了作者的名字。。。）

作为一名小测试，可能很难改变这种现状，但是千万不要把自己的工作局限于每天点点点。打好基础，努力保持每天都在学习，才不会被淘汰。