---
title: "macOS安装v2ray客户端"
date: 2024-07-27T07:45:32+08:00
tags:
  - it
  - macos
---

主要介绍如何安装和配置v2ray，以及思路。如果只想要客户端配置，请跳至末尾

## v2ray介绍和服务端安装

v2ray服务端安装之前写过一篇: [《VPN Setup》](../vpnsetup)。安装后记录关键配置
* **serverIp** (e.g. 20.163.126.10)
* **serverPort** (e.g. 35555)
* **uuid** (e.g. e2f30fgk-xxxx-xxxx-xxxx-xxxxxxxxxxxx)
* **alertId** (e.g. 42)

## 安装v2ray客户端

v2ray客户端有很多，V2rayU/V2rayX/V2box等。要么多年无人维护，要么安装繁琐。到底还是开源社区维护的[v2fly/v2ray-core](https://github.com/v2fly/v2ray-core)最靠谱

简单说，v2ray是一个支持vmess加密通讯服务。启动两个服务，一个在外网 (server)，一个在本地mac上 (client)。client和server通过vmess通讯。client向mac提供socket代理服务

![](/img/macos_v2ray_client/dataflow.png#center)

*注: 技术上说，图中client和server都是网络服务。mac将client的socket服务配置为系统代理才能起效*

### brew install v2ray

`brew`是macOS的第三方包管理工具，可以用国内脚本安装 (github链接不稳定)。建议将镜像源设置成阿里云。

```bash
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"
```

然后安装v2ray
```bash
brew install v2ray
```

## 全局代理配置

从最简单开始。这是所有流量都走代理的配置，保存到`$HOME/config.json`

`inbounds`表示入口 (监听)用`socks`协议，监听`127.0.0.1:1080`。不需要认证

`outbounds`表示出口用`vmess`协议，`vnext`下是链接服务端的地址和认证

意味着client会将发往`127.0.0.1:1080`的`tcp,udp`流量通过`vmess`转发到服务端

```json
{
    "inbounds": [{
        "port": 1080, // v2ray客户端监听端口
        "listen": "127.0.0.1", // 只允许本电脑链接
        "protocol": "socks",
        "settings": {
            "auth": "noauth",
            "udp": true
        }
    }],
    "outbounds": [{
        "protocol": "vmess",
        "settings": {
            "vnext": [{
                "address": "serverIp", // v2ray服务器IP
                "port": serverPort, // 服务端口
                "users": [{
                    "id": "uuid",
                    "alterId": alertId
                }]
            }]
        }
    }]
}
```

启动client
```bash
# 可以先检查配置格式
# v2ray test
v2ray run
```

## macOS启用代理

我的电脑用Wi-Fi，点开`详细信息...`，把client信息配入`SOCKS代理`
![](/img/macos_v2ray_client/macos_socks_proxy_config.png#center)

如果嫌系统配置繁琐，可以用脚本
```bash
#!/bin/sh

ENABLED=$(networksetup -getsocksfirewallproxy Wi-Fi | head -1 | awk '{print $2}')

if [[ "$ENABLED" == "No" ]]; then
    networksetup -setsocksfirewallproxystate Wi-Fi on
    pgrep -qf v2ray || v2ray run &
else
    networksetup -setsocksfirewallproxystate Wi-Fi off
    kill $(pgrep -f v2ray)
fi
```

如是，全局tcp/udp流量启用。所谓全局指浏览器、应用和类似iCloud的后台服务都会走代理。

全局代理不够经济实惠。试想，iCloud后台同步数据流量大，走代理即占带宽又降速——国内本来能直连的网站从国外绕了一圈，舍近取远。有些国内网站走代理甚至反而会被墙

## 配置路由和DNS规则

目标是:
1. 只有常用的国外网站走代理
2. 剩余下都直连，不走代理

首先，`outbounds`应该配置两个出口: `direct`直连，`proxy`代理。`direct`在前，作默认路由。意味着数据流量默认直连，除非匹配`proxy`对应的规则

之后配置重点: `routing`，路由规则。规则从上到下 (数组从前到后)，先匹配先奏效，类似`iptables`。`"domain": ["geosite:geolocation-!cn"]`包含了常用国外网站域名。去往这些域名的流量会走代理

但实践发现，iCloud的cdn有时会把流量“错误”地导向国外。鉴于苹果的国内服务稳定，要前插一条规则确保苹果流量都直连

最后是一条通用规则，让没匹配前置规则所有`tcp,udp`流量都直连

`dns`规则是个补充，作用是默认使用电信DNS (114.114.114.114)，只有常用的国外网站用Cloudflare DNS (1.1.1.1)

```json
{
    "log": {
        "loglevel": "warning"
    },
    "inbounds": [
        {
            "port": 1080,
            "listen": "127.0.0.1",
            "protocol": "socks",
            "settings": {
                "auth": "noauth",
                "udp": true
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "freedom",
            "tag": "direct"
        },
        {
            "protocol": "vmess",
            "settings": {
                "vnext": [
                    {
                        "address": "serverIp",
                        "port": serverPort,
                        "users": [
                            {
                                "id": "uuid",
                                "alterId": alertId
                            }
                        ]
                    }
                ]
            },
            "tag": "proxy"
        }
    ],
    "routing": {
        "rules": [
            {
                "type": "field",
                "domain": [
                    "geosite:apple",
                    "geosite:cn"
                ],
                "outboundTag": "direct"
            },
            {
                "type": "field",
                "domain": [
                    "geosite:geolocation-!cn"
                ],
                "outboundTag": "proxy"
            },
            {
                "type": "field",
                "network": "tcp,udp",
                "outboundTag": "direct"
            }
        ]
    },
    "dns": {
        "servers": [
            "114.114.114.114",
            {
                "address": "1.1.1.1",
                "port": 53,
                "domains": [
                    "geosite:geolocation-!cn"
                ]
            }
        ]
    }
}
```

最后，注意观察v2ray的console日志，看看流量走向是否符合预期。例如下面日志显示google和github流量走代理，iCloud则没走
```
2024/07/27 11:19:55 tcp:127.0.0.1:58714 accepted tcp:www.google.com:443 [proxy]
2024/07/27 11:20:21 tcp:127.0.0.1:58801 accepted tcp:raw.githubusercontent.com:443 [proxy]
2024/07/27 11:20:21 tcp:127.0.0.1:58803 accepted tcp:edge-062.xasha1.icloud-content.com.cn:443 [direct]
```

*注: `geosite:microsoft`包含github域名。为了加速不配在前置`direct`规则里*

参考来源:
* [Routing路由](https://v2raydocs.web.illinois.edu/config/routing.html#routingobject)
* [路由规则设定方法](https://toutyrater.github.io/routing/configurate_rules.html)
* [关于路由规则的注意事项](https://toutyrater.github.io/basic/routing/notice.html)
