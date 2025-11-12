---
title: "Siri接大模型唠嗑"
date: 2025-11-12T10:58:24+08:00
draft: true
tags:
  - it
  - iphone
---

## 简介

创建一个 *快捷指令* 供Siri使用。快捷指令会循环听用户问题，调API聊天接口，朗读大模型回答。可以多轮问答，包含上下文

聊天接口使用阿里云百炼，可以改成其他兼容OpenAI API标准的

必须提供API Key。**不免费**

## 用法

下载快捷指令: [唠嗑]()

将`Bearer sk-xxx`替换成有效的API Key。具体获取方法参考接口提供方文档。例如[阿里云获取API Key](https://bailian.console.aliyun.com/?tab=api#/api/?type=model&url=2712195)。就可以使用了

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

如果觉得不好算，实践更直观。为了确保不少算，我把提示词限制升到80个单词 (`Answer briefly, under 80 words`)。结果如下

3轮问答消耗消耗11K Tokens (输出10.5K, 输入0.5K)。请求Token数匹配最低档位，因此花费 0.0015 * 10.5 + 0.00015 * 0.5 = 0.015825元

如果还不够直观，再加个时间视角: 10元可以跑630次快捷指令，每次问满3轮。均摊到一年折合每天可跑1.7次快捷指令

顺带一提，问答包含上下文，因此每轮Token数会累积变大，例如第3轮输入包含了前2轮的问题和回答。因此，如果把回答限制到40个单词，只问2轮，消耗的Token可以降到5K或更低

总结: 偶尔用的话，10元管一年

## 实现

快捷指令实现的功能其实很简单，碍于没法把它导出成文本，只好用等效python代码展示逻辑

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

至此，礼貌的干货部分结束。下面是吐槽环节

## 苹果真傻逼

快捷指令傻逼，AI功能迟迟不引进傻逼，iPhone 16发布前宣传很快能用上AI更是傻逼中的傻逼。苹果你要好好搞