---
title: "Siri接大模型唠嗑"
date: 2025-11-12T10:58:24+08:00
tags:
  - it
  - iphone
---

## 简介

创建一个 *快捷指令* 供Siri使用。快捷指令会循环听用户问题，通过API发给大模型，后朗读回答。可以多轮问答，包含上下文

API使用阿里云百炼，可以改成其他兼容OpenAI标准的

必须提供API Key。**不免费**

## 用法

下载快捷指令: [唠嗑](https://www.icloud.com/shortcuts/fc956f86a62044e08a9c89bec49e6be8)

将`Bearer sk-xxx`替换成有效的API Key。具体获取方法参考接口提供方文档，例如[阿里云获取API Key](https://bailian.console.aliyun.com/?tab=api#/api/?type=model&url=2712195)。就可以使用了

以下参数可以按需调整
* `model_name`: 默认`qwen-flash`，可以改成其他模型，如`qwen-plus`
* URL: 兼容OpenAI标准的endpoint
* 重复`3`次: 控制问答轮次，默认最多3轮。控制轮次是为了节约Token。由于问答包含上下文，轮次越多携带上下文越多，Token消耗越多
* `enable_search: True`: 允许模型推理过程中联网，默认开启。模型更好回答时效性问题。代价是会多消耗些许Token
* prompt `Answer briefly, under 40 words`: 提示词。建议限制字数，否则回答可能很冗长。提示词也算Token，其本身不建议写长

## 费用

价格请以服务商官网实时报价为准。本文价格基于2025-11-12阿里云官网。`qwen-flash`报价为

| 单次请求的输入Token数 | 输入单价（每千Token） | 输出单价（每千Token） |
|------------|------------------------|------------------------|
| 0<Token≤128K | 0.00015元 | 0.00015元 |
| 128K<Token≤256K | 0.0006元 | 0.006元 |
| 256K<Token≤1M | 0.0012元 | 0.012元 |

1个Token通常对应一个汉字，或一个英语单词。看起来不贵，但实际计费不仅按问题的字数算，还有提示词、搜索网络等其他计费项

如果觉得不好算，实践更直观。为了确保不少算，我把提示词限制升到80个单词: `Answer briefly, under 80 words`

3轮问答消耗消耗11K Tokens (输出10.5K, 输入0.5K)。请求Token数匹配最低档位，因此花费 0.0015 * 10.5 + 0.00015 * 0.5 = 0.015825元

如果还不够直观，再加个时间视角: 10元可以跑630次快捷指令，每次问满3轮。均摊到一年折合每天可跑1.7次

再次说明，问答包含上下文，因此每轮Token数会累积变大，例如第3轮输入包含了前2轮的问题和回答。如果把回答限制到40个单词，只问2轮，消耗的Token可以降到5K或更低

总结: 偶尔用的话，10元管一年

## 实现

功能其实很简单，碍于没法把快捷指令导出成文本，只好用等效python代码展示逻辑

```python
import requests

model_name = "qwen-flash"
api_key = "sk-xxx"
answer = ""
url = "https://dashscope.aliyuncs.com/compatible-mode/v1"
headers = {"Content-Type": "application/json", "Authorization": "Bearer " + api_key}
messages = [{"role": "system", "content": "Answer briefly, under 40 words"}]

for i in range(3):
    if answer:
        messages.append({"role": "assistant", "content": answer})

    question = input("Question: ")
    messages.append({"role": "user", "content": question})
    body = {"model": model_name, "enable_search": True, "messages": messages}
    response = requests.post(url, headers=headers, json=body)
    answer = response.json()["choices"][0]["message"]["content"]
    print("Answer: ", answer)

```

## 参考

附上2个我参考的来源。感谢二位大佬

[【DeepSeek接入苹果Siri，只需3步，保姆级教程｜iPhone教程｜快捷指令】](https://www.bilibili.com/video/BV1t9NgeLE4B/?share_source=copy_web&vd_source=24b9548cc4159506bc478709fee25105)

https://github.com/fatwang2/siri-ultra

*注意: siri-ultra的快捷指令不兼容OpenAI标准，服务商是search1api。我不知道它的来头*

至此，礼貌的干货部分结束。下面是吐槽环节

## 苹果真傻逼

快捷指令傻逼，AI功能迟迟不引进傻逼，iPhone 16发布会宣传AI更是傻逼中的傻逼

先说虚假宣传。AI没做好，就先承诺出去。作为当时市值第一的公司，恬不知耻。更糟糕的是一年多下来，AI部门人员换了几茬，东西是一点起色没有。不诚实还没能力

自己AI不行算了，引进中国也不推进。跟百度聊，跟阿里聊，宣传半天，就是没动作。给我急坏了

当今AI的发展速度一年一个飞跃。2024的ChatGPT在2025年的DeepSeek面前嫩得像个雏。苹果浪费一年，差距落下可不是一星半点。闹到最后大概还得接第三方API，图啥

快捷指令我也不能忍。这十几年的玩意儿一点进步都没有。自己不好好搞AI，还不给用户提供顺手工具。罪加一等

快捷指令本质就是编程，只不过基于图形UI的脚本。但既然是编程，就该搭配高效编程工具。基于图形UI给一个变量赋值低效到离谱。除了工具，编程文档也很重要，苹果也没做好。例如上面的代码，在一般编程语言里把`body`变量声明在循环外和循环内应该都能用，比方下面的代码应该等效

```python
# 省略
body = {"model": model_name, "enable_search": True}
for i in range(3):
    # 省略
     body["messages"] = messages
     print(body)
    # 省略
```

但实际上不行。用 *给词典键赋值* 的指令在循环赋值`body的messages键`，`print(body)` 打印的词典永远不包含`messages`。没有官方文档解释其中的奥妙

官方没文档，那社区呢？你看，编程是代码，有代码才能分享、传播。苹果最次也应该提供转文本的能力。还是那话，十几年了，啥也没有，社区无从谈起

我分享的这个版本，是迭代的第3版。为啥这么简单的功能要迭代3版？就因为这破玩意儿连变量赋值都不符合常理，各个步骤都是硬试出来的，走不少弯路。有种前互联网时代编程的感觉，没苦硬吃
