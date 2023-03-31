---
title: "插件SwitchyOmega"
date: 2023-03-30T20:26:29+08:00
draft: true
tags:
  - it
---

## 背景

SwitchyOmega是著名Chrome代理插件，[源代码](https://github.com/FelisCatus/SwitchyOmega)开源在GitHub，最近release是2018年8月

它很健全，但算不上完美。截止写本文的2023年3月，GitHub仍有新issue提交，但开发者已经了无音讯

所以当我想对它做改动，只能直接修改源代码，自己制作crx了

我的需求是: 任意网页上，如果出现了 `switchy://+profile_name:10.1.2.3:8080` ，SwitchyOmega可以一键修改该 `profile_name` 对应情景模式中代理服务器的地址和端口。为达到目标，我把工作拆分成了几步

1. 学习Chrome插件基本原理
2. 了解SwitchyOmega如何实现代理功能
3. 确定需求是否可能实现，如果是，找到切入点
4. 学习本地开发和调试
5. 真正着手实现功能

我不想发布这个模改的插件，毕竟它仅对我个人有用。所以到这就差不多了

对了，我没前端经验，也没开发过Chrome插件

## 基础知识

* [Chrome Extension development basics](https://developer.chrome.com/docs/extensions/mv3/getstarted/development-basics/)
* [AngularJS](https://www.runoob.com/angularjs/angularjs-scopes.html)
* [CoffeeScript](https://coffeescript.org/)
* [Building the project](https://github.com/FelisCatus/SwitchyOmega/blob/master/README.md#building-the-project)

在Chrome插件开发基础中主要了解4个点
1. 插件的主要构成。包含前端、后端、manifest等
2. 插件前后端通信机制
3. Chrome代理相关API
4. SwitchyOmega如何基于配置调用Chrome API实现代理

SwitchyOmega代码 76.7% 是 CoffeeScript，用到了AngularJS框架。它依赖 NodeJS/npm/Grunt/bower 构建 (build)。nodejs相关教程比较多，不再赘述

SwitchyOmega构建会失败，其依赖的一个第三方库下架并停止维护了，下文会解释如何解决

## 配置保存流程

![](/img/extend_switchyomega/switchy_omega_save_profile.png)

## 本地开发和调试

### 构建

开发的第一步，是走通原有的构建流程，在Chrome里试用正常。如果参考构建文档，会在 `npm run deps` 阶段遇到如下报错

> bower angular-spectrum-colorpicker#~1.3.5      ECMDERR Failed to execute "git ls-remote --tags --heads https://github.com/Jimdo/angular-spectrum-colorpicker.git", exit code of #128 remote: Repository not found. fatal: Authentication failed for 'https://github.com/Jimdo/angular-spectrum-colorpicker.git/'

报错原因是 [angular-spectrum-colorpicker](https://www.npmjs.com/package/angular-spectrum-colorpicker) GitHub 已经被删除了，bower校验失败。我试过用其他fork仓库替代，但后续还会遇到其他麻烦事儿。最简单的办法是拿现成的js库塞入代码库，删除bower依赖配置

先从Chrome应用商店安装发布版SwitchyOmega，从中提取 `angular-spectrum-colorpicker.min.js`，保存到 `omega-web/lib/angular-spectrum-colorpicker/angular-spectrum-colorpicker.min.js`

修改 `https://github.com/FelisCatus/SwitchyOmega/blob/master/omega-web/bower.json`，把其中 `angular-spectrum-colorpicker`相关的2个健值对删除

这样构建过程就能跳过被废弃的依赖库，扩展直接使用代码库中的 `angular-spectrum-colorpicker.min.js`

我在开发过程中让模改插件和发布版共存，这样方便对照。为此，需修改 [manifest.json](https://github.com/FelisCatus/SwitchyOmega/blob/master/omega-target-chromium-extension/overlay/manifest.json#L6) 中`key`。它是一个public key，是插件的唯一标识，如不修改，模改插件会覆盖发布版。对于不发布的情况可以自己生成
```bash
openssl genrsa 2048 | openssl pkcs8 -topk8 -nocrypt -out key.pem
openssl rsa -in key.pem -pubout -outform DER | openssl base64 -A
```

我的操作系统语言是中文，也要[omega-web.po](https://github.com/FelisCatus/SwitchyOmega/blob/master/omega-locales/zh_CN/LC_MESSAGES/omega-web.po)，给模改插件换个名字，方便识别

顺利的话，完成以上修改后插件就能构建并在Chrome中工作了。构建完成后的unpacked插件目录在`omega-target-chromium-extension/build` ，而非文档中提到的 `omega-chromium-extension/build/`

### TODO: 代码
