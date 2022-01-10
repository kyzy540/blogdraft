---
title: "微软好终端"
date: 2022-01-10T18:39:01+08:00
keywords:
  - windows
  - terminal
  - wsl
  - gow
---

Windows发展到今日，已非复吴下阿蒙。被诟病多年的"终端难用"问题已经有了不错的解决方案。本文分享几个让我受益颇多的工具

在介绍工具前，读者应了解几点知识:
* 掌握一种常用SHELL，例如bash,
* 知道终端和SHELL的区别。如果不了解，请读[博文](https://www.hanselman.com/blog/WhatsTheDifferenceBetweenAConsoleATerminalAndAShell.aspx)
* 知道cygwin，以及它和Linux/Mac下终端的区别
* 知道Gnu和Linux/Unix的区别

---

## Powershell

Powershell早在Windows7时代就被捆绑在系统内，在当代Windows电脑上基本都能用。我不推荐Powershell，但有时条件受限，不得不依仗它，例如
* 只能用早期操作系统，例如Windows7
* 没有Administrator权限，无法添加/关闭系统功能
* 机器性能贫瘠，跑cygwin都迟钝
* SDK或build环境局限于Windows

Powershell本身并不差，很多时候用户只因没法在其中使用grep/less/vim等命令，而抱怨不已。但其实这问题有解

[GOW](https://github.com/bmatzelle/gow/wiki) (Gnu On Windows)囊获了100+个常用Gnu命令。它开源、免费，只包含一堆EXE文件，大小18MB左右。我日常用到的Gnu命令都在其中

想找出java文件中包含Log关键字最多的前3名？

```powershell
PS C:\Users> grep -c Log *.java | sort {[int]$_.split(":")[1]} -D | head -n 3
A.java:20
B.java:13
C.java:7
```

其中grep和head来自GOW，sort来自Powershell。如何确定这些命令的来源？

```powershell
PS C:\Users> which head
C: \Program Files (x86)\Gow\bin\grep.EXE

PS C:\Users> which head
C: \WINDOWS\system32\sort.EXE
```

有了GOW，Powershell已经能应付简单的终端任务

另外，Powershell支持history，用户可以很方便地用ctrl+r回溯历史命令，这是cmd所不具备的

### 可选工具

下面介绍的工具可能不适合所有人，只是我比较常用，不装也无妨

[choco](https://chocolatey.org/) 是Windows下包管理工具，运行在终端内，类似apt和yum。推荐给爱尝鲜新软件的读者。以安装GOW为例

```bat
choco install gow
```

[ZLocation](https://github.com/vors/ZLocation) 类似autojump，是一款目录导航工具，可以理解为cd命令的补充。它能让用户在常用目录间左右横跳

[ripgrep](https://github.com/BurntSushi/ripgrep) 是更好用的grep，用于文本搜索。我认为其性能和易用性都优于grep。例如下面命令会递归搜索当前目录下 .java和 .properties 文件，找 Log 关键词。rg会自动排查隐藏文件和二进制文件 (.class和.jar)，搜索效率很高

```bat
rg -tjava Log
```

[jq](https://github.com/stedolan/jq) 是个JSON解析工具，对于经常写脚本解析JSON数据的朋友有帮助

至此，如果读者还在用Powershell默认的蓝底白字配色做开发，估计已经快瞎了。试试[Dracula Theme](https://draculatheme.com/powershell/)，个人感觉还不错。Powershell的美化方子还有很多，例如[oh-my-posh](https://github.com/JanDeDobbeleer/oh-my-posh)，这里不展开介绍

## WSL

WSL全称Windows Subsystem for Linux，介绍请戳[官方文档](https://docs.microsoft.com/zh-cn/windows/wsl/)。简单点说，它是个自下而上完整的Linux系统。比虚拟机更轻、更快、更强（可以访问Windows文件系统，运行EXE）。再简单点说，它同时具备Linux和Windows的优点，好用得不讲理

使用WSL需要Windows10和Administrator权限，详细请参考[安装指南](https://docs.microsoft.com/zh-cn/windows/wsl/install-win10)

完整的Linux终端下，能玩的花活不计其数。这里只提几点心得

![img](/img/windows_terminal/p73572541.jpg)

Properties->Options 可以开启复制/粘体快捷键。如果看不到如图选项，说明系统版本过低，请升级。系统版本可以用winver命令查看

expect可以方便地实现自动化登录功能。zssh可以实现rz/sz 上传/下载远端数据。用惯XShell刚转投WSL的朋友可以试试这2个工具

alias 可以让调用EXE更方便，我的.bashrc有如下配置
```
alias java='java.exe'
alias jar='jar.exe'
alias subl='subl.exe'
alias ii='explorer.exe'
```

为什么把explorer.exe设成ii？因为Powershell中打开资源管理器命令是ii。例如，```ii .```会在当前目录下打开资源管理器。WSL2支持explorer.exe浏览WSL文件系统，WSL1不行

Linux下调用EXE有什么好处？举个例子: 在sublime text中打开包含Log关键词的java文件，命令如下
```bash
for i in $(grep -l Log *.java); do subl $i; done
```

## Windows Terminal

![img](/img/windows_terminal/p73572542.jpg)

[Windows Terminal](https://en.wikipedia.org/wiki/Windows_Terminal) 是我用过最好的终端工具之一。支持pane、tab、高度可定制等，优点不一而足。[官方文档](https://docs.microsoft.com/zh-cn/windows/terminal/)也非常详实

再次提醒，Windows Terminal(wt)不是SHELL。安装wt不会自带WSL，后者必须另外装。如果读者不了解terminal和SHELL的区别，也不爱看博文，那以一个不恰当的比喻解释就是: SHELL好比MOBA游戏中的英雄，terminal好比皮肤。没解锁英雄，皮肤是用不上的

wt可自定义的配置很多，稍加调教就能得心应手。下面贴出部分我的配置供参考

```json
{
    // 默认启动WSL而非Powershell
    "defaultProfile": "{2c4de342-38b7-51cf-b940-2309a097f518}",

    // 选中字符自动拷贝到剪切板
    "copyOnSelect": true,

    // 分隔符不包含"."(dot)，方便双击选中IP和完整的文件名
    "wordDelimiters": "/\\()\"'-:,;&lt;&gt;~!@#$%^&amp;*|+=[]{}~?│",

    "profiles":
    {
        "defaults":
        {
            // 光标形状
            "cursorShape": "filledBox",
            // 退出时关闭tab。若不设always，退出时遇到"process exited with code xxx" tab不会关闭
            "closeOnExit": "always"
        },
        "list":
        [
            {
                "guid": "{2c4de342-38b7-51cf-b940-2309a097f518}",
                "hidden": false,
                "name": "Ubuntu",
                "source": "Windows.Terminal.Wsl",
                // 启动目录为WSL中的$HOME
                "startingDirectory": "\\\\wsl$\\Ubuntu\\home\\xxx"
            }
        ]
    }
}
```

截至到2020年6月，wt最新版本为1.0。功能和性能都比较稳定，但问题还是有的
* 亚克力效果 ("acrylic": true)对性能有损耗
* 暂不支持传统的"透明度"调整
* wt性能低于native console (直接启动WSL活Powershell)

目前想到的就这些，往后有新货再补充。最后，win键，wt，enter，enjoy!