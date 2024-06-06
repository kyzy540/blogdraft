---
title: "macOS 录屏配字幕"
date: 2024-06-06T14:59:51+08:00
tags: 
  - macos
---

我想做个简短的软件演示视频

如果有条件安装iMovie，可能很方便。但我的电脑因特殊原因没法登录App Store，下不了iMovie。只能拿来系统录屏功能+ffmpeg凑合

我先准备`srt`字幕再录视频。`srt`内容很简单，可以当大纲用

```
1
00:00:00,000 --> 00:00:04,000
hello

2
00:00:04,000 --> 00:00:07,000
world
```

准备好后，按下 Shift-Command-5 打开“截屏”工具，开始录制

录制好的文件会被保存城`.mov`格式，可以用ffmpeg转换成`mp4`
```
ffmpeg -i input.mov -c:v libx264 -c:a aac -strict experimental output.mp4
```

最后根据视频调整`srt`字幕，合入视频。大功告成
```
ffmpeg -i output.mp4 -vf subtitles=subtitle.srt -y output_with_sub.mp4
```

参考:

[Mac OS中利用ffmpeg为视频添加字幕](https://developer.aliyun.com/article/996308)

[在 Mac 上截屏或录屏](https://support.apple.com/zh-cn/guide/mac-help/mh26782/mac)

[ffmpeg将mov转换为mp4](https://cloud.tencent.com/developer/information/ffmpeg%E5%B0%86mov%E8%BD%AC%E6%8D%A2%E4%B8%BAmp4-ask)

[基于ffmpeg给视频添加时间字幕](https://blog.csdn.net/a486259/article/details/133718711)