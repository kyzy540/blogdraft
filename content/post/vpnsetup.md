---
title: "VPN Setup"
date: 2022-10-18T09:16:11+08:00
tags:
  - it
---

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

另外建议防火墙屏蔽包括 SSH 在内所有其他链接，避免不必要的安全问题
