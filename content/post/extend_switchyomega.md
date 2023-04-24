---
title: "扩展插件SwitchyOmega"
date: 2023-03-30T20:26:29+08:00
tags:
  - it
---

## 背景

SwitchyOmega是著名Chrome代理插件，[源代码](https://github.com/FelisCatus/SwitchyOmega)开源在GitHub，最近release是2018年8月

它很健全，但算不上完美。截止写本文的2023年3月，GitHub仍有新issue提交，但开发者已经了无音讯

所以当我想对它做改动，只能直接修改源代码，自己制作crx了

需求是: 任意网页上，如果出现了如下span，SwitchyOmega可以一键载入代理服务器的地址和端口

```html
<span id="proxy" data-profileName='demo' data-patterns='["192.168.*"]'>
  HTTP://172.168.1.1:8080
</span>
```

工作可以拆分成几步

1. 学习Chrome插件基本原理
2. 了解SwitchyOmega如何实现代理功能
3. 确认需求可否实现
4. 学习本地开发和调试
5. 着手实现

我不想发布这个模改的插件，毕竟它仅对个人有用。所以到这就差不多了

对了，我没前端经验，也没开发过Chrome插件

## 基础知识

* [Chrome Extension development basics](https://developer.chrome.com/docs/extensions/mv3/getstarted/development-basics/)
* [chrome.proxy](https://developer.chrome.com/docs/extensions/reference/proxy/)
* [AngularJS](https://www.runoob.com/angularjs/angularjs-scopes.html)
* [CoffeeScript](https://coffeescript.org/)
* [Building the project](https://github.com/FelisCatus/SwitchyOmega/blob/master/README.md#building-the-project)

在Chrome插件开发基础中主要了解4个点
1. 插件的主要构成。包含前端、后端、manifest、options
2. 插件前后端通信机制
3. Chrome代理相关API
4. SwitchyOmega如何基于配置调用Chrome API实现代理

SwitchyOmega代码 76.7% 是 CoffeeScript，用到了AngularJS框架。依赖 NodeJS/npm/Grunt/bower 构建 (build)

## SwitchyOmega配置代理流程

下图展示了在SwitchyOmega 选项页面里创建一个 "情景模式" (代理配置) 是如何生效的

![](/img/extend_switchyomega/switchy_omega_save_profile.png)

首先，SwitchyOmega本身没实现代理功能。它实现了一套界面，让用户可以方便使用Chrome代理API。Chrome内部完成了具体的代理实现

其次，SwitchyOmega在后台保存了一份核心数据结构`options`，其会被转译并导入`chrome.proxy.settings`。SwitchyOmega的前后端设计都围绕着维护`options`数据结构设计。要实现点击span加载配置，重点就在于搞明白如何将页面数据导入`options`

## 本地开发和调试

### 构建

开发的第一步，是走通原有的构建流程，在Chrome里试用正常。如果参考构建文档，会在 `npm run deps` 阶段遇到如下报错

> bower angular-spectrum-colorpicker#~1.3.5      ECMDERR Failed to execute "git ls-remote --tags --heads https://github.com/Jimdo/angular-spectrum-colorpicker.git", exit code of #128 remote: Repository not found. fatal: Authentication failed for 'https://github.com/Jimdo/angular-spectrum-colorpicker.git/'

报错原因是 [angular-spectrum-colorpicker](https://www.npmjs.com/package/angular-spectrum-colorpicker) GitHub 已经被删除了，bower校验失败。我试过用其他fork仓库替代，但后续还会遇到其他麻烦事儿。最简单的办法是拿现成的js库塞入代码库，删除bower依赖配置

先从Chrome应用商店安装发布版SwitchyOmega，从中提取 `angular-spectrum-colorpicker.min.js`，保存到 `omega-web/lib/angular-spectrum-colorpicker/angular-spectrum-colorpicker.min.js`

![](/img/extend_switchyomega/angular-spectrum-colorpicker.png#center)

修改 `https://github.com/FelisCatus/SwitchyOmega/blob/master/omega-web/bower.json`，把其中 `angular-spectrum-colorpicker`相关的2个健值对删除

这样构建过程就能跳过被废弃的依赖库，直接使用代码库中的 `angular-spectrum-colorpicker.min.js`

开发过程中为了让模改插件和发布版共存，方便对照，需修改 [manifest.json](https://github.com/FelisCatus/SwitchyOmega/blob/master/omega-target-chromium-extension/overlay/manifest.json#L6) 中`key`。它是一个public key，是插件的唯一标识，如不修改，模改插件会覆盖发布版。对于不发布的情况可以自己生成
```bash
openssl genrsa 2048 | openssl pkcs8 -topk8 -nocrypt -out key.pem
openssl rsa -in key.pem -pubout -outform DER | openssl base64 -A
```

对于操作系统语言是中文的，建议修改[omega-web.po](https://github.com/FelisCatus/SwitchyOmega/blob/master/omega-locales/zh_CN/LC_MESSAGES/omega-web.po)，给模改插件换个名字，方便识别

顺利的话，完成以上修改后插件就能构建并在Chrome中工作了。构建完成后的unpacked插件目录在`omega-target-chromium-extension/build` ，而非文档中提到的 `omega-chromium-extension/build/`

### 设计

回到目标，要想让网页里的代理配置保存到插件里，得用到 [Content scripts](https://developer.chrome.com/docs/extensions/mv3/content_scripts/)。其既可以往网页注入javascript代码，又能调用chrome.runtime访问插件内部数据，正好切合需求

![](https://waitingphoenix.com/content/images/2017/04/Blank-Flowchart---Page-2.png)

![](/img/extend_switchyomega/content_script_sandbox.png)

### 代码

代码包含`content-script.coffee` 、 `background.coffee` 和`manifest.json`。主要逻辑在`content-script.coffee`里

![](/img/extend_switchyomega/content_script_workflow.png)

1. 脚本向页面中的`id=proxy`元素注册click event listner，以触发添加配置代码
2. click发生时解析页面内proxy配置
3. 调用chrome.runtime接口将options patch发送到后端

代码如下
```coffee
console.log("<----- Content script started ----->")

autoSwitchKey = "+auto switch"
HostWildcardCondition = "HostWildcardCondition"
profileColors = [
  "#9ce", "#9d9", "#fa8", "#fe9", "#d497ee", "#47b", "#5b5", "#d63", "#ca0"
]

class Log
  log: (content) ->
    console.log(content)
    @_logBackground("log.log", "content-script: " + content)

  error: (content) ->
    console.error(content)
    @_logBackground("log.error", "content-script ERROR: " + content)

  _logBackground: (method, content) ->
    chrome.runtime.sendMessage({
      method: method
      args: [content]
      noReply: true
    })

logger = new Log

isObjectEmpty = (obj) ->
  for own key, val of obj
    return false
  return true

nameToKey = (name) -> "+#{name}"

generateRevision = -> (new Date()).getTime().toString(16)

diffProfile = (profile, scheme, host, port) ->
  pf = profile.fallbackProxy
  if pf
    fallbackProxy = {}
    if pf.scheme != scheme
      fallbackProxy.scheme = [pf.scheme, scheme]
    if pf.host != host
      fallbackProxy.host = [pf.host, host]
    if pf.port != port
      fallbackProxy.port = [pf.port, port]

    if isObjectEmpty(fallbackProxy)
      return {}
  else
    # 直接连接
    fallbackProxy = [{
      "scheme": scheme,
      "port": port,
      "host": host
    }]

  patch = {}
  patch[nameToKey(profile.name)] = {
    "fallbackProxy": fallbackProxy
    "revision": ["oldRevision", generateRevision()]
  }
  return patch

newProfilePatch = (name, scheme, host, port) ->
  choice = Math.floor(Math.random() * profileColors.length)
  patch = {}
  patch[nameToKey(name)] = [{
    "profileType": "FixedProfile",
    "name": name,
    "bypassList": [
      {
        "conditionType": "BypassCondition",
        "pattern": "127.0.0.1"
      },
      {
        "conditionType": "BypassCondition",
        "pattern": "[::1]"
      },
      {
        "conditionType": "BypassCondition",
        "pattern": "localhost"
      }
    ],
    "color": profileColors[choice],
    "revision": generateRevision(),
    "fallbackProxy": {
      "scheme": scheme, # http
      "port": port, # int
      "host": host
    }
  }]
  return patch

patchAutoSwitch = (switchProfile, profileName, patterns) ->
  rules = {}
  lastIndex = switchProfile.rules.length

  existsPatterns = (
    rule.condition.pattern for rule in switchProfile.rules when \
    rule.profileName == profileName and
    rule.condition.conditionType == HostWildcardCondition
  )

  for pattern in patterns when pattern not in existsPatterns
    rules[lastIndex] = [{
      "condition": {
        conditionType: HostWildcardCondition
        pattern: pattern
      }
      "profileName": profileName
    }]
    lastIndex++

  if isObjectEmpty(rules)
    return {}
  else
    # 增/删/改 auto switch都要加 _t: "a"
    rules["_t"] = "a"
    switchPatch = {
      "rules": rules
      "revision": ["oldRevision", generateRevision()]
    }
    return switchPatch

showAlert = (message, error) ->
    if error
      bgColor = "rgba(231, 76, 60, 0.8)"; # 半透明红色
    else
      bgColor =  "rgba(39, 174, 96, 0.8)" # 绿色半透明
    div = document.createElement("div")
    div.style.position = "fixed"
    div.style.top = "50%"
    div.style.left = "50%"
    div.style.transform = "translate(-50%, -50%)"
    div.style.padding = "20px"
    div.style.backgroundColor = bgColor
    div.style.color = "#fff"
    div.style.textAlign = "center"
    div.innerHTML = message
    parentNode = document.getElementsByClassName("goback")[0].parentElement
    parentNode.appendChild(div)
    setTimeout (-> parentNode.removeChild(div)), 3000


proxyUrlElement = document.getElementById("proxy")

# click发起代理配置
proxyUrlElement?.addEventListener "click", (event) ->
  proxyUrlElement = event?.target || event.srcElement
  if not proxyUrlElement?
    logger.error("fail to get element in click event")
    logger.error(JSON.stringify(event))
    return

  profileName = proxyUrlElement.dataset.profileName?
  if no profileName
    logger.log("no profileName")
    return
  # 从 data-patterns读auto switch规则
  patterns = JSON.parse(proxyUrlElement.dataset.patterns)

  # 读现有配置
  chrome.runtime.sendMessage({
    method: "getAll"
    args: []
  }, (response) ->
    try
      options = response.result
      logger.log("getAll from background: #{options}")

      proxyUrl = event.target.textContent
      logger.log("proxyUrl: #{proxyUrl} from #{document.URL}")
      [protocal, rawIp, proxyPort] = proxyUrl.split(":")

      #TODO 检查合法性
      scheme = protocal.toLowerCase()
      ip = rawIp.replace("//", "").trim()
      port = parseInt(proxyPort)

      # 增/改profile patch
      profileKey = nameToKey(profileName)
      if profileKey of options
        logger.log("existing profile #{profileName}")
        patch = diffProfile(options[profileKey], scheme, ip, port)
        if not patch?
          logger.log("#{profileName} does not update")
      else
        logger.log("create new profile #{profileName}")
        patch = newProfilePatch(profileName, scheme, ip, port)

      if patterns.length > 0
        switchPatch = patchAutoSwitch(
          options[autoSwitchKey],
          profileName,
          patterns
        )
        if isObjectEmpty(switchPatch)
          logger.log("#{patterns.length} switch patterns already exists")
        else
          logger.log("switchPatch: #{JSON.stringify(switchPatch)}")
          patch[autoSwitchKey] = switchPatch
      else
        logger.log("empty patterns for #{profileName}")

      if not isObjectEmpty(patch)
        chrome.runtime.sendMessage({
          method: "patch"
          args: [patch]
        }, (response) ->
          if chrome.runtime.lastError? or response?.error
            logger.error("unknown patch error: " + JSON.stringify(response))
            showAlert("代理设置失败", 1)
        )
        showAlert("代理配置成功")
      else
        showAlert("代理配置已存在")
    catch e
      logger.error("unknown proxy config error: " + e.stack)
  )
logger.log("injected #{proxyUrlElement.tagName} click") if proxyUrlElement

```

这段代码并不周全，例如缺少配置合法性检查。如果发生异常要排错怎么办？此时日志收集很关键。SwitchyOmega后端有日志导出功能，因此content-script里要把关键日志发送到后端

`background.coffee` 在`chrome.runtime.onMessage.addListener`里加入如下代码
```coffee
else if request.method == 'log.log' or request.method == 'log.error'
  target = options.log
  method = target[request.method.split(".")[1]]
```

`manifest.json` 加入content_scripts配置
```json
"content_scripts": [{
  "matches": ["*://example.com/*"],
  "js": ["js/content-script.js"]
}],
```

至此，核心代码就齐备了。构建，导入浏览器插件，祝开发顺利

## 其他参考

[前端科普系列](https://zhuanlan.zhihu.com/p/113009496)

[Chrome Extension Tutorial: How to Pass Messages from a Page's Context](https://www.freecodecamp.org/news/chrome-extension-message-passing-essentials/)

[How to make your chrome extension access webpage?](https://waitingphoenix.com/how-to-make-your-chrome-extension-access-webpage/)
