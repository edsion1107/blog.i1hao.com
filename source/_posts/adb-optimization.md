---
title: 浅析adb原理和优化
date: 2019-05-30 15:53:41
categories:
    - 自动化测试
tags:
    - adb
    - android
---

# 浅析 ADB 原理和优化

最近研究小程序自动化的时候，拜读了几篇关于小程序原理的文章，其中提到了一个词"管控"。作为平台方，必须保证提供的服务是可控的（细节与主题无关，这里不再展开）。受此启发，对于我们的平台来说，如果希望对外提供自动化能力，就必须对执行环境做一定的保护。如果具体到 android 平台，那首先要研究的就是 ADB（Android Debug Bridge）。

> 具体要管控什么？举个例子，执行`adb kill-server` 会使 adb server 退出——因为是 adb server 提供的接口，所以不涉及权限问题（即使 adb server 是通过 root 权限启动，普通用户通过 adb client 调用此接口，adb server 也会退出。）。

<!-- more -->

## ADB 原理

首先，上源码地址：

- googlesource: [https://android.googlesource.com/platform/system/adb](https://android.googlesource.com/platform/system/adb)
- github: [https://github.com/aosp-mirror/platform_system_core.git](https://github.com/aosp-mirror/platform_system_core.git)

将代码 clone 到本地，可以找到文档`OVERVIEW.TXT`，这里详细介绍了整个 adb 服务的结构：

{% asset_img Jietu20190530-210218.png %}



其中，ADB client 与 ADB server 是运行在 pc 本地的，ADB daemon (adbd) 则运行在设备（手机）中。

> ADB client 与 ADB server 代码是写在一起的，最后编译为同一个可执行程序。当 ADB server 未运行时，ADB client 会自动拉起 ADB server 并执行命令。
>
> ADB server 不会主动退出，一直在后台监听 USB 设备的连接，判断是 android 设备后获取信息和做一些初始化的工作。



ADB client 执行的命令（在这被叫做 service ）被分为两类：Host Services （与 ADB server 之间交互）和 Local Services（请求经 ADB server 转发给 ADB daemon 执行）。

- Host Services 的例子：打开终端，执行`adb devices`，此时请求到达 ADB server 后，会把当前连接的所有设备返回—— ADB server 不是每次请求时才去查询设备，而是不停在后台轮询，当设备状态变化时记录下来。
-  Local Services 的例子：打开终端，执行`adb shell ps`，此时请求经过 ADB server，转发到 ADB daemon 执行。这也就解释了为什么在Windows上执行`adb shell ps |grep xxx`会出错，但`adb shell "ps |grep xxx"`却可以正确执行。



具体的通信协议细节，可以查看`protocol.txt`。这里唯一要提到的就是，整个通信的过程是基于socket连接的，之后谈到性能优化的时候还会用到。



## ADB 的优化

了解了一些背景知识以后，初步确定可以有两个方案来进行优化：

- 方案A：将 adb 可执行程序封装为接口，以 sdk 方式提供给外界使用
- 方案B：为 ADB server 添加一层“代理”，通过接口的形式暴露出来



为了确定最终方案，我搜索了下 github 和 pypi（这里以 python 为例，毕竟自动化测试首选语言）上的有关项目，并结合平日使用的一些经验，最终敲定了方案B。

先说说方案A的优缺点：

1. 实现简单（直接 subprocess 模块调用即可）。
2. 如果使用的是Windows，安装各种助手以后，出现多个进程去抢占 5037 端口（ADB server 默认端口）问题，非常影响稳定性。
3. 不区分标准输出（stdout）和标准错误（stderr）。
4. 不传递 shell 命令的 return code（返回的永远都是0），而这在某些时候是很有用的。
5. 可能错误地使用命令等原因，导致出现“僵尸进程“。
6. subprocess 创建子进程在性能上开销较大。
7. 平台需要设备间隔离的能力，但原始的 adb 不具备这个能力。也就是说，只要暴露 5037（或其它端口，启动时制定）给用户，用户可以轻易绕过 sdk 限制对设备进行操作。
8. 稳定性原因，如果设备发生闪断，可以对外接口中加入重试机制。



计划中的方案B：

{% asset_img Jietu20190530-210247.png %}



初步计划，Proxy Server 采用 gRPC 通信，这样对 client 端也没有语言层面的限制，基于 gRPC 的实践可以相对容易地解决开放接口上的各种问题（比如认证、鉴权等）。

基于我们的平台策略，ADB server 与 ADB daemon 之间，可以只考虑 USB 本地连接，回避通过 tcp 连接带来的不稳定因素。

至于 Proxy server 与 ADB server 之间的连接，也是我们目前主要关注的部分，采用 Unix 域套接字来代替监听5037 端口。这样更安全：一方面，无权限用户无法操作 Unix域套接字；另一方面，ADB server 通过监听 Unix 域套接字启动后，即使再启动一个 ADB server 去监听 5037 或其它端口，也获取不到设备列表，也就无从干扰其它设备的运行。

另外，出于性能考虑，监听 Unix 套接字性能也更好一些。这里是一个例子：

```python
#!/usr/bin/env python3  
# -*- coding:utf-8 _*- 
""" 
@author: edsion
@file: socket_adb.py
@time: 2019/05/28
@contact: edsion@21kunpeng.com

"""
import sys
import socket
import struct

host = '127.0.0.1'
port = 5037


class Protocol:
    OKAY = 'OKAY'
    FAIL = 'FAIL'
    STAT = 'STAT'
    LIST = 'LIST'
    DENT = 'DENT'
    RECV = 'RECV'
    DATA = 'DATA'
    DONE = 'DONE'
    SEND = 'SEND'
    QUIT = 'QUIT'

    @staticmethod
    def decode_length(length):
        return int(length, 16)

    @staticmethod
    def encode_length(length):
        return "{0:04X}".format(length)

    @staticmethod
    def encode_data(data):
        b_data = data.encode('utf-8')
        b_length = Protocol.encode_length(len(b_data)).encode('utf-8')
        return b"".join([b_length, b_data])


@profile
def main():
    l_onoff = 1
    l_linger = 0
    if len(sys.argv) == 1:
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.setsockopt(socket.SOL_SOCKET, socket.SO_LINGER, struct.pack('ii', l_onoff, l_linger))
        s.connect((host, port))
    else:
        s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
        s.setsockopt(socket.SOL_SOCKET, socket.SO_LINGER, struct.pack('ii', l_onoff, l_linger))
        s.connect('/Users/edsion/Documents/fb-adb/build/adb.sock')

    s.settimeout(10)

    data = Protocol.encode_data('host:devices')
    s.send(data)
    state = s.recv(4).decode('utf-8')
    print(state)
    length = int(s.recv(4).decode('utf-8'), 16)
    recv = bytearray(length)
    view = memoryview(recv)
    s.recv_into(view)
    print(recv)
    s.close()


if __name__ == '__main__':
    main()

```

这段代码不要直接执行，需要安装`line-profiler`— python 的性能分析工具，然后通过其提供的`kernprof`命令来运行。



首先，测试原来的 tcp 5037端口方式。



打开终端，启动 ADB server：

```bash
# 先启动 ADB server
> adb kill-server
> adb -L tcp:5037 nodaemon server		# 指定 nodaemon 参数是为了使 ADB server 启动后不要退出当前进程 
```



再另外开一个终端，执行：`kernprof -l -v tadb/socket_adb.py `，结果如下：

```python
OKAY
bytearray(b'a2c0fe65\tdevice\n')
Wrote profile results to socket_adb.py.lprof
Timer unit: 1e-06 s

Total time: 0.000856 s
File: tadb/socket_adb.py
Function: main at line 45

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
    45                                           @profile
    46                                           def main():
    47         1          4.0      4.0      0.5      l_onoff = 1
    48         1          1.0      1.0      0.1      l_linger = 0
    49         1          2.0      2.0      0.2      if len(sys.argv) == 1:
    50         1         32.0     32.0      3.7          s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    51         1         23.0     23.0      2.7          s.setsockopt(socket.SOL_SOCKET, socket.SO_LINGER, struct.pack('ii', l_onoff, l_linger))
    52         1        165.0    165.0     19.3          s.connect((host, port))
    53                                               else:
    54                                                   s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    55                                                   s.setsockopt(socket.SOL_SOCKET, socket.SO_LINGER, struct.pack('ii', l_onoff, l_linger))
    56                                                   s.connect('/Users/edsion/Documents/fb-adb/build/adb.sock')
    57
    58         1          7.0      7.0      0.8      s.settimeout(10)
    59
    60         1         15.0     15.0      1.8      data = Protocol.encode_data('host:devices')
    61         1         29.0     29.0      3.4      s.send(data)
    62         1        447.0    447.0     52.2      state = s.recv(4).decode('utf-8')
    63         1         46.0     46.0      5.4      print(state)
    64         1         17.0     17.0      2.0      length = int(s.recv(4).decode('utf-8'), 16)
    65         1          2.0      2.0      0.2      recv = bytearray(length)
    66         1          5.0      5.0      0.6      view = memoryview(recv)
    67         1          8.0      8.0      0.9      s.recv_into(view)
    68         1         10.0     10.0      1.2      print(recv)
    69         1         43.0     43.0      5.0      s.close()
```



再来测试监听 Unix 套接字：



更换参数，重新启动 ADB server：

```
adb kill-server
sudo adb -L localfilesystem:/Users/edsion/Documents/fb-adb/build/adb.sock nodaemon server
```



再另外开一个终端，执行：`kernprof -l -v tadb/socket_adb.py 1 ` （这里通过传递给脚本的参数个数，区分是tcp还是unix套接字流程，实际对脚本其它逻辑没有影响），结果如下：

```python
OKAY
bytearray(b'a2c0fe65\tdevice\n')
Wrote profile results to socket_adb.py.lprof
Timer unit: 1e-06 s

Total time: 0.000514 s
File: tadb/socket_adb.py
Function: main at line 45

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
    45                                           @profile
    46                                           def main():
    47         1          4.0      4.0      0.8      l_onoff = 1
    48         1          1.0      1.0      0.2      l_linger = 0
    49         1          1.0      1.0      0.2      if len(sys.argv) == 1:
    50                                                   s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    51                                                   s.setsockopt(socket.SOL_SOCKET, socket.SO_LINGER, struct.pack('ii', l_onoff, l_linger))
    52                                                   s.connect((host, port))
    53                                               else:
    54         1         24.0     24.0      4.7          s = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    55                                                   # s.setsockopt(socket.SOL_SOCKET, socket.SO_LINGER, struct.pack('ii', l_onoff, l_linger))
    56         1         27.0     27.0      5.3          s.connect('/Users/edsion/Documents/fb-adb/build/adb.sock')
    57
    58         1          7.0      7.0      1.4      s.settimeout(10)
    59
    60         1         12.0     12.0      2.3      data = Protocol.encode_data('host:devices')
    61         1         19.0     19.0      3.7      s.send(data)
    62         1         84.0     84.0     16.3      state = s.recv(4).decode('utf-8')
    63         1         59.0     59.0     11.5      print(state)
    64         1         13.0     13.0      2.5      length = int(s.recv(4).decode('utf-8'), 16)
    65         1          3.0      3.0      0.6      recv = bytearray(length)
    66         1          5.0      5.0      1.0      view = memoryview(recv)
    67         1         21.0     21.0      4.1      s.recv_into(view)
    68         1         10.0     10.0      1.9      print(recv)
    69         1        224.0    224.0     43.6      s.close()
```



然后，通过例如`state = s.recv(4).decode('utf-8')`分析这行代码的耗时，来对比两种方式进行通信的时间：

| tcp:5037 | localfilesystem:adb.sock |
| :------: | :----------------------: |
|   447    |            84            |



当然一次结果不能说明问题，我又进行了多次对比，发现最终结论依然是 Unix 套接字远比 TCP 快。

后来又查了一下资料才明白，问题出在 AF_UNIX 与AF_INET 上：

{% asset_img 1.jpg [AF_UNIX] %}

AF_UNIX 域绕过了链路层、IP层、TCP层去除头部、检查校验等，直接从操作系统的内核缓冲区进行的通信，所以速度快很多。



## 结语

这是我这两天研究出来的一点微小成果，考虑到开发效率的原因，暂时还是先用最熟悉的 python 实现整个流程能够跑通，验证方案整体的可行性。对于之后的计划，可能还要考虑用更高性能的跨平台的语言，实现更底层的功能，例如用 Go 来重写 ADB server。