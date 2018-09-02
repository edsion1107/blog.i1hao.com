# README.md

[![Build Status](https://travis-ci.com/edsion1107/blog.i1hao.com.svg?branch=master)](https://travis-ci.com/edsion1107/blog.i1hao.com)

## 环境和工具

### 必备

- [git](https://git-scm.com/downloads)
- [vscode](https://code.visualstudio.com/download) 和 [Typora](https://typora.io)（ markdown 编辑器，根据喜好选择）

### 可选

因为已经改为 Travis-CI 服务部署，所以本地可以不安装 hexo

- node.js ( nvm & npm )
- yarn
- hexo
- NexT

### 安装 hexo 和插件

> 如果使用 Travis CI 进行部署，本地可以不再安装 hexo 和插件。

#### npm 安装

```bash
npm install -g hexo # 全局安装
npm install    # 安装的依赖已在 package.json 中定义
```

### yarn 安装

```bash
yarn global add hexo    # 全局安装
yarn install    # 安装的依赖已在 package.json 中定义
```

### 安装 NexT 主题（为git仓库添加子模块）

```bash
git submodule add https://github.com/theme-next/hexo-theme-next.git   themes/next
```

### 更新NexT主题

```bash
cd themes/next
git pull
```

## 常用命令

### 创建新文章

```bash
# 方式一（本地安装了hexo）：
hexo new xxxx
# 方式二（未安装hexo）：
touch xxxx.md   # 注意命名不使用下划线，而是使用中划线“-”，而且全部用小写字母
mkdir xxxx  # 创建同名资源文件夹
```

### 启动本地调试服务器

```bash
hexo server
```

> **注意**：修改`_config.yml`需要重启server

### 发布

#### 手动（需安装 hexo 和 插件 ）—— 已废弃，因为不再使用 hexo-deployer-git 插件

```bash
hexo clean
hexo g
hexo d
```

#### 自动

因为借助 Travis-CI 实现了自动部署，所以直接`git push`即可。

```bash
git add .
git commit -m "xxx"
git push
```
