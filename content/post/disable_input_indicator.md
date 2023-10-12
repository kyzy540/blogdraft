---
title: "macOS Sonoma 关闭光标输入法提示"
date: 2023-10-12T11:00:52+08:00
tags: 
  - macos
  - input indicator
  - sonoma
---

Sonoma在每次切换输入法时都会在光标下跳一个提示框。如果装了`AltTab`的话，输入法提示会时不时自动蹦出来，很烦人。可以用如下命令禁用提示框。执行完需要重启系统才能生效

```bash
sudo mkdir -p /Library/Preferences/FeatureFlags/Domain
sudo /usr/libexec/PlistBuddy -c "Add 'redesigned_text_cursor:Enabled' bool false" /Library/Preferences/FeatureFlags/Domain/UIKit.plist
```

参考: https://stackoverflow.com/questions/77248249/disable-macos-sonoma-text-insertion-point-cursor-caps-lock-indicator