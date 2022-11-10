---
title: "VPN Setup"
date: 2022-10-18T09:16:11+08:00
tags:
  - it
---

## L2TP

L2TP协议是通用VPN协议，包括ios/android/windows/macos等系统原生支持。优点是易搭建易配置，缺点是容易被屏蔽

一键部署脚本和文档: https://github.com/hwdsl2/setup-ipsec-vpn  。核心代码就一段

```bash
# All values MUST be placed inside 'single quotes'
# DO NOT use these special characters within values: \ " '
wget https://get.vpnsetup.net -O vpn.sh
sudo VPN_IPSEC_PSK='your_ipsec_pre_shared_key' \
VPN_USER='your_vpn_username' \
VPN_PASSWORD='your_vpn_password' \
sh vpn.sh
```

os选ubuntu。不选centos是因其没带 `wget` ，懒得装。防火墙放行 UDP 500 和 4500 端口。ios的VPN配置类型选 L2TP

建议防火墙屏蔽包括 22(ssh) 在内所有其他端口，避免不必要的安全问题

## V2Ray

关于V2Ray，读 [V2Ray简介和搭建教程](https://itlanyan.com/v2ray-tutorial/)

文章写得很好，我仅补充一点: 文中的安装脚本 (goV2.sh) 在2022年11月7日报错。我仅针对报错修改了一下脚本，可自取

```bash
bash <(curl -sL https://raw.githubusercontent.com/kyzy540/blogdraft/main/static/scripts/goV2.sh --version v5.1.0)
```

改动点如下
* 删除`v2ctl` 相关代码，`v2ctl` 在v2ray v5.1.0 中被移除
* 修改`v2ray.service`启动命令，增加了`run`子命令和 `v2ray.vmess.aead.forced=false` 环境变量

### 诊断V2Ray代理

在未配置 `v2ray.vmess.aead.forced=false` 环境变量时V2Ray代理虽然能链接，但Safari打不开任何网页，报错类似

> safari cannot open the page cannot establish a secure connection

诊断的关键是在 `/etc/v2ray/config.json` 加入日志配置，并重启v2ray服务

```json
{
  "log": {
    "access": "/var/www/v2ray/access.log",
    "error": "/var/www/v2ray/error.log",
    "loglevel":"info"
  },
  "inbounds": ...
}
```

重启后再测试，看到日志有如下
> VMessAEAD is enforced and a non VMessAEAD connection is received. You can still disable this security feature with environment variable v2ray.vmess.aead.forced = false

在启动命令加入环境变量后问题解决

## Trojan

trojan协议和上文教程中提到的 [V2Ray高级技巧：流量伪装](https://itlanyan.com/v2ray-traffic-mask/) 类似。稳定性更高，但成本也高。用户需额外购买域名，并为之签发HTTPS证书。又因国内域名要求备案，购买国外域名基本是唯一出路，因而还涉及支付问题，就不展开讨论。仅列两篇部署教程，供参考

[Install Trojan-GFW Server on Ubuntu Linux](https://sedap.github.io/install-trojan-gfw-on-ubuntu.html)

[How To Easily Install Trojan GFW on Ubuntu - A Step by Step Tutorial](https://privacymelon.com/trojan-gfw-cdn-tutorial/)
