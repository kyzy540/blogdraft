---
title: "zkcli"
date: 2023-09-16T21:06:11+08:00
tags:
  - it
keywords:
  - go
  - zookeeper
---

*2023-09-16*

一个zookeeper cli客户端工具。基于`let-us-go/zkcli`改进。原版地址 (内附安装教程): https://github.com/let-us-go/zkcli

原版有个问题: `deleteall`子命令不能递归删除zookeeper node。原因是 [代码](https://github.com/let-us-go/zkcli/blob/master/core/cmd.go#L160) 没实现递归删除

鉴于仓库已超过1年不更新 (最后一次commit在2022-08-16)，我只好自己fork代码，增加了`deleteall`的递归逻辑。安装命令

```bash
go install github.com/kyzy540/zkcli
```

相较于zookeeper自带的cli工具 `zkCli.sh`，这个go版本有3点优势
- 交互模式支持 `tab` 自动提示
- 支持 batch 模式执行命令
- 方便安装配置，不依赖java

`zkcli`默认会读配置文件`$HOME/.config/zkcli.conf`，可以避免每条命了都要输入冗长参数。配置文件参考 [zkcli.conf](https://github.com/let-us-go/zkcli/blob/master/zkcli.conf)。个人常用配置如下
```
s 127.0.0.1:2181
```

P.S. 这是本人第一次写`go`代码，如有问题还请见谅🙏
