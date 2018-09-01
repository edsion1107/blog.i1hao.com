---
title: Hexo + GitHub Pages + Travis CI 自动化部署静态博客
date: 2018-09-01 16:58:09
update: 2018-09-01 16:58:09
categories:
    - 经验分享
tags:
    - hexo
    - GitHub Pages
    - Travis CI
    - 最佳实践
    - 2018
description: 2018最新 Hexo + GitHub Pages + Travis CI 自动化部署静态博客最佳实践。
---

# Hexo + GitHub Pages + Travis CI 自动化部署静态博客

[TOC]

## 开篇

做测试已经8年了，之前没有写博客的习惯，一方面对于知识沉淀做的不好，另一方面不符合分享的互联网精神。而且因为做的项目很杂，踩了很多坑，都说好记性不如烂笔头，所以今年初就萌生了写博客的想法。

本文主要以踩坑为主，对于一些过于基本的操作，比如安装各种工具，可能不会详细到步骤的介绍。

Let's go!

## 方案选择

### 静态博客生成

市面上静态博客的生成方案很多，开始我尝试了自建服务器 + caddy + hugo，折腾了一下感觉还是有些缺点：

- hugo 虽然速度快，但是主题偏少，各种功能插件还比较欠缺，需要自己动手的地方比较多
- Hugo 还比较“年轻”，文档比较简单，遇到问题查起来也比较痛苦

当然还有静态博客的方案，但是一番纠结最终选择了 hexo。

> 有种说法：hexo 毕竟是 node.js 开发的，搞前端的也比较擅长各种布局，所以主题的数量和质量都相对较高

### 部署环境

最开始使用的是腾讯云服务器+CDN+对象存储等各种云服务，好处就是自定义程度高，可优化空间大（比如都不支持 tls1.3 的时候，自己编译 Nginx 实现），不足的地方就是太折腾了，而且每年各种服务的费用加起来也要两三百块钱，如果不想打广告这钱就得自己掏腰包。

所以最终我使用的是 Github Pages，一方面免费、对程序员友好，另一方面大厂的服务非常可靠，唯一缺点就是偶尔被墙，不过因为我考虑的是自建服务器作备用站，所以问题不大。

### Markdown

markdown的好处我就不啰嗦了，我列举几个注意的地方：

- markdown 编辑器非常多，我本人最喜欢的是 vscode（尤其是 markdownlint 插件，帮我写出符合标准的文档）和 Typora（免费、跨平台，可以自定义主题、css），根据个人喜好选择。

- 写了静态博客，可能忍不住想顺手把微信公众号也搞定。但是公众号编辑器不支持 markdown，因为很多文章涉及代码、引用外部图片资源等，简单搜了下还没找到我认为合理的解决方案，所以这个问题先放置。

- 基本的 markdown 虽然够用，但是很多标准对其做了扩展，推荐使用比较流行的`GitHub Flavored Markdown`（简称`GFW`）。



### 代码托管（数据备份）

为了数据安全（这里指的是不丢失，不是不被盗用）我们一定要做数据备份。数据备份有个 3-2-1 规则（您的数据应该被存储3次，你应该使用2种不同的技术，应始终将1个备份保存在工作场所之外）。既然博客是公开的，而且也希望更多的人看到和分享，那么，博客的“代码”（markdown等）都应该是开源的。如果我们把文章都当作代码来处理，那么有个非常好用的工具是 git。而 git 有本地分支和远程分支的概念，而且远程分支可以指定多个。

所以，我们可以把代码分为 本地分支、远程分支-GitHub、远程分支-Gogs（自建的轻量级代码托管服务）。而且 Gogs 可以搭建在家里的 NAS （没有的话路由器+外挂硬盘也可以），通过路由的端口转发，实现完备的数据备份方案。

当然，如果你不想开源，用 Github 的私有仓库（收费）或者仅使用自建的代码托管服务也是可行的。


### 总结

经过一番考察，最终我的环境是这样的：

- 开发（书写）环境： 

  - macOS 10.13.6 （Windows当然也可以）
  - [vscode](https://code.visualstudio.com/download) （有些地方需要修改配置/代码，选一个顺手的编辑器就行）
  - [Typora](https://typora.io) (Markdown编辑器，也是选自己顺手的)

- 正式服务器：[GitHub Pages](https://pages.github.com)

- 备用服务器：

- - [gogs](https://gogs.io) ，自建的轻量级代码托管服务
  - [腾讯云主机](https://cloud.tencent.com/redirect.php?redirect=1001&cps_key=d61d10b4df29eee56b529f879e701bb2&from=console) ，1C1G1M的最低配即可
  - [腾讯云CDN](https://cloud.tencent.com/redirect.php?redirect=1027&cps_key=d61d10b4df29eee56b529f879e701bb2&from=console) 
  - [腾讯云对象存储](https://cloud.tencent.com/redirect.php?redirect=1005&cps_key=d61d10b4df29eee56b529f879e701bb2&from=console) 
  - [Let's Encrypt](https://letsencrypt.org/) + Google DNS ，或 [腾讯云SSL证书与Dnspod](https://cloud.tencent.com/redirect.php?redirect=1009&cps_key=d61d10b4df29eee56b529f879e701bb2&from=console) 。推荐 Google DNS（支持DNS CAA），每月仅需0.29美元，但是我在申请 Let's Encrypt 的通配符证书时用的是 dns 的方式验证域名所有权，[acme.sh](https://acme.sh) 这个工具不支持 Google DNS ，所以最后不得不迁回国内（国内DNS得实名，比较麻烦，但是有对国内运营商优化；国内DNS在使用国外服务时，有可能对方解析不出来，导致使用异常）

- 部署： [Travis CI](https://travis-ci.com) 注意：Travis CI分 https://travis-ci.com 和 https://travis-ci.org ，目前（2018年）正在从 org 迁移到 com，两者并不通用。打开网站时，一定要多多注意域名，进错了会导致命令行工具不可用、授权配置等出错

- 监控和其他服务：

  - 腾讯云网站备案（只用 GitHub Pages 不需要备案，但是为了将来使用各种服务免去麻烦，还是走一下流程吧，只是比较慢而已，并不麻烦。另外，别忘了还有个公安机关备案）
  - 腾讯云拨测 （告警服务，网站挂了第一时间得到通知；如果想得到微信通知可以试试`server酱`这个免费服务）
  - 百度统计

  - 谷歌分析

废话有点多了，下面就正式开始了。



## 环境搭建

我把这套方案，分为简单和完整两种环境。

### 书写环境

参照下面的完整环境，如果把整站搭建起来、自动化部署都完成以后，实际本地撰写文章时，只需要一个 git 客户端、一个 markdown 编辑器即可。

极端情况，只要能访问 github，本地什么软件都不需要安装，可以直接在 github 网站上编写和修改博客。

### 完整环境

这里默认你是一个程序员，对`git`、`homebrew`等命令行工具能熟练使用。（不会的话，遇到不懂的勤用 google ，强烈不建议用百度，因为你搜到的大概率是错误的教程和示范）。

####  1. 安装 Node.js

[Node.js](https://nodejs.org/en/) macOS 使用homebrew 安装，win 直接下载安装包双击即可。

```bash
brew install node
```



> 推荐使用 [nvm](https://github.com/creationix/nvm) 解决需要多个 node.js 并存的问题。如果你不知道这一条什么意思，那忽略即可。

#### 2. 安装 yarn（可选）

安装了 node.js 以后，自带包管理工具 `npm`，但是比较慢、还有些其他缺点我觉得比较难接受，所以如果你坚持用`npm`，那么请自行查阅文档，替换之后的命令。

```bash
brew install yarn	# 这个工具是全局安装的，与 node.js 版本无关
```

> yarn 不能解决墙的问题。如果没有稳定的梯子，可以使用国内镜像来解决

#### 3. 安装 hexo，并执行初始化

直接上命令：

```bash
yarn global add hexo	# 全局安装
hexo init blog.i1hao.com    # 初始化，blog.i1hao.com 是我的文件夹（工程）的名字，可自行修改
```

全局安装，是为了初始化和之后使用比较方便，但是有可能会导致切换 node.js 版本后 hexo 丢失，重装即可。



当然，你如果有洁癖，也可以像我一样尝试非全局安装：

```bash
mkdir blog.i1hao.com	# 创建一个文件夹
yarn add hexo    # 非全局安装hexo（安装在当前目录的 node_modules 文件夹）
node_modules/.bin/hexo init blog.i1hao.com    # 初始化一个文件夹（临时）
mv blog.i1hao.com/* . && rm -rf blog.i1hao.com    # 把 hexo 生成的文件移到当前目录，并删除临时文件夹
```

安装到当前路径，使用 hexo 的命令是`node_modules/.bin/hexo`，相对麻烦一点。

> 后面部署会用到 Travis-CI，你会发现我在 `.travis.yml`中没有使用全局安装。这是因为我在log中发现有这么一行代码：`export PATH=./node_modules/.bin:$PATH`，也就是说平台会把它加到环境变量里。所以为了精简构建的脚本，我采用的是非全局安装

#### 4. 验证 hexo 安装成功，安装依赖并初始化 git 仓库

上一节已经安装的 hexo ，其实全名是 hexo-cli，也就是hexo的命令行工具，hexo 要运行本身还有各种依赖，另外 hexo 还有插件机制，这里列出安装依赖和插件的命令：

```bash
yarn install    # 安装依赖（hexo init的时候生成 package.json，里面列出了所需依赖）
yarn add hexo-pdf    # 安装 pdf 插件，安装其他插件也是一样的

yarn upgrade    # 如果希望更新依赖执行此步，正常不需要
```



先验证一下hexo没问题（下文如果你采用了非全局安装，脑内自动补全 hexo 的路径吧）：

```bash
$ hexo server
INFO  Start processing
INFO  Hexo is running at http://localhost:4000 . Press Ctrl+C to stop.
```

如果看到类似这样的 log，证明启动成功，浏览器再打开`http://localhost:4000`确认下没问题即可。



之前提到过，基于数据备份的考量，需要使用 git 对项目进行管理，所以我们先初始化当前文件夹为 git 仓库：

```bash
$ git init 
Initialized empty Git repository in /Users/edsion/Documents/blog.i1hao.com/.git/
```



如果没问题，再配置当前项目的用户名和邮件地址（用于 github 和 gogs 识别你的身份）。

```bash
# 这里我建议不要用全局配置，配置只针对当前项目
git config --local user.name edsion1107    # 配置用户名
git config --local user.email xxx@xxx.com    # 配置电子邮件地址
git config --local -l    # 查看配置的结果
```
注意：用户名和邮箱不要配置错了，最容易犯的错误就是用户名、登录账号、昵称等不一致，不知道该用哪个。这里有个小技巧，不管是 GitHub 、Gogs 、Gitlab，只要你建一个仓库，那你的仓库一定是 `user.name`/`仓库名`的格式命名的，所以用这个就对了。

> 用户名、电子邮件地址如果配置错误，不影响你的推送和拉取代码（根据你使用的是 HTTP(S) 协议，还是 SSH 协议，可能需要输入账号和密码），但是在平台展示的 log 里就会变成没有头像的“黑户”了

下一步，在 Github 和 Gogs 平台上，分别手动创建一个仓库，同样命名为 `blog.i1hao.com`，成功后该仓库首页会显示地址，记下来一会儿用。

#### 5. 安装 Next 主题

hexo 的主题是非常多的，去官网挑一个自己喜欢的就行了。这里我以常见的 [Next](https://github.com/theme-next/hexo-theme-next) 主题为例介绍我是怎么安装和使用的。

> 注意 Next 主题有两个仓库，一个是放在开发者 [iissnan](https://github.com/iissnan/hexo-theme-next) 个人名下的，一个是放在组织 [theme-next](https://github.com/theme-next/hexo-theme-next) 名下的。6.x以后的新版本，都是放在后面这个仓库，前面那个已经不更新。有个BUG，Next 主题的官方网站和文档，还是指向已经废弃的仓库，这一点一定要小心。

将 Next 主题的仓库作为子模块，添加到项目中：

```bash
git submodule add https://github.com/theme-next/hexo-theme-next.git   themes/next
```

这里要注意在当前git仓库根目录下执行，也就是说，执行这条命令后，`themes`目录下会多出来一个`next`文件夹，它是我们要使用的主题。

如果将来要更新主题（子模块），则执行：

```bash
# 方案一，在仓库根目录下执行
git submodule foreach git pull
git add .
git commit -m "更新主题"

# 方案二，在Submodule的目录下面更新
cd themes/next
git pull
cd ../../ 
git add .
git commit -m "更新主题"
```

这种子模块的方式，可以保证你始终使用的是该主题最新代码。在子模块里可以切换到指定分支、指定tag等操作，使用起来非常灵活，但是需要较高的学习成本。如果无法接受，下载压缩包直接解压就好了。



#### 6. 修改配置

关于 hexo 和主题的配置，网上的教程很多，单独拿出来能写好几篇文章了，太占篇幅所以我直接放例子供参考吧：

[blog.i1hao.com/_config.yml](https://raw.githubusercontent.com/edsion1107/blog.i1hao.com/master/_config.yml)

 #### 7. 添加和推送远程仓库

网址就是前面通过 GitHub 和 Gogs 创建的仓库地址：

```bash
git remote add origin git@github.com:edsion1107/blog.i1hao.com.git    # 添加 github
git remote add origin http://xxxx.com:edsion1107/blog.i1hao.com.git    # 添加自建 gogs
git remote -v    # 查看远程仓库
git push --set-upstream origin master	# (首次)推送到远程仓库
```

首次推送到远程仓库，会提示设置`upstream`，之后就可以只用`git push`命令了。

细心的你，可能注意到了，我的 GitHub 地址为什么没带http(s)，这是因为我使用的是ssh协议；但是因为我自建的 Gogs 没有配置好证书和端口，所以只支持http。

关于 git 的 ssh 协议使用，建议英语不是太差的，直接阅读 Github 文档；实在有困难，也尽量不要用百度搜索，搜出来的文章错误太多（写教程的太不专业，局限性很大，而且互相抄把错误也抄过来了）！

> 注意，hexo 本地运行没问题了，需要先执行 `git add .`和`git commit -m "initialization"`，提交一次（到本地），再执行`git push`。否则，不会产生master分支。

#### 8. Travis-CI 部署

首先，用 Github 账号登录Travis-CI网站 ：https://travis-ci.com （千万别用org的域名，那个是旧版网站，原因前面解释过）。

等网站把你的项目同步过来，找到对应项目，点击 Settings ，跳转设置。这里面要添加一个`Environment Variables（即构建时用到的环境变量）`。

> 旧版 (https://travis-ci.org) 是每个项目有个开关，控制是否开启 CI 服务。新版是所有项目默认开启的，所以看旧版教程的同学，找不到就别找了！如果你有强迫症，去 Github 的设置找这个开关。

为了把构建结果再 push 给`GitHub Pages`，你需要生成一个 `Personal access tokens`。这个 token 作为环境变量提供给你的构建环境，相比于直接提供账号密码或 SSH Key，更安全，还能够控制权限。

访问 Github，找到 settings -> Developer settings -> Personal access tokens，点击 Generate new token，给你的 token 起个易于辨识的名字，然后勾选第一组 repo 下的所有权限，点击 Generate token 。

注意，生成的 `Personal access token` **只显示一次**，刷新、关闭当前页面后 token 不显示。安全起见，你不应该把它保存在任何文件中！正确的做法是，直接复制该 token，然后切换到刚刚的 Travis-CI 页面，创建一个环境变量名为 `GITHUB_TOKEN`（当然这个名字可以随意），把 token 粘贴进去然后保存，并立即清空剪贴板。

Ps： 这里的环境变量也是，只在创建时显示一次，之后只能删除无法查看。

此时打开我们的编辑器，创建一个名为`.travis.yml`的文件，保存在工程的根目录下。

以下是我的例子：

```yaml
language: node_js
sudo: required
# 指定node_js版本
node_js: 
  - lts/*
cache:	# 缓存，提高下次构建的效率
  yarn: true
  directories:
    - node_modules

install:    # 安装阶段（详细的各阶段顺序，请查询 Travis-CI 相关文档）
  - yarn install

script:    # 执行构建阶段
  - hexo clean    # hexo 的清理命令
  - hexo generate    # hexo 的生成（构建）命令，即最核心的生成静态文件过程

# GitHub Pages Deployment
deploy:    # 部署阶段
  provider: pages    # 约定 pages 为 GitHub Pages 服务，必须且不可更改
  skip-cleanup: true  # 必须跳过清理，否则过程中生成的文件（要发布的静态资源）会被清理
  github-token: $GITHUB_TOKEN  # 部署时使用的 token
  keep-history: false   # 设置为 false 时，使用 `git push --force` 命令覆盖历史记录
  on:
    branch: master    # 仅监听 master 分支的变化，才执行构建
  target-branch: gh-pages   # 用于存放静态资源的分支
  local-dir: public   # `hexo generate` 命令生成的静态资源所在路径
  fqdn: blog.i1hao.com    # 自定义域名
```

这里面每一项几乎都有注释，完整的解释请翻阅 Travis-CI 官方文档，我认为这几乎是最精简的配置了，如果要修改请仔细查阅文档。

我这里与其他人最大的不同，就是省去了“脱裤子放屁”环节，直接使用 Travis 的服务完成部署流程。如果你使用搜索引擎，那么能找到的方案不外乎以下两种：

1. 使用 hexo-deployer-git 插件
2. 使用脚本 `git push`生成的静态资源

这两种方案，最大的问题就是你的账号密码不安全，即使你使用的是环境变量，也不能保证你所使用的插件/脚本中不会将其输出到 log 中（ Travis-CI 不会明文输出环境变量到 log ）。

可能有人看到有上传 SSH key 的办法，虽然确实是经过加密存储了，但是这样也把完整权限提供给了平台/构建脚本，一旦泄漏产生的后果也是比较严重的。



另外还需要注意：

1. 这里会把生成的静态博客发布到`gh-pages`分支。相对于采用两个仓库分开，这样设计管理起来更加容易。只需要你在当前仓库的 settings （注意不是个人的设置）-> GitHub Pages -> Source（来源）下面，切换到该分支（`gh-pages`）即可。
2. 设置了`keep-history: false`。保证该分支很“干净”，只有最后一次的提交记录。
3. 如果你看了帮助文档，其实还可以指定`gh-pages`分支的提交人，默认的是`traviscibot`，我觉得还挺萌的，就没改。
4. 通过`fqdn`字段，我指定了自定义域名。这个不是只在这配置就行了，还需要去 DNS 服务商那配置 CNAME，在项目的 settings 里也配置相同的自定义域名。
5. 之前没用 Travis-CI 自动部署的时候，我本地使用的是 hexo-deployer-git 插件，我发现每次更新了文章都要重新去 github 仓库重新配置一下自定义域名（自动丢失了），否则不生效。
6. 前面提到了 `gh-pages`分支，如果有强迫症，只想在 github 保留这个分支，本地和自建的 gogs 不要这个分支，那么：在本地先删除 gogs 远程仓库的 url -> 创建 gh-pages 分支（不能直接创建远程分支、必须是本地建立分支推送到远程仓库，再删掉本地分支）-> 推送到到远程仓库 -> 删除本地分支 -> 把 gogs 远程仓库的地址加回来
7. 虽然显示构建成功，但是部署不一定成功。平台通过命令执行的返回值去判断结果，`hexo generate`命令无论成功失败永远返回`0`（成功）。这个时候只能通过 log 排查，相信你遇到一次就知道以后怎么解决了。

#### 9. 其他问题和注意事项

如果你是首次使用和部署 hexo 到 GitHub pages，那么你可能还会遇到：

- 部署成功，但是页面排版错乱（其实是css、js都404了）。此时分别创建文件`source/.nojekyll`和`deploy_git/.nojekyll`，内容为空。再在`_config.yml`的根结点添加：

  ```yaml
  include:
    - .nojekyll
  ```

  即可解决。—— 应该是使用 hexo-deployer-git 插件的问题，不使用的话可能不需要。

- 首次部署，Github 设置 CDN 、配置服务器可能需要一点时间，耐心等待。



#### 10. 待解决问题

1. Lighthouse 未达到满分（500分）—— 根据 v2ex 上网友分享的经验去解决
2. 开始说的备用服务器问题还未完全部署好，暂时不可用
3. 文章输出到公众号的问题，还没有一个自动化的、轻量级的方案（目前正文中含代码，会有排版错误/代码高亮问题），markdown here插件的效果很不理想



## 最后

放上博客代码的 GitHub 地址： [https://github.com/edsion1107/blog.i1hao.com.git](https://github.com/edsion1107/blog.i1hao.com.git) 

如果有问题，欢迎来提 [Issues](https://github.com/edsion1107/blog.i1hao.com/issues)  （因为未开放评论功能）。

欢迎打赏，或点击文中腾讯云服务（用的是推广链接）对我进行赞助。

