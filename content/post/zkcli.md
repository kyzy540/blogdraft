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

一个轻量zookeeper cli客户端工具。基于`let-us-go/zkcli`改进。原版地址 (内附安装教程): https://github.com/let-us-go/zkcli

原版有个问题: `deleteall`子命令不能递归删除zookeeper node。原因是 [代码](https://github.com/let-us-go/zkcli/blob/master/core/cmd.go#L160) 没实现递归删除

鉴于仓库已超过1年不更新 (最后一次commit在2022-08-16)，我只好自己fork代码，增加了`deleteall`的递归逻辑。安装命令

```bash
go install github.com/kyzy540/zkcli
```

P.S. 这是本人第一次写`go`代码，如有问题还请见谅🙏
