---
title: "Zsh on macOS"
date: 2024-08-19T10:18:44+08:00
tags:
  - it
  - macos
---

前些日换Mac电脑，发现迁移zsh的配置有点繁琐，整了好一阵才顺手。索性写个文档记录下

先安装oh-my-zsh，和俩爱用的插件 (自动补全和高亮)

```bash
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)" && \
  git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions && \
  git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
```

再来个Dracula主题
```bash
git clone https://github.com/dracula/zsh.git
mv zsh/dracula.zsh-theme ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes
mv zsh/lib ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/themes
```

改`~/.zshrc`，启用主题和插件，禁用自动更新 (自动更新影响shell启动速度)
```bash
ZSH_THEME="dracula"
zstyle ':omz:update' mode disabled  # disable automatic updates
plugins=(git history zsh-autosuggestions zsh-syntax-highlighting)
```

对于常用的`alias`，参考`.zshrc`备注，写到`${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/aliases.zsh`

同理，在`custom`目录下写`pyenv.zsh`初始化`pyenv` (omz的pyenv插件不好使)。写`auth.zsh` export 常用认证相关环境变量

希望对每个命令都奏效的环境变量写到`~/.zshenv`
```bash
# read even when Zsh is launched to run a single command
export COPYFILE_DISABLE=1
# go
export GOPATH="$HOME/go"
export PATH="$GOPATH/bin:$PATH"
```

参考: [What should/shouldn't go in .zshenv, .zshrc, .zlogin, .zprofile, .zlogout?](https://unix.stackexchange.com/questions/71253/what-should-shouldnt-go-in-zshenv-zshrc-zlogin-zprofile-zlogout)

这样做的好处是简化`.zshrc`，避免臃肿

另外我惯用iterm2，快捷键要设`Profiles → Keys → Load Preset... → Natural Text Editing`。主题也用Dracula

最后是brew常用软件，个人和工作电脑都爱用
```
gnu-sed
hugo
iproute2mac
telnet
trash
wget
alt-tab
flameshot
iterm2
maccy
sublime-text
```