---
title: 【SCF CLI实践】腾讯云serverless + 企业微信群机器人，轻松解决告警通知问题
date: 2019-06-28 23:25:45
tags:
    - serverless
    - 腾讯云SCF
    - SCF CLI
    - 企业微信
    - 群机器人
---

市面上有什么好用的从服务器推报警和日志到手机的工具？之前私下用的是[[Server酱]](http://sc.ftqq.com/3.version)的服务，非常方便。
但是考虑到安全原因，这个服务如果用在生产环境心里还是有点慌（虽然我相信[Server酱]是很有节操的）。

另外，出于想把工作和生活分开的私心（并不能），所以一直在探索使用企业微信来实现通知的方案（自建应用的话还要考虑通知的到达率，实在有心无力）。

之前已经基于企业微信的自建应用API，开发和部署了一个服务，技术方案是 docker + django + mysql + celery + redis。开发时要达到高可用，涉及 access_token 的刷新，还要考虑任务队列、分布式锁等等，真是折腾。

今天（2019-6-28）在更新企业微信时，发现增加了个群机器人的功能，赶紧查了一下文档发现挺符合需求的。群机器人的优点是，通知基于群组，有问题时直接可以在群内沟通，也不会存在重新拉群还需要介绍背景，人员发生变动时拉新人进群即可。

监控报警的脚本可以放在自己的服务器上，但是一旦遇到光纤被挖断了、停电这种外部因素，就需要一些外部手段来监控了。这里我推荐一下[腾讯云无服务器云函数（Serverless Cloud Function，SCF）](https://cloud.tencent.com/redirect.php?redirect=10633&cps_key=d61d10b4df29eee56b529f879e701bb2)，免费，如果用腾讯云的其它服务还有各种加成，现在还有【[活动链接](https://cloud.tencent.com/redirect.php?redirect=10633&cps_key=d61d10b4df29eee56b529f879e701bb2)】（写个简单的demo就行）送腾讯云的通用代金券、公仔等。

这里我就写个简单的天气小助手，来演示一下整个流程。

<!-- more -->
## 环境搭建

首先要先安装 SCF CLI

```bash
mkdir WeworkWeatherReport
cd WeworkWeatherReport
# 这里用的是pipenv安装的，如果觉得麻烦或者不熟悉此命令，也可以按照官方的演示直接用pip安装
# 即：用"pip install"替换文中"pipenv install"，"scf"替换文中"pipenv run scf"
$pipenv install --python 3.6 -i https://mirrors.cloud.tencent.com/pypi/simple scf
Creating a virtualenv for this project…
Pipfile: /Users/edsion/Documents/WeworkWeatherReport/Pipfile
Using /Users/edsion/.pyenv/versions/3.6.8/bin/python3 (3.6.8) to create virtualenv…
⠧ Creating virtual environment...Using base prefix '/Users/edsion/.pyenv/versions/3.6.8'
New python executable in /Users/edsion/.local/share/virtualenvs/WeworkWeatherReport-et9Xz9ZJ/bin/python3
Also creating executable in /Users/edsion/.local/share/virtualenvs/WeworkWeatherReport-et9Xz9ZJ/bin/python
Installing setuptools, pip, wheel...
done.
Running virtualenv with interpreter /Users/edsion/.pyenv/versions/3.6.8/bin/python3

✔ Successfully created virtual environment!
Virtualenv location: /Users/edsion/.local/share/virtualenvs/WeworkWeatherReport-et9Xz9ZJ
Creating a Pipfile for this project…
Installing scf…
Adding scf to Pipfile's [packages]…
✔ Installation Succeeded

$pipenv run scf --version   # 验证是否安装成功
SCF CLI, version 0.0.3
```

这里使用的是 python 3.6，尽量使用SCF支持的版本来开发和调试。使用腾讯云的镜像，可以加速下载，避免pypi被墙的问题。

通过SCF CLI可以简化流程，规避一些配置、部署等问题，尽快看到效果，所以强烈建议使用。

## 配置SCF CLI

这里需要 账号的 APPID，SecretId，SecretKey 以及产品期望所属的地域。

### APPID

通过[腾讯云的控制台](https://console.cloud.tencent.com/developer)，点击自己头像 - 账号信息查看。

### SecretId和SecretKey

基于安全考虑，这里需要使用“子账户”。打开[腾讯云控制台](https://console.cloud.tencent.com/cam) - 访问管理 - 用户 - 新建用户，随便起个名字，例如`scf-admin`，勾选“编程访问”后点击下一步。从策略列表中选取策略关联，搜索“scf”相关策略，根据需要勾选。用户创建完成后，回到用户列表，点击该用户选择“API密钥”，即可看到SecretId和SecretKey。

{% asset_img 006tNc79ly1g4h2imn34hj316o0prwh6.jpg %}

### 产品期望所属的地域

目前通过控制台可以看到地域有广州、上海、香港、北京、成都。如果有用到腾讯云的其它服务，例如云服务器CVM、对象存储COS，建议可用区保持一致，即使将来收费，按照良心云的套路同可用区内的流量应该是免费的。

> 这里文档有个小坑，并没有写明可用区的名称，所以参考了[COS文档中关于地域的说明](https://cloud.tencent.com/document/product/436/6224)。我是用的上海，所以地域是`ap-shanghai`。

### 配置CLI

根据以上参数，执行以下命令，完成 SCF CLI 的配置：

```bash
$pipenv run scf configure set --region ap-shanghai --appid 125xxxxx --secret-id AKIxxxxxxxxxx --secret-key uxxlxxxxxxxx
$pipenv run scf configure get    # 检查配置
API config:
appid = 125xxxxx
region = ap-shanghai
secret-id = ********************************M1c9
secret-key = ****************************Z4Ne
```

如果没有错误，不会显示任何提示，此时配置完成进入下一步。

## 初始化模板项目

通过 scf init 命令进行项目初始化操作。详细的参数可以[查看文档](https://cloud.tencent.com/document/product/583/33450)。

```bash
$pipenv run scf init --runtime python3.6 --name WeworkWeatherReport --output-dir .
[+] Initializing project...
Template: /Users/edsion/.local/share/virtualenvs/WeworkWeatherReport-et9Xz9ZJ/lib/python3.6/site-packages/tcfcli/cmds/init/templates/tcf-demo-python
Output-Dir: .
Project-Name: WeworkWeatherReport
Runtime: python3.6
[*] Project initialization is complete

$ tree
.
├── Pipfile
├── Pipfile.lock
└── WeworkWeatherReport
    ├── README.md
    ├── __pycache__
    │   └── index.cpython-36.pyc
    ├── index.py
    └── template.yaml

2 directories, 6 files
```

## 创建企业微信群机器人

> 经过咨询，目前群机器人还是处于灰度测试阶段，不是所有企业都已自动开通。联系你们公司的管理员，如果找不到 “企业微信后台 - > 应用与小程序 - > 自建 -> 群机器人”，只能耐心等待。

首先，确保你的企业微信已更新到最新（2.8.7）。然后拉一个群（至少3个人），在群里点右上角，添加成员的下方会多出来一个“群机器人”。按照提示填写一些基本信息即可完成创建（比较简单，看图吧）：

{% asset_img IMG_0526.png %}{% asset_img IMG_0527.png %}{% asset_img IMG_0528.png %}{% asset_img IMG_0529.png %}


最后，拿到`webhook`地址就算完成了。

> 注意：webhook地址里已经携带了鉴权参数，所以此URL **不应** 泄漏给任何人。
>
> 另外，目前测试如果群内有外部人员时，是无法使用群机器人的，不知道后面会不会放开。

用CURL测试一下：

```bash
$ curl 'https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=acd8cabe-xxx-xxx' -H 'Content-Type: application/json' -d '{"msgtype":"text","text":{"content":"hello world"}}'
{"errcode":0,"errmsg":"ok"}
```

{% asset_img 006tNc79ly1g4h45msshjj30n01ds75e.jpg %}

## 编写和调试代码，实现通过免费API获取天气和推送消息

免费天气API有很多，这里使用的是彩云天气的，可以根据经纬度获取实时天气，还有精确到分钟级的降雨预报。代码很简单，就不详细解释了。

```python
#!/usr/bin/env python3
# -*- coding:utf-8 _*-
"""
@author: edsion
@file: index.py
@time: 2019/06/28
@contact: edsion@21kunpeng.com

"""
import os
import json
import urllib3

X = os.getenv('X', '121.6544')  # 要获取天气信息的经度
Y = os.getenv('Y', '25.1552')   # 要获取天气信息的纬度

SKYCON = {
    'CLEAR_DAY': "晴（白天）",
    'CLEAR_NIGHT': "晴（夜间）",
    'PARTLY_CLOUDY_DAY': '多云（白天）',
    'PARTLY_CLOUDY_NIGHT': '多云（夜间）',
    'CLOUDY': '阴',
    'WIND': '大风',
    'HAZE': '雾霾',
    'RAIN': '雨',
    'SNOW': '雪',
    'Unknown': '其它'
}
COMFORT = {
    -1: '未知',
    0: '闷热',
    1: '酷热',
    2: '很热',
    3: '热',
    4: '温暖',
    5: '舒适',
    6: '凉爽',
    7: '冷',
    8: '很冷',
    9: '寒冷',
    10: '极冷',
    11: '刺骨的冷',
    12: '湿冷',
    13: '干冷',
}


def translate_aqi(n):
    if 0 < n <= 50:
        return f'<font color="info">{n}-优</font>'
    elif 50 < n <= 100:
        return f'<font color="info">{n}-良</font>'
    elif 100 < n <= 150:
        return f'<font color="warning">{n}-轻度污染</font>'
    elif 150 < n <= 200:
        return f'<font color="warning">{n}-中度污染</font>'
    elif 200 < n <= 300:
        return f'<font color="warning">{n}-重度污染</font>'
    elif n > 300:
        return f'<font color="warning">{n}-严重污染</font>'
    else:
        return f'<font color="comment">{n}-未知</font>'


def template_string(**kwargs):
    return """当前《公司》位置天气<font color="warning">{skycon}</font>
> 气温：{temperature:.2f}摄氏度(℃)
> 气压：{pres:.2f}千帕(kPa)
> 相对湿度：{humidity:.0f}%
> 风向：{wind_direction:.0f}(从北顺时针0~360°)
> 风速：{wind_speed}公里/小时(km/h)
> 能见度：{visibility}公里(km)
> pm2.5：{pm25}μg/m³
> AQI（国标）：{aqi}
> 最近的降水带：<font color="warning">{precipitation}</font>公里(km)
> 舒适度：{comfort}
""".format(**kwargs)


def main_handler(event, context):
    http = urllib3.PoolManager()
    r = http.request(
        method='GET',
        url=f'https://api.caiyunapp.com/v2/TAkhjf8d1nlSlspN/{X},{Y}/realtime.json?lang=zh_CN&unit=metric&tzshift=28800')
    print(f'API Response:{r.data}')
    res = json.loads(r.data)
    assert res.get('status') == 'ok'
    result = res.get('result')
    assert result
    assert result.get('status') == 'ok'
    content = template_string(
        skycon=SKYCON.get(result.get('skycon', 'Unknown')),
        temperature=result.get('temperature', 'Unknown'),
        pres=result.get('pres', 0) / 1000,
        humidity=result.get('humidity', 0) * 100,
        wind_direction=result.get('wind', {}).get('direction', 'Unknown'),
        wind_speed=result.get('wind', {}).get('speed', 'Unknown'),
        precipitation=result.get('precipitation', {}).get('nearest', {}).get('distance', 'Unknown'),
        visibility=result.get('visibility', 'Unknown'),
        pm25=result.get('pm25', 'Unknown'),
        aqi=translate_aqi(result.get('aqi', -1)),
        comfort=COMFORT.get(result.get('comfort', {}).get('index', -1)),
    )
    encoded_data = json.dumps({"msgtype": "markdown", "markdown": {'content': content}}).encode('utf-8')
    rr = http.request(method='POST', url=os.getenv('WEBHOOK_URL'), body=encoded_data,
                      headers={'Content-Type': 'application/json'})
    print(f'webhook response:{rr.data}')
    assert json.loads(rr.data).get('errcode') == 0
    return r.status


if __name__ == '__main__':
    main_handler(None, None)


```

要在本地运行此代码，配置环境变量：经纬度、webhook地址，即可——如此设计，不用改代码就可以轻易修改位置、群机器人。

本地运行结果如下：

{% asset_img 006tNc79ly1g4h7b54oh2j307f06wmxw.jpg %}

> 关于本地调试，因为场景过于简单，我是直接在pycharm中配置环境变量后运行的。如果需要对事件来源（event）等参数进行调试，可以使用 SCF CLI 的 `native invoke` 和`local invoke`，具体请查阅[文档](https://cloud.tencent.com/document/product/583/35402)。

## 函数打包和部署

scf cli 通过 `package` 子命令来完成函数打包。依据指定的函数模板配置文件打包后，生成部署使用的配置文件。

根据前面的步骤，项目文件夹内应该已经生成了模版配置文件`template.yaml`，根据实际情况修改环境变量（经纬度、webhook地址），并优化一下如描述、超时时间等参数：

```yaml
Resources:
  default:
    Type: TencentCloud::Serverless::Namespace
    WeworkWeatherReport:
      Type: TencentCloud::Serverless::Function
      Properties:
        CodeUri: ./WeworkWeatherReport/
        Description: 这是一个通过企业微信群机器人进行天气预报的小程序
        Environment:
          Variables:
            X: "121.6544"   # 要获取天气信息的经度
            Y: "25.1552"    # 要获取天气信息的纬度
            WEBHOOK_URL: https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=acd8cabe-xxxxx  # 企业微信群机器人的webhook地址
        Handler: index.main_handler
        MemorySize: 128
        Runtime: Python3.6
        Timeout: 10
Globals:
  Function:
    Timeout: 30

```

> 注意：我们这里的目录结构与官方文档中不太一致，所以我修改了`CodeUri`，否则之后部署的时候会出错。

执行命令，进行打包：

```bash
$ pipenv run scf package --template-file WeworkWeatherReport/template.yaml
Generate deploy file 'deploy.yaml' success
```

根据显示的信息可以注意到，新增了一个`deploy.yaml`的文件：

```yaml
Resources:
  default:
    Type: TencentCloud::Serverless::Namespace
    WeworkWeatherReport:
      Properties:
        CodeUri: ./
        Description: "\u8FD9\u662F\u4E00\u4E2A\u901A\u8FC7\u4F01\u4E1A\u5FAE\u4FE1\
          \u7FA4\u673A\u5668\u4EBA\u8FDB\u884C\u5929\u6C14\u9884\u62A5\u7684\u5C0F\
          \u7A0B\u5E8F"
        Environment:
          Variables:
            WEBHOOK_URL: https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=acd8cabe-xxxxx
            X: "121.6544"
            Y: "25.1552"
        Handler: index.main_handler
        LocalZipFile: /Users/edsion/Documents/WeworkWeatherReport/./.tcf_build/52a5e968-99ae-11e9-b0db-003ee1c86353.zip
        MemorySize: 128
        Runtime: Python3.6
        Timeout: 10
      Type: TencentCloud::Serverless::Function

```

再执行`deploy`子命令即可执行部署：

```bash
$ pipenv run scf deploy --template-file deploy.yaml
deploy default begin
Deploy function 'WeworkWeatherReport' success
deploy default end
```

如无异常，此时在[腾讯云的控制台](https://console.cloud.tencent.com/scf/list?rid=4&ns=default)中已经可以看到刚创建的`WeworkWeatherReport`“函数”：

{% asset_img 006tNc79ly1g4h8d82i7uj31d10aljss.jpg %}

如果代码需要更新，要重新执行打包-部署的流程，但是部署时需要添加`--forced`参数：

```bash
$ pipenv run scf package --template-file WeworkWeatherReport/template.yaml
Generate deploy file 'deploy.yaml' success
$ pipenv run scf deploy --forced --template-file deploy.yaml
deploy default begin
default WeworkWeatherReport already exists, update it now
Deploy function 'WeworkWeatherReport' success
deploy default end
```

## 添加触发方式

最后，在腾讯云的控制台中添加触发方式，测试通过后本次目标宣布正式达成。

{% asset_img 006tNc79ly1g4h8l0ivz3j30kp0lb3ze.jpg %}



## 几个小技巧

### 1. 只给自己发通知

先建群（除去自己额外再拉 2 个人进群，并且必须不是外部联系人）再把其他人踢掉就可以了。只要踢人之前不说话，对其他人来讲是无感知的。

### 2. 机器人共享

在群里把鼠标指向机器人，可以看到群聊机器人是可以发布到公司的（可能将来会有一个内部的 AppStore ？），还可以添加到其它群聊，这样可玩性还可以继续发掘一下。

> 测试过程中发现有的时候Windows和macOS的客户端可能不是很稳定？

## 总结

我这里只演示了最简单的、无第三方依赖的情况，实际使用还可以结合腾讯云的其它产品（例如API网关）来实现更复杂的需求。

通过这么一个简单的功能，可以说解决了一个困扰我多时的难题，关键它还免费！真心要为腾讯云和企业微信点个赞！！
