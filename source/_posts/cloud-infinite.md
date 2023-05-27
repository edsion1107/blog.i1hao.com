---
title: 借助数据万象（原万象优图），让 hexo 也用上 webp
date: 2019-07-02 20:58:37
tags:
    - 数据万象
    - 腾讯云
    - hexo
    - webp
---

本博客目前是使用 hexo + Next 主题搭建在 GitHub Pages 上的，使用 git 管理，并接入了 Travis-CI 自动发布。一直以来，对于图片的处理是我的一块心病。虽然hexo官方提出了[资源文件夹](https://hexo.io/zh-cn/docs/asset-folders.html)的概念，但是`{% asset_img example.jpg This is an example image %}`这种方式几乎不被任何 Markdown 编辑器支持。

>个人习惯是使用 Typora 码字，然后再用 vscode 的 markdownlint 扩展进行语法检查（代码格式上的洁癖）—— vscode 用得顺手，甚至可以放弃 Typora。

为了不滥用GitHub仓库、节省空间和提高访问速度，可以使用腾讯云的[对象存储（COS）](https://cloud.tencent.com/redirect.php?redirect=10644&cps_key=d61d10b4df29eee56b529f879e701bb2) + [内容分发网络（CDN）](https://cloud.tencent.com/redirect.php?redirect=10502&cps_key=d61d10b4df29eee56b529f879e701bb2)。良心云每个月都赠送了一定的免费额度，对于个人博客来讲一般是够用的。

最近CDN也不能满足我的胃口了，在尝试极限优化的路上，我又发现了一个更有想象力的方案，那就是借助腾讯云的[数据万象（原万象优图）](https://cloud.tencent.com/redirect.php?redirect=10371&cps_key=d61d10b4df29eee56b529f879e701bb2)服务，对图片进行预处理或者实时处理，从而减小图片体积、提高打开速度。

对于数据万象我在之前的一篇文章[薅羊毛党的胜利 —— 合理利用腾讯云搭建服务
](https://blog.i1hao.com/2018/06/29/qcloud-free/)提到过。

<!-- more -->

>如果看完这篇文章觉得有帮助，欢迎点击我的推广链接：[https://cloud.tencent.com/redirect.php?redirect=10371&cps_key=d61d10b4df29eee56b529f879e701bb2](https://cloud.tencent.com/redirect.php?redirect=10371&cps_key=d61d10b4df29eee56b529f879e701bb2)，购买腾讯云相关服务，谢谢支持。

## 第一步，开启相关服务，并进行配置

数据万象本身是基于对象存储服务的，并且也可以开启CDN对其进行加速。这里三者之间的关系有点难理清，我在尝试了一番之后，推荐按照如下顺序进行操作：

### 1. 创建存储桶

访问[数据万象控制台](https://console.cloud.tencent.com/ci/bucket)，创建存储桶 Bucket（选择绑定-新建）。所属地域随便选择，访问权限“公有读私有写”，注意**不要**开启 CDN 加速。

> 刚开始先不要开防盗链、原图保护等高级功能，先把流程跑通再回来处理。

以下是我的配置页面，供参考：
<picture><source srcset="https://img.blog.i1hao.com/20190702221156.png?imageView2/format/webp" type="image/webp"><source srcset="https://img.blog.i1hao.com/20190702221156.png?imageView2/format/png" type="image/png"><img src="20190702221156"></picture>

### 2. 绑定自定义域名

为 Bucket 绑定自定义域名，并开启CDN。我的博客域名是`blog.i1hao.com`，所以我新建了一个子域名`img.blog.i1hao.com`用于此处。**注意自己的域名要提前备案**。使用自定义域名主要是好记，使用腾讯云提供的默认域名也不是不可以。

> 不使用自定义域名，可以在第一步直接开启CDN。创建时不建议开是为了缩短 CDN 的部署时间，毕竟部署一次要5分钟，我是不太想等的。

### 3. 为自定义域名申请 HTTPS 证书

都9102年了，HTTPS 应该是标配。证书颁发比较慢，所以尽量早点做好。这里就不推荐用 Let's Encrypt 的证书了，因为证书要用到 CDN 上，如果每三个月更新一次，就有点小麻烦了。
> 如果恰好用的是腾讯云的云解析，那你有福了。申请证书和配置到之后的CDN，只需要通过网页勾选几下就可以了，非常方便。

具体过程这里就不展开了，网上有大量文章介绍。

### 4. 配置CDN

这里主要分两步：配置 DNS 解析的 CNAME ，开启HTTPS。
下图是我在腾讯云云解析上的配置，如果用其它服务商，也可以参考：
<picture><source srcset="https://img.blog.i1hao.com/20190702221642.png?imageView2/format/webp" type="image/webp"><source srcset="https://img.blog.i1hao.com/20190702221642.png?imageView2/format/png" type="image/png"><img src="20190702221642"></picture>

至于 CDN 的配置，要去[内容分发网络的控制台](https://console.cloud.tencent.com/cdn)，在对应域名的“高级配置”中开启：
<picture><source srcset="https://img.blog.i1hao.com/cdn_console.png?imageView2/format/webp" type="image/webp"><source srcset="https://img.blog.i1hao.com/cdn_console.png?imageView2/format/png" type="image/png"><img src="cdn_console"></picture>

> 上图中还开启了HTTP2.0，也能一定程度上提高访问速度，建议开启。

稍等约5分钟，等待 CDN 部署完成。

### 5. 开启子账号，创建 API 密钥

为了您的账号安全，我强烈建议，对于腾讯云的每个产品都创建相应的子账号，并按照最小化权限原则进行设置。如果确定不想这么麻烦，可以省略这一步。

打开腾讯云[访问管理控制台](https://console.cloud.tencent.com/cam/overview)，从用户列表新建用户 - 子用户。
这里只要跟着流程走就可以了，没什么坑，我只放一个最终配置好的图吧（不用跟我保持完全一致）：
<picture><source srcset="https://img.blog.i1hao.com/20190702222910.png?imageView2/format/webp" type="image/webp"><source srcset="https://img.blog.i1hao.com/20190702222910.png?imageView2/format/png" type="image/png"><img src="20190702222910"></picture>

这一步完成后，需要记住账号ID、SecretId、SecretKey，之后会用到。

### 6. 为子账号配置 Bucket 的访问权限

如果未启用子账号，则跳过此步骤。
打开腾讯云[对象存储控制台](https://console.cloud.tencent.com/cos5)（这里比较混乱，数据万象中没有权限配置的入口），在存储桶列表中找到数据万象中用到的 Bucket，在“权限管理”页面将子账号加入，并选择合适的权限：
<picture><source srcset="https://img.blog.i1hao.com/20190702223537.png?imageView2/format/webp" type="image/webp"><source srcset="https://img.blog.i1hao.com/20190702223537.png?imageView2/format/png" type="image/png"><img src="20190702223537"></picture>

### 7. 配置验证

到这里，有关腾讯云的基础配置就完成了。现在需要上传一张图片到 Bucket，然后通过自定义域名访问，验证整个流程是否打通。

首先通过 web 页面上传一张名为`IMG_0526.png`的图片。此时通过页面可以查到，系统分配给我的原始URL为：`http://blog-i1hao-com-1251008864.picsh.myqcloud.com/IMG_0526.png`。如果此链接无法访问（一般不可能），说明是对象存储或数据万象的配置（例如读写权限）有问题。

所以，通过我的自定义域名（CDN）访问此图片的地址为：`https://img.blog.i1hao.com/IMG_0526.png`。如果此处无法访问，大概率跟 CDN 有关。检查 CNAME 是否生效（nslookup命令），CDN 是否在部署中，和 HTTPS证书、回源策略等。

通过数据万象处理为 webp 格式的图片链接为：`https://img.blog.i1hao.com/IMG_0526.png?imageView2/format/webp`。
<picture><source srcset="https://img.blog.i1hao.com/20190702224558.png?imageView2/format/webp" type="image/webp"><source srcset="https://img.blog.i1hao.com/20190702224558.png?imageView2/format/png" type="image/png"><img src="20190702224558"></picture>
数据万象添加水印后的图片链接为：`https://img.blog.i1hao.com/IMG_0526.png?watermark/2/text/YmxvZy5pMWhhby5jb20%3D`
<picture><source srcset="https://img.blog.i1hao.com/20190702225242.png?imageView2/format/webp" type="image/webp"><source srcset="https://img.blog.i1hao.com/20190702225242.png?imageView2/format/png" type="image/png"><img src="20190702225242"></picture>

后面这两个链接能访问，并且结果格式或显示不正确，可能是参数问题，或者样式分隔符设置问题，可以通过根据[数据万象文档](https://cloud.tencent.com/document/product/460/6929)，进行排查。

> 图片上传后首次访问，可能由于 CDN 回源，或图片处理，速度不够理想，之后就没问题了。当然也可以手动去刷 CDN 缓存进行“预热”。
> 我这里只是用了数据万象中“图片格式转换”这一个功能，根据API在URL中附加参数，还可以实现图片缩放、裁剪、缩略图、文字和图像水印、获取EXIF信息、盲水印等等，很多高级的功能。

## 第二步，选择一个顺手的图片上传工具

macOS 上的 [iPic](https://apps.apple.com/cn/app/ipic-markdown-%E5%9B%BE%E5%BA%8A-%E6%96%87%E4%BB%B6%E4%B8%8A%E4%BC%A0%E5%B7%A5%E5%85%B7/id1101244278?mt=12) 产品理念、执行力，都非常令人佩服，难能可贵的还是由独立开发者完成的。之后涌现大量模仿者都或多或少受到了一定的启发。

因为需要考虑到 webp 并非所有浏览器都兼容，所以我采用的是在 Markdown 中插入 html 的方案来解决的。另外还可以通过 JS 来检测，但是需要修改 hexo 代码，担心版本升级的时候还要自己处理冲突，所以没有采用。

纯html代码解决兼容性的示例代码如下：

```html
<picture>
  <source srcset="https://img.blog.i1hao.com/IMG_0526.png?imageView2/format/webp" type="image/webp">
  <source srcset="https://img.blog.i1hao.com/IMG_0526.png?imageView2/format/png" type="image/png">
  <img src="IMG_0526.png">
</picture>
```

非常遗憾的是，我没有使用 iPic，而是[PicGo](https://github.com/Molunerfinn/PicGo)，因为通过它的“自定义链接格式”功能，可以生成模版代码，能提高一些效率。但是 PicGo 唯一令我不太满意的就是，我在配置腾讯云COS为图床时，遇到了点小问题，如果是新手可能会卡在这里一会。

配置参照下图：
<picture><source srcset="https://img.blog.i1hao.com/20190702233755.png?imageView2/format/webp" type="image/webp"><source srcset="https://img.blog.i1hao.com/20190702233755.png?imageView2/format/png" type="image/png"><img src="20190702233755"></picture>

> 需要注意的是，腾讯云COS分 V4 和 V5 两个版本，现在新用户应该是 V5 的了。如果不确定，或者想从 V4 升到 V5，请发工单。
> 存储空间名，并非 Bucket 的名字，而是 Bucket 名称 + APPID。从数据万象看到的只是 Bucket 名称，对象存储显示的才是正确的“Bucket 名称 + APPID”。这里可能责任也不在开发者，在我印象里腾讯云好像很久之前升级 SDK 修改过此处。总之知道怎么正确即可。

接下来，在 PicGo 的设置里，将自定义链接格式修改为:

```html
<picture><source srcset="$url?imageView2/format/webp" type="image/webp"><source srcset="$url?imageView2/format/png" type="image/png"><img src="$fileName"></picture>
```

使用的是`$url`和`$fileName`占位符，PicGo会自动帮你替换。

最后，在 PicGo 的上传区，将链接格式改为`Custom`即可：
<picture><source srcset="https://img.blog.i1hao.com/20190702234727.png?imageView2/format/webp" type="image/webp"><source srcset="https://img.blog.i1hao.com/20190702234727.png?imageView2/format/png" type="image/png"><img src="20190702234727"></picture>

一般来说，上传图片的流程可以是：

- 右键点击一个图片文件，选择复制（macOS上选择"拷贝"），点击 PicGo 在系统托盘中的图标，点击待上传。
- 打开 PicGo 的界面，或直接拖拽图片文件。
- 使用截图软件、图片编辑软件，将图片复制到剪贴板，点击 PicGo 在系统托盘中的图标，点击待上传。

以上三种方式任意一种，执行成功后，`Custom`那段代码都会自动复制到你的剪贴板，只要在需要的地方粘贴即可。

## 第三步，使用中意的 Markdown 编辑器，撰写文章

本文开始，我推荐过 Typora，自身能完美配合 iPic。但是由于它不能加载 html 中的图片，所以只能忍痛放弃。
<picture><source srcset="https://img.blog.i1hao.com/20190702235553.png?imageView2/format/webp" type="image/webp"><source srcset="https://img.blog.i1hao.com/20190702235553.png?imageView2/format/png" type="image/png"><img src="20190702235553"></picture>

vscode 对于程序员来说很容易上手，我只安装了`Markdown All in One`和`markdownlint`这两个插件，并且在（hexo 博客）工程根目录下创建了文件`.markdownlint.yml`，内容如下：
<picture><source srcset="https://img.blog.i1hao.com/20190703000042.png?imageView2/format/webp" type="image/webp"><source srcset="https://img.blog.i1hao.com/20190703000042.png?imageView2/format/png" type="image/png"><img src="20190703000042"></picture>

使用 vscode 编辑好博客文章后，执行发布，即可通过 Chrome、Safari等浏览器来验证此方案。

关于 Github Pages + Travis-CI 自动构建和发布的内容，可以参考我的 Github [仓库](https://github.com/edsion1107/blog.i1hao.com/tree/master)，不是本文主题就先不展开了。

## 总结

通过我目前这套方案，可以较好做到博客文章和图片资源的分离，并且在支持 webp 的浏览器上得到更好的体验；大量使用免费资源，降低部署和维护博客的成本；使用COS存储图片，还可以降低博客迁移成本。何乐而不为呢？

当然，缺点也不是没有，比如 vscode 如果作为 markdown 编辑器，距离专业的 Typora 还是有一定的差距，不支持 TOC 自动生成目录，不支持导出成PDF、图片等格式；访客达到一定量级后，可能会产生 CDN 费用。

如果大家还有其它方案，欢迎一起讨论，通过 Github 的 Issues 或[博客](https://blog.i1hao.com)上的其它方式都可以联系到我。
