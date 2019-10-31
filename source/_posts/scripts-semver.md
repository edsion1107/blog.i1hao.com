---
title: 浅谈自动化测试的版本控制
date: 2019-10-31 14:47:27
tags:
    - git
    - SemVer
    - pytest
---

随着项目逐步迭代，自动化覆盖率提升，自动化测试的脚本会变得越来越复杂，我们需要在脚本中引入版本控制。

这里我举几个遇到过的例子：

- 某项目需要同时测试多个app，并且最终的数据要汇总到一起。因为不同的app（的测试代码）由不同的人员去维护，经常会导致负责公共模块的同学更新后，某些app没有及时更新，最终执行时需要人为去调查和解决冲突问题。
- 有这样一家公司，为了整合测试资源，在内部搭建了一个云测试平台，把部署在全国各地多个节点统一进行调度。但是由于平台不够稳定，经常是在平台测试不通过，但是相关同学在本地跑一样的代码却完全没有问题。经过一番调查，发现某个节点磁盘满了导致脚本没有更新。
- 小A是某公司的测试开发，在内部作为多个项目公共资源，主要负责各项目核心自动化脚本的编写。因为各个项目的发布频率不同，有的项目设置需要两个月才需要维护一次。每次到这个项目上时，小A都要花很久时间回忆之前有什么改动，本次需要做什么。公司也面临着人员离职或者磁盘损坏导致代码丢失的风险。

所以，为了解决这些复杂的问题，我们尝试在测试代码上也引入版本控制，并且参考了[《语义化版本控制规范》](https://semver.org/lang/zh-CN/)，为每一次交付的产物都制定一个版本。

> 关于版本控制的工具，首先推荐 git（与svn的优劣对比，不是本文重点，这里不展开）。要明确一点的是，git 不等于 github，即使不涉及多人协作，不借助各种托管平台，也可以作为本地仓库使用。

这里存在一个小小的问题，实际应用中怎么打包、确定版本号、有了版本号怎么快速找到对应代码，下文就逐步剖析我目前采用的方案和流程。

ps：下文主要以 python 为例。
<!-- more -->

## 解题思路

---------------
根据官方文档：[single-sourcing-package-version](https://packaging.python.org/guides/single-sourcing-package-version/)，有多种方式可以用来维护项目的版本号：

1. 读取 setup.py，并使用正则表达式解析版本。
2. 使用外部构建工具来管理两个位置的更新，或者提供两个位置都可以使用的API。
3. 在项目中某个模块添加`__version__`全局变量（例如version.py），使用时（如 setup.py ）导入 。
4. 将值放在一个简单的`VERSION`文本文件中，并让 setup.py 和项目代码读取它。
5. 在 setup.py 中设置，项目代码中使用 pkg_resources API 获取。
6. 在 `sample/__init__.py` 中将设置为`__version__`，并在 setup.py 中导入 sample。
7. 将版本号保留在版本控制系统的标签（Git，Mercurial等）中，而不是保留在代码中，然后使用 [setuptools_scm](https://pypi.org/project/setuptools-scm/) 自动将其提取。

如果没尝试过完整开发和发布过一个模块，看到这个可能会一脸懵逼。反复提到的`setup.py`和`__version__`是啥，代码中怎么用？感觉上面有几个方案差不多是一回事啊。

这里的`setup.py`是python里一种标准的组织代码的方式，声明了依赖、版本、作者等等各种信息。起初我也花费了大量精力去研究这种结构，但是时间久了发现这个方向有点偏离初衷，给简单的项目引入了一系列复杂的问题：

- 学习成本高。setup.py 有非常多的配置项，如果是打包供第三方调用，确实是非常好的一个标准。但是正是因为其配置过于复杂，在理解不深时比较容易出错。（例如`setup_requires`和`install_requires`,`tests_require`的区别）
- 不擅长依赖管理。[这里](https://github.com/pypa/pipenv/issues/1161#issuecomment-349972287)有一句非常经典的话解释这个问题：**Pipenv is for applications. Setup.py is for packages.**
- setup.py 中的 test，不是设计用来组织测试代码的，而是用来测试代码的，这块代码一般不会打包到最终产物中。或者说，test相关的功能是用来对你的代码进行单元测试的。这里非常容易陷入一个怪圈：我的代码是设计用来测试某个app的，我需要写测试代码（单元测试）来测试我的代码吗？那我是否还需要写测试代码，来测试我的测试代码的测试代码？
- 某些情况下，setup.py 打包出来的代码，部署在同一台机器上，可能面临环境隔离和权限问题。
- 个人比较推荐 pytest 这个框架组织用例，但是 setup.py 要求我们的代码要放到一个包（文件夹）内。这样在 pycharm 调试时，直接点击函数/方法前的箭头执行单条 case 的调试时会有一点小问题，需要每次修改 Configurations，带来一点点的不便。

所以综合以上几条，我尝试了一个简化版的方案：在某个关键文件内，添加`__version__`全局变量，然后通过[bump2version](https://github.com/c4urself/bump2version)“自动”更新版本号，并且在版本号改变后自动提交到git。

这里第一个自动加引号，是因为更新版本号时要手动指定更新的是主要版本号、次要版本号还是修订版本号，代码是没法自动判断的。
> 其实有[semantic-release](https://github.com/semantic-release/semantic-release)和[python-semantic-release](https://github.com/relekang/python-semantic-release)这种工具能按照所谓的《语义化版本控制规范》来自动更新，但其本质是依赖 git 的 commit message，需要按照规范编写，考虑学习成本还是不推荐了。

## 为 python 代码添加版本号

---------------

首先，准备一个包含测试脚本的工程如下：

```Shell
.
├── Pipfile             # Pipfile 和 Pipfile.lock 是 pipenv 用来管理工程依赖的
├── Pipfile.lock
├── README.md           # 工程的说明文件，记录一些常用的命令、安装、部署流程等
├── config.yaml         # 工程配置文件（方便测试代码的参数化）
├── conftest.py         # pytest 的 fixture
├── pytest.ini          # pytest 的配置文件
├── test_aweme.py       # test_xxx 是 pytest 对测试脚本命名的约定
├── test_ilife.py
├── test_news.py
├── test_wechat.py
└── utils.py            # 一些工具方法
```

初始化本地 git 仓库：

```Shell
git init
git add .
git commit -m "init"
```

安装依赖：

```Shell
pipenv install --dev bump2version gitpython  # dev主要是本地打包、性能优化等需要的依赖，正式环境部署时不需要。这也是使用pipenv的理由之一
```

创建配置 bump2version 的配置文件`.bumpversion.cfg`，内容如下：

```INI
# filename: .bumpversion.cfg
[bumpversion]
current_version = 1.2.0     # 当前版本号
commit = True               # 自动提交（Git）
tag = True                  # 自动Tag

[bumpversion:file:conftest.py]
```

bump2version 的其他配置可以查看[文档](https://github.com/c4urself/bump2version/blob/master/README.md)，还可以修改 commit message 等。这里的`bumpversion:file:conftest.py`指定了版本号位于文件`conftest.py`中。所以，下一步是添加`__version__==1.2.0`到`conftest.py`文件中（修改后不要忘记`git commit`）。

> 为什么是`conftest.py`，不是`__init__.py`或者`VERSION`文件，这是根据我的项目实际情况（使用了pytest框架，一般都会有`conftest.py`），另外也是为了减少文件数量。

执行以下命令进行环境测试，检查配置文件是否正确：

```Shell
$ pipenv run bump2version --allow-dirty --dry-run --list patch
current_version=1.2.0
commit=True
tag=True
new_version=1.2.1
```

`--dry-run`表示不会实际执行版本号更新，patch 是更新修订版本号，minor 是更新次要版本号，major 是更新主要版本号。如果终端中最后显示`new_version = 1.2.1`那说明这一步是正确的了。

下一步，执行更新试试：

```Shell
$ pipenv run bump2version --allow-dirty --list patch    # 工程中免不了有未提交的文件，所以加--allow-dirty忽略
current_version=1.2.0
commit=True
tag=True
new_version=1.2.1
```

执行此命令后，如果`conftest.py`中的`__version__ = '1.2.1'`，而且执行`git log`和`git tag -l`显示了新的提交记录和 tag ，则表示一切ok。

最后执行`git reset HEAD^ --hard`把改动复原（不要忘记删除新增的tag）。

## 通过 git archive 命令打包

---------------

可能看到标题有的人会问，直接压缩文件夹不可以吗？主要是手动操作比较容易出错。常见的就是有时候多带一层文件夹，有的时候又忘记带，这样结构变来变去容易出问题。另外，如果你是使用的 macOS，直接右键压缩，还会带上`__MACOSX`等各种隐藏文件，或者又有可能把一些想打包进去的“隐藏文件”（.开头，一般是配置文件）忘记打包。

用到`git archive`命令，就不得不提到`.gitignore`和`.gitattributes`这两个文件。前者指定哪些文件会被 git 忽略（不加入版本控制，自然打包的时候也就不会包含），后者（通过为文件指定`export-ignore`属性）声明哪些在版本控制中的文件打包时不需要包含。

以下是我的一个小例子：

```TXT
# filename: .gitignore
### macOS template
# General
.DS_Store
.AppleDouble
.LSOverride

# Icon must end with two \r
Icon

# Thumbnails
._*

# Files that might appear in the root of a volume
.DocumentRevisions-V100
.fseventsd
.Spotlight-V100
.TemporaryItems
.Trashes
.VolumeIcon.icns
.com.apple.timemachine.donotpresent

# Directories potentially created on remote AFP share
.AppleDB
.AppleDesktop
Network Trash Folder
Temporary Items
.apdisk

!/.pytest_cache/
```

```TXT
# filename: .gitattributes
.bumpversion.cfg     export-ignore
.gitattributes       export-ignore
.gitignore           export-ignore
README.md            export-ignore
```

最后，尝试以下命令打包（采用的是打包时间+版本的命名方式）：

```Shell
git archive -o `date +%Y%m%d%H%M`_v1.2.0.zip v1.2.0     # 打包 tag 是 v1.2.0的版本
git archive -o `date +%Y%m%d%H%M`.zip HEAD              # 打包最新版本
```

> 注意：对`.gitignore`和`.gitattributes`两个文件，是需要先提交才能生效。

## “自动化”打包

---------------

最后，我在 utils.py 中添加了一段代码，来帮助我实现上面这一堆繁琐的命令：

```Python
import time

def git_archive():
    """git打包最后一个标签的包，并以时间+版本号命名"""
    from git import Repo
    repo = Repo('.')
    latest = repo.tags[-1]
    with open(f'{time.strftime("%Y%m%d%H%M")}_{latest}.zip', 'wb') as fp:
        repo.archive(fp, treeish=latest, format='zip')


def archive():
    """更新版本号，然后打包测试脚本"""
    choose = input("""选择要升级的版本号：
    0. 跳过版本号更新，直接以最近一个tag重新打包
    1. 主版本号，1.0.0 → 2.0.0
    2. 次要版本号，1.0.0 → 1.1.0
    3. 修订版本号版本号，1.0.0 → 1.0.1

    """)
    part = None
    skip = False
    if choose.strip() == "0":
        skip = True
    elif choose.strip() == "1":
        part = 'major'
    elif choose.strip() == "2":
        part = 'minor'
    elif choose.strip() == "3":
        part = 'patch'
    else:
        input('未知选项，回车退出')
        exit(-1)
    if not skip:
        from bumpversion import cli
        config_file = Path(__file__).with_name('.bumpversion.cfg')
        cli.main(['--config-file', str(config_file), '--allow-dirty', part])
    git_archive()

```

前面提到过，因为代码没法根据《语义化版本控制规范》自动更新版本，所以这里需要手动选择一下。

> 直接用 python 调用`bump2version`和`gitpython`代码，不用考虑环境、依赖等问题，比创建子进程性能也略优。

## 版本号的使用

---------------

根据前面的步骤，我们已经能够做到在代码中添加版本号了，只要在代码里**尽量靠前**添加 print 一下或者写入 log 即可。

但是如果在 pytest 中使用，因为框架的加载机制，并不一定能够真正显示出来。如果放在case的log里，一方面反复打印也没必要，另一方面如果是 fixture 、甚至是 `pytest_addoption`、`pytest_collection_modifyitems`等 hook 的函数中出错，也没法看到版本号。所以我目前采用的方案是在 conftest.py 里添加`pytest_cmdline_main`这个函数（基本上是第一个加载的 hook 了），在这里打印版本号到终端的标准输出（将来也许能找到更好的方案），来保证始终能显示版本号。

## 结语

目前很难找到一种方案满足所有要求，最终可能还得根据项目实际情况灵活调整，这里先做个抛砖引玉。目前测试开发这条路上，除了封装各种 Driver 不停“造轮子”以外，工程化实践上还有很长的路要走。
