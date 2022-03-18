---
title: "Android http-server"
date: 2022-03-17T18:36:21+08:00
tags:
  - android
keywords:
  - android
---

Android可以玩花活儿，这是我一直很喜欢它的原因。最近有个想法: 能不能将手机上的mp4串流到其他设备上播放？现在的电影都很大，动辄3、5G，拷贝很麻烦。而且有些设备不一定适合拷贝，要是能直接像流媒体一样直接播放手机里的mp4那该多爽

办法其实不复杂，[http-server](https://www.npmjs.com/package/http-server) 就能胜任这项工作。我要做的就是把它在手机上跑起来，步骤如下

第一步，安装 [Termux](https://f-droid.org/packages/com.termux/)。进入终端，启用并授权访问存储设备
```bash
termux-setup-storage
```

第二步，更新pkg源，安装nodejs和http-server。nodejs-lts是LTS版本，要装current版本可以`pkg install nodejs`
```bash
pkg update
pkg install nodejs-lts
npm install -g http-server
```

如果嫌默认源访问速度慢，可以考虑换用`termux-change-repo`切换到清华源。[参考链接](https://wiki.termux.com/wiki/Package_Management)

第三步，启动http-server。这里`storage/dcim/Camera`是服务对外开放的路径。如果写`storage/shared`可以开放整个手机存储
```bash
http-server storage/dcim/Camera
```

可以看到服务启动，监听`http://30.43.99.138:8080`
![](/img/android_httpserver/android_httpserver.png)

同局域网内的其他设备就可以直接访问这个地址浏览文件了
![](/img/android_httpserver/android_httpserver1.png)