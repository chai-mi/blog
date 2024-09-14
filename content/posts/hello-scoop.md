---
slug: hello-scoop
title: Hello! scoop
date: 2024-05-29 21:28:19
tags: 
  - Windows
categories: 
  - 记录
summary: 好吧，到刚刚为止我一直在手动安装 Windows 下任何程序，但我不想再一遍一遍手动升级电脑上的 python、golang、nodejs 了。是时候改变这一切了！
---

好吧，到刚刚为止我一直在手动安装 Windows 下任何程序，虽然说可以在安装时一睹官网的风采，但随着软件越来越多管理难度也越来越大，并且我真的不希望我偶尔打开一个软件的时候提示我可更新了。并且，我也不想再一遍一遍手动升级电脑上的 python、golang、nodejs 了

是时候改变这一切了！

# Windows 下的包管理器
Linux 下，包管理器帮助用户管理一切 bin（~~make install~~好吧，几乎一切）。自然有一个包管理器帮助我管理软件是极其方便的，这就是 scoop

虽然我早有耳闻 scoop，但也许是出于惰性，我一直没进行尝试。但是一经上手，熟悉的体验让我非常满意

## 安装
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
```
打开 powershell，敲下这两句，你已经成功安装了 scoop。是的，就是这么简单！接下来你要做就是 `scoop install <name>`，如果你熟悉 Linux 的话，一定会倍感亲切的

## 使用
让我们尝试安装 python
```powershell
scoop install python
```
你已经安装完成了，如此的省心！

## 常用命令
```powershell
# 更新所有软件
scoop update *
# 清理旧版本
scoop cleanup *
```
其他功能可 `scoop help` 或在 [scoop 官网](https://scoop.sh) 上查看

# 体验
不得不承认，我应该早点尝试 scoop 的，一个高效的工具可以使精力不必浪费在无用的地方。虽然我不提倡为工具而工具，但是 scoop 确实为我剩下了不少功夫！