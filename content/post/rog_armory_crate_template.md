---
title: "删除ROG奥创中心手柄映射配置"
date: 2025-11-01T21:44:47+08:00
tags:
  - it
keywords:
  - rog
  - armory crate
  - 奥创中心
---

ROG的奥创中心可以为每个游戏设独立的键位映射，但有个缺陷: 每个游戏的映射都会被保存下来，罗列在映射模版列表里。该如何删除自定义映射模版?

先找到模版文件目录
```
C:\Users\MyName\AppData\Local\Packages\B9ECED6F.ArmouryCrateSE_qmba6cd70vzyy\LocalState\GamepadCustomize
```

其中`MyName`是本地用户名。`B9ECED6F.ArmouryCrateSE_qmba6cd70vzyy`前后两串码我不确定是否随机，反正中间`ArmouryCrateSE`是一定的

目录的`Profiles`和`Templates`里有若干`<uuid>.json`文件，里面包含了模版名。删除json文件即可

**注意**，模版文件一式两份，两个目录下都要删除