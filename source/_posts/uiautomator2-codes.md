---
title: Uiautomator2 源码分析
date: 2018-12-28 19:09:48
tags:
    - 自动化测试
    - python
    - uiautomator2
---
# Uiautomator2 源码分析

## 引言

关于著名的软件测试金字塔理论，可以看一下 [The Practical Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) 这篇文章，译文在[这里](https://mp.weixin.qq.com/s/Dee2j_lhNkOSnkflagBohg)。

对于UI自动化测试，根据其实现原理可以大致分为：基于坐标的（monkeyrunner）、基于控件的（uiautomator）、基于图像识别的（sikuli）等。针对不同的测试场景，不同方案各有其用武之地。

[Android Uiautomator2 Python Wrapper](https://github.com/openatx/uiautomator2) 是基于 Google uiautomator2 库实现的自动化测试框架。该框架主要解决了两大痛点：

1. 测试脚本只能使用Java语言 
2. 测试脚本必须每次被上传到设备上运行

其实现原理就是在手机上运行了一个 http 服务器，将 uiautomator 中的功能开放出来，然后再将这些 http 接口，封装成 Python 库。

<!-- more -->

## Python 基础知识

### 类和实例

面向对象最重要的概念就是类（Class）和实例（Instance），必须牢记类是抽象的模板，比如Student类，而实例是根据类创建出来的一个个具体的“对象”，每个对象都拥有相同的方法，但各自的数据可能不同。

```python
class Student(object):
    def __init__(self, name, score):
        self.name = name
        self.score = score

    def get_grade(self):
        if self.score >= 90:
            return 'A'
        elif self.score >= 60:
            return 'B'
        else:
            return 'C'
        
lisa = Student('Lisa', 99)
bart = Student('Bart', 59)
print(lisa.name, lisa.get_grade())
print(bart.name, bart.get_grade())

=========================
Lisa A
Bart C
```



### 魔术方法`__call__`

什么是魔术方法？他们是面向对象的 Python 的一切。他们是可以给你的类增加”magic”的特殊方法。他们总是被双下划线所包围(e.g. `__init__` 或者 `__lt__`)。

你也许已经知道，在 Python 中，方法也是一种高等的对象。这意味着他们也可以被传递到方法中就像其他对象一样。这是一个非常惊人的特性。 在Python中，一个特殊的魔术方法可以让类的实例的行为表现的像函数一样，你可以调用他们，将一个函数当做一个参数传到另外一个函数中等等。这是一个非常强大的特性让Python编程更加舒适甜美。 `__call__(self, [args...])`

允许一个类的实例像函数一样被调用。实质上说，这意味着 `x()` 与 `x.__call__()` 是相同的。注意 `__call__` 参数可变。这意味着你可以定义 `__call__` 为其他你想要的函数，无论有多少个参数。

`__call__` 在那些类的实例经常改变状态的时候会非常有效。调用这个实例是一种改变这个对象状态的直接和优雅的做法。用一个实例来表达最好不过了:

```python
class Entity:
'''调用实体来改变实体的位置。'''

def __init__(self, size, x, y):
    self.x, self.y = x, y
    self.size = size

def __call__(self, x, y):
    '''改变实体的位置'''
    self.x, self.y = x, y
```

```python
class A:
    def __call__(self, *args, **kwargs):
        print(args)


class B:
    pass


a = A()
b = B()
a(3)
b(3)
==============
(3,)
Traceback (most recent call last):
  File "/Users/edsion/Documents/qqmusic/11111.py", line 23, in <module>
    b(3)
TypeError: 'B' object is not callable
```



## 源码分析

首先，来看一个最简单的例子：

```python
import uiautomator2

d = uiautomator2.connect("a2c0fe65")
d(text="我的").click()
```

第一行代码，执行导入 uiautomator2 这个库。这个没什么可解释的，python的基础语法。

第三行，执行 `connect()`，参数是`"a2c0fe65"`，按字面意思是连接设备，并将返回的对象赋值给变量`d`。

第四行，给`d`传递一个参数`text`，值为`"我的"`，然后调用执行`click()`方法。



下面我们来看一下代码的具体实现。打开 pycharm，把这一段代码敲进去。首先从`connect()`入手，所以`^ + 鼠标左键单击`，跳转到源代码。此时发现，这段代码位于`site-packages/uiautomator2/__init__.py`文件中。

对于第三方模块，最佳实践是拆分成多个模块（.py文件），然后在`__init__.py`里逐个`import`，这样可以清晰地指定某些模块和方法暴露出来供第三方调用，某些模块和方法作为“隐藏”的仅供内部使用。uiautomator2 这个库，可能是作者考虑代码量不是很多，并没有对其进行拆分，而是所有代码一股脑写在了`__init__.py`里，这样也问题不大。



{% asset_img 006tNbRwly1fymj8vcjwcj30d20m5405.jpg [__init__.py] %}

根据代码量和用途，例如某些一眼就能看出来的工具类和异常类，总结一下最终此模块的核心代码：

- 函数：connect()（包括connect_usb() 和 connect_wifi()）
- 类：Selector(), Session(), UIAutomatorServer(), UIObject()



查看 connect() 的逻辑发现，这里主要实现了根据你传入的参数判断是走usb连接还是Wi-Fi连接（当然，最终的本质还是通过 http ）。如果连接成功，最后返回一个 UIAutomatorServer 实例。



代码继续往下走，下面就是执行“查找控件并点击（前面例子里的第四行代码）”操作了。根据前面的基础知识，结合刚才的分析，可以知道这里的 `d`是 UIAutomatorServer 的实例，如果要想让实例作为一个可调用的对象，需要在类里面实现`__call__()`方法。所以在 UIAutomatorServer 找一下`__call__()`方法，发现最终是调用的 Session 这个类。在找 Session 里的`__call__()`方法，最终发现参数`text="我的"`被传递到 Selector，然后调用的 UiObject 类的 `click()`方法。再然后，就是把参数和方法转为http请求发送至服务端（手机）去执行了。至此完整的调用逻辑已经基本理清了。



我们先看一下 google uiautomator 的例子：

```java
mDevice = UiDevice.getInstance(getInstrumentation());

// Bring up the default launcher by searching for a UI component
// that matches the content description for the launcher button.
UiObject allAppsButton = mDevice
        .findObject(new UiSelector().description("Apps"));

// Perform a click on the button to load the launcher.
allAppsButton.clickAndWaitForNewWindow();
```



按照我的粗浅理解，如果这段代码转为 python 的话，大概是这样的：

```python
device = uiautomator2.connect("a2c0fe65")

selector = Selector(description="Apps")
button = device.findObject(selector)
button.click()

# 或者简写成
device.findObject(Selector(description="Apps")).click()
```

可以看出来，缺点是比前面的例子书写起来更费力，当然可能从语义上来讲可能表达的更清晰。 另一个常用的自动化测试框架 —— Appium，风格就更接近这个。



两者孰优孰略不做评论，主要是通过理清这个过程，加深对框架的理解。因为文档不完善，有些细节其实需要我们深入研究代码才能掌握。



## 实战

### implicitly_wait() 与 wait_timeout 

在文档中搜索wait，关于默认等待控件出现的默认时间，有`implicitly_wait()`和`wait_timeout`两处。

纸面的区别，一个是 Session 类的方法，一个是 UIAutomatorServer 类的属性。但是深入到逻辑来看，最终`implicitly_wait()`方法也是去设置的 UIAutomatorServer 的 wait_timeout 属性。所以从效果来看，应该是一样的。



### click() 与 click_exists() 

1. `click()`不含异常捕获逻辑，找不到控件会报错（代码不会继续执行）；`click_exists()`包含异常捕获逻辑，当发生`UiObjectNotFoundError`时不会报错（代码可以继续执行下去）

2. `click()`永远返回`None`，`click_exists()`会根据是否触发`UiObjectNotFoundError`返回`True`或`False`

3. `click()`的默认参数是`timeout=None`，`click_exists()`的默认参数是`timeout=0`，逻辑上会根据`timeout is None`判断。即当直接使用`click()`（不指定 timeout 参数）时，等待最多`wait_timeout`秒后，如果控件存在执行点击操作；当直接使用`click_exists()`（不指定 timeout 参数）时，立刻执行点击操作。如果都指定了timeout参数，则遵照此参数。
