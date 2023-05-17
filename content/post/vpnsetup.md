---
title: "VPN Setup"
date: 2022-10-18T09:16:11+08:00
tags:
  - it
---

## V2Ray

关于V2Ray介绍，请参考 [V2Ray简介和搭建教程](https://itlanyan.com/v2ray-tutorial/)

文章写得很好，我仅做点补充: 文中的安装脚本 (goV2.sh) 在2022年11月7日报错。我仅针对报错修改了一下脚本，可自取

```bash
wget https://raw.githubusercontent.com/kyzy540/blogdraft/main/static/scripts/goV2.sh
chmod +x goV2.sh
sudo ./goV2.sh --version v5.1.0
```

改动点如下
* 删除`v2ctl` 相关代码，`v2ctl` 在v2ray v5.1.0 中被移除
* 修改`v2ray.service`启动命令，增加了`run`子命令和 `v2ray.vmess.aead.forced=false` 环境变量

该脚本集成了配置文件生成，位置在`/etc/v2ray/config.json`。其中的uuid和端口均自动生成。安装完成后参考配置放行端口，启用服务

```bash
# ubuntu 20.02
port=$(grep port /etc/v2ray/config.json | awk '{print $2}' | cut -d, -f1)
sudo ufw allow ${port}/tcp
#启用 v2ray服务
sudo systemctl enable v2ray
sudo systemctl start v2ray
```

注意云实例的防火墙也要放行配置中的端口(TCP协议)

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

### Xray (未实践)

[Xray教程](https://itlanyan.com/xray-tutorial/) 是V2Ray进阶篇，搭建更复杂，稳定性也更高。其中有一步骤制作TLS证书，可以按文中所述购买域名，也可以用自签名证书，但无论如何都有点麻烦。我的V2Ray还算稳定，所以这里仅做记录，以备日后不时之需

我不推荐购买域名。因国内域名要求备案，购买国外域名基本是唯一出路，支付不便。另外 [Let's Encrypt](https://letsencrypt.org/)签发的证书只有3个月有效期，必须定期续期，有额外管理成本

自签名证书有技术门槛，但一次投入，长期收益，因此是个更合理的选择。参考文档:

[局域网内搭建浏览器可信任的SSL证书](https://www.tangyuecan.com/2021/12/17/%E5%B1%80%E5%9F%9F%E7%BD%91%E5%86%85%E6%90%AD%E5%BB%BA%E6%B5%8F%E8%A7%88%E5%99%A8%E5%8F%AF%E4%BF%A1%E4%BB%BB%E7%9A%84ssl%E8%AF%81%E4%B9%A6/)

[如何创建自签名SSL证书支持私有域名的HTTPS服务](https://blog.heylinux.com/2021/11/%E5%A6%82%E4%BD%95%E5%88%9B%E5%BB%BA%E8%87%AA%E7%AD%BE%E5%90%8Dssl%E8%AF%81%E4%B9%A6%E6%94%AF%E6%8C%81%E7%A7%81%E6%9C%89%E5%9F%9F%E5%90%8D%E7%9A%84https%E6%9C%8D%E5%8A%A1/)

[Certbot DNS problem - not using /etc/hosts](https://stackoverflow.com/questions/69566131/certbot-dns-problem-not-using-etc-hosts)

## Trojan (未实践)

Trojan 和 Xray 类似，带伪装和加密，也需要TLS证书。但 [trojan-go](https://github.com/p4gefau1t/trojan-go) 代码库已经1年多没更新，最新版停留在 v0.10.6。相比 Xray 项目显得缺少活力。这里仅记两篇部署教程，供参考

[Install Trojan-GFW Server on Ubuntu Linux](https://sedap.github.io/install-trojan-gfw-on-ubuntu.html)

[How To Easily Install Trojan GFW on Ubuntu - A Step by Step Tutorial](https://privacymelon.com/trojan-gfw-cdn-tutorial/)

## L2TP (不推荐)

L2TP协议是通用VPN协议，包括ios/android/windows/macos等系统原生支持。优点是易搭建易配置，缺点是容易被屏蔽……多容易呢？活不过3天

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
