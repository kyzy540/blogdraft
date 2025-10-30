---
title: "iTerm2 + Ollama启用AI功能"
date: 2025-10-28T17:36:04+08:00
tags:
  - it
  - macos
---

iTerm2 3.5后提供了AI功能，有2种用法:
* `Command + Shift + .`打开**composer**，输入自然语言后`Command + y`生成shell命令
* `Command + Shift + y`打开**AI chat**，关联iterm2 session作为上下文，和大模型问答

自然语言生成shell命令挺方便，不用去浏览器打开大模型网站，提问、复制、粘贴。安装步骤不复杂，参考文章:

[《Maximize Productivity with iTerm2 AI Features with Ollama 100% free》](https://voipnuggets.com/2024/11/29/maximize-productivity-with-iterm2-ai-features/)

也有别家文章介绍iTerm2 AI其他配法，例如输入API Key远程调用Dashscope。但我推荐Ollama，重点是数据安全
* Ollama是开源软件，本地运行大模型，数据不会上传云端
* Ollama支持多个大模型，如llama、qwen、deepseek等，可自行选择
* 除下载模型外，不依赖互联网
* 100%免费

当然，Ollama也非完美，有些功能没法用，例如`AI Completion`也就是命令自动补全。但我毕竟最优先考虑数据安全和免费，功能缺点也就凑合了。退一步想，如果生成shell命令足够快，自动补没有也罢

说完有的没的，聊具体安装。我基于上面博文做了些调整

用brew安装iTerm2 AI Plugin和ollama-app更方便
```bash
brew install --cask itermai
brew install --cask ollama-app
```

安装ollama-app图方便。不必每次敲`ollama serve`，还占一个session

*Configure AI Models Manually* 重点配置和文章一样，区别是模型选**deepseek-coder:1.7b**
>API: Llama
>
>API URL: http://localhost:11434/api/chat
>
>Model: deepseek-coder:1.7b

我测试了几个模型: qwen2.5:3b/codellama:7b/qwen2.5-coder:1.5b/qwen3:4b/deepseek-r1:1.5b/deepseek-coder:1.7b。 **deepseek-coder:1.7b** 最好用。生成命令一般5秒左右，慢也不超过10秒。如上所说，快才好使，不然就开浏览器得了

次优选是 *qwen2.5-coder:1.5b*，10~20秒生成命令。deepseek-r1和qwen3默认启用thinking，耗时超过1分钟，iTerm2会超时报错 (我没找到禁用thinking的方法，也许禁用后性能会不一样)

至于生成命令的质量，说实话每个模型都不如意。复杂的任务还得找云端大模型

最后，我的电脑是macbook air m3。频繁推理后机器会明显升温。没主动散热的笔记本还是不太适合跑大模型