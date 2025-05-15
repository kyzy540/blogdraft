---
title: "pyenv迁移uv"
date: 2025-05-15T13:48:54+08:00
tags:
  - it
---

pyenv和uv是俩python工具，都可以管理版本。只不过后者还提供管理项目的功能。看他们github介绍可见一斑

[pyenv](https://github.com/pyenv/pyenv)
> Simple Python version management

[uv](https://github.com/astral-sh/uv)
> An extremely fast Python package and project manager, written in Rust.

uv所谓的项目管理，包含构建、测试、发布等功能。传统软件工程的老几样。显然管辖范围比pyenv大。在uv之前python的项目管理工具还有poetry、pdm等。综合比较项目活跃度、易用性、性能后，uv是其中较好的一款工具

关于项目管理不展开说，仅介绍如何从pyenv迁移到uv。这对开发者来说通常是绕不开的第一步

如果有耐心，可以先看[uv官方文档](https://docs.astral.sh/uv/getting-started/#using-uv-build)。下面是实操

1. 安装uv，在macos下用brew
```bash
brew install uv
```

2. 进入项目目录，初始化项目
```bash
uv init
```

此时，uv会创建`.python-version`、`main.py`、`pyproject.toml`等文件，和`.venv`目录，对应未初始化的虚拟环境。对于已有项目的迁移，程序入口和 Python 版本通常已经明确，可以保留 `pyproject.toml` 和 `.venv`，其余删除

4. 编辑`pyproject.toml`，指定python版本和pyenv一致

5. 导入依赖包
```bash
uv add -r requirements.txt
```

虽然导依赖只有一行命令，但它其实做了3件事: 用指定python版本真正初始化`.venv`虚拟环境 -> 修改`pyproject.toml`文件添加依赖 -> 安装依赖包。如果第一次运行且依赖多。这里可能会多花些时间

顺带一提，uv是多线程装包，速度比pyenv+pip快得多

6. 删除pyenv初始化配置，通常在`.zshrc`等地方。记得 **重启shell**。否则`source .venv/bin/activate`可能会失效。这也是我放弃pyenv的原因之一。它可能引入冲突，难以排查
```bash
#eval "$(pyenv init -)"
#eval "$(pyenv virtualenv-init -)"
```

到这其实迁移已完成。为了方便，可以加点佐料

*7. 安装direnv*
```bash
brew install direnv
```

*8. 启用direnv和python的oh-my-zsh插件*
```bash
# vi ~/.zshrc
plugins=(... python direnv)
PYTHON_VENV_NAME=".venv"
PYTHON_AUTO_VRUN=true
source $ZSH/oh-my-zsh.sh
```

direnv方便自动设环境变量 (如`PYTHONPATH`)，写在`.envrc`里即可。python插件可以自动启用/退出虚拟环境，注意`PYTHON_VENV_NAME`和`PYTHON_AUTO_VRUN`，必须在`source $ZSH/oh-my-zsh.sh`前声明

这样cd项目目录时直接可以投入使用，离开目录自动清空设置。相当方便

由于以上插件都目录相关，推荐使用zsh和[autojump](https://github.com/wting/autojump)，方便跳转

## 可能遇到的问题

最后补充一点工具无关的。如果在ARM架构上装grpcio包，可能会遇到变异报错`distutils.compilers.C.errors.CompileError`。导包前加下面环境变量可解决
```bash
export LDFLAGS="-L/opt/homebrew/opt/openssl@3/lib"
export CPPFLAGS="-I/opt/homebrew/opt/openssl@3/include"
export GRPC_PYTHON_BUILD_SYSTEM_OPENSSL=1
export GRPC_PYTHON_BUILD_SYSTEM_ZLIB=1
```