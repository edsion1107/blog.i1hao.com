---
title: pytest 优化 - fixture 中获取用例描述
date: 2019-09-19 21:52:49
tags:
    - pytest
---
今天在实践pytest过程中，有这么一条需求：
> 用例描述写在`test_xxx`函数的docstring（文档字符串）中，如何在代码中获取该字符串？

开始想到的笨办法，是通过 request 这个 fixture，拿到执行的case所在文件、行数等，去解析文件读取。

翻了半天 pytest 文档中没有找到有用的信息，最后经过调试发现可以通过以下代码实现：

```python
request._pyfuncitem._obj.__doc__
```

下一篇，分享向 xml 中添加字段、属性。