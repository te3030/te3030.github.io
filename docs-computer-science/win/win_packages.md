# Windows 包管理器
Windows 上的命令行包管理器种类繁多，各自针对不同的开发需求和应用场景：

- Chocolatey 和 winget 适合常规软件管理。
- Scoop 更适合开发工具和小型应用。
- Conda、NPM 和 Yarn 主要服务于编程和开发环境，特别是 Python 和 JavaScript 生态。
- MSYS2 和 Vcpkg 更适合 C/C++ 开发者。
- NuGet 适用于 .NET 开发。

<!--more-->



## Scoop

### 安装

```bash

```

### 使用命令
```bash
# 安装软件
scoop install git

# 更新软件
scoop update

# 卸载软件
scoop uninstall git
```



## winget

### 安装

```bash
# 使用 scoop 安装 winget
scoop install winget
```

### 使用命令

```bash
# 安装软件
winget install <软件名称>

# 显示可升级软件
winget upgrade
```


### 常用软件安装

```bash
# 显示并执行可用的升级
winget upgrade
# PowerShell
winget install Microsoft.PowerShell
# Git
winget install Git.Git
# python
winget install  Python.Python.3.12
# Python.3.13
winget install Python.Python.3.13

# Mozilla Firefox (x64 en-US)
winget install Mozilla.Firefox

# PotPlayer-64 bit
winget install Daum.PotPlayer

# VLC media player
winget install VideoLAN.VLC

# XMind
winget install Xmind.Xmind

# Node.js
winget install OpenJS.NodeJS.LTS

# Eclipse Temurin JDK with Hotspot 8u332-b09 (x64)
winget install EclipseAdoptium.Temurin.8.JDK

# Go Programming Language amd64 go1.24.2
winget install GoLang.Go

# Microsoft Edge
winget install Microsoft.Edge

# Microsoft Visual C++ 2015-2022 Redistributable (x64) - 14.31.31103
winget install Microsoft.VCRedist.2015+.x64

# Microsoft Visual C++ 2013 Redistributable (x86) - 12.0.40660
winget install Microsoft.VCRedist.2013.x86

# Microsoft Visual C++ 2015-2019 Redistributable (x86) - 14.29.30139
winget install Microsoft.VCRedist.2015+.x86

# Microsoft Visual C++ 2013 Redistributable (x64) - 12.0.40660
winget install Microsoft.VCRedist.2013.x64

# 钉钉
winget install Alibaba.DingTalk

# Cherry Studio
winget install kangfenmao.CherryStudio

# Another Redis Desktop Manager 1.5.2
winget install qishibo.AnotherRedisDesktopManager

# balenaEtcher 1.18.11
winget install Balena.Etcher

# DeepL
winget install DeepL.DeepL

# Edit
winget install Microsoft.Edit

# Fork git client
winget install Fork.Fork
```




> 参考链接： https://www.cnblogs.com/suv789/p/18831144
