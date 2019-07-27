---
title: 使用腾讯软件源，为 homebrew 加速
date: 2019-07-27 19:31:55
tags:
    - 腾讯云
    - homebrew
    - 镜像
    - macOS
---

[Homebrew](https://brew.sh/index_zh-cn) 是一款自由及开放源代码的软件包管理系统，用以简化macOS系统上的软件安装过程。  
其核心是一个托管在 GitHub上 的 git 版本库，由于某些原因，国内访问速度不快，并且稳定性堪忧。  

为了提高体验，我们可以用[腾讯软件源](https://mirrors.cloud.tencent.com/#/index)对其进行加速，打开“终端”，依次执行以下命令即可：  

```bash
# 替换 Homebrew 的 formula 索引的镜像（即 brew update 时所更新内容）
git -C "$(brew --repo)" remote set-url origin https://mirrors.cloud.tencent.com/homebrew/brew.git
git -C "$(brew --repo homebrew/core)" remote set-url origin https://mirrors.cloud.tencent.com/homebrew/homebrew-core.git

# 替换 Homebrew 二进制预编译包的镜像
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.cloud.tencent.com/homebrew-bottles' >> ~/.bash_profile
source ~/.bash_profile

#（可选）禁用“自动更新” —— 执行`brew install xxx`时会自动执行`brew update`，导致安装过程比较慢。
# 通过此方法禁用后，选择合适时机手动执行`brew update`
echo 'export HOMEBREW_NO_AUTO_UPDATE=1' >> ~/.bash_profile
source ~/.bash_profile

# 更新
brew update
```  

复原：
```bash
git -C "$(brew --repo)" remote set-url origin https://github.com/Homebrew/brew.git

git -C "$(brew --repo homebrew/core)" remote set-url origin https://github.com/Homebrew/homebrew-core

brew update

# Homebrew-bottles: 需要去`~/.bash_profile`中删除'HOMEBREW_BOTTLE_DOMAIN'
```
