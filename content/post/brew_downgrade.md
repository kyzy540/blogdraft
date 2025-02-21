---
title: "brew 降级"
date: 2025-02-20T14:05:23+08:00
tags:
  - it
  - macos
---

参考: [Homebrew install specific version of formula?](https://stackoverflow.com/questions/3987683/homebrew-install-specific-version-of-formula)

以[alt-tab](https://formulae.brew.sh/cask/alt-tab)为例，从7.20.0 (有bug会崩溃) 降级到7.19.1
1. 打开 https://formulae.brew.sh/cask/alt-tab
2. 跳转 [Cask code on GitHub](https://github.com/Homebrew/homebrew-cask/blob/0ef28d3335279b043b131ffd4d5cfb3759837912/Casks/a/alt-tab.rb)
3. 点开 [alt-tab.rb History](https://github.com/Homebrew/homebrew-cask/commits/master/Casks/a/alt-tab.rb)
4. 找到要目标版本 (要降低到的版本)，点`View code at this point`
5. 下载 [alt-tab.rb 7.19.1](https://github.com/Homebrew/homebrew-cask/blob/cbdcf50a724cb180d1c64719d331c9b70d85eca5/Casks/a/alt-tab.rb)
6. 控制台进alt-tab.rb所在目录，执行`brew rnstall alt-tab.rb`

该方法虽然麻烦，但通用。如果软件预备了多版本，例如 [mysql-client](https://formulae.brew.sh/formula/mysql-client)，可以通过`brew install mysql-client@8.4`切版本，相对方便