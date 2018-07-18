---
layout: blog
article: true
category: git
title:  git多个远程仓库
date:   2018-07-12 23:45:27
background-image: ../style/images/2018-06/bwar-article.png
tags:
- git
---

### 前言
&emsp;&emsp; 用 GitHub 管理自己的开源项目有几年了，最近一年更新得比较多，仓库也越来越多越来越大。有时候感觉GitHub太慢，尤其是最近感觉更为明显，于是萌生了再找个国内类似GitHub的代码托管平台的想法，同时我也还想持续更新GitHub上的仓库，于是需要一个本地仓库（我自己的开发机）多个远程仓库（Github、码云、coding）。

### 一个远程仓库的 git config
&emsp;&emsp; 我的开源项目 [Nebula](https://github.com/Bwar/Nebula)一个基于事件驱动的高性能TCP网络框架的git配置文件.git/config如下：
```
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = https://github.com/Bwar/Nebula.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
        remote = origin
        merge = refs/heads/master
```

### 用 git 命令行添加多个远程仓库
 &emsp;&emsp; 添加一个名为“ mirror ”的远程仓库：
```
git remote add mirror https://gitee.com/Bwar/Nebula.git
```
&emsp;&emsp; 执行完这条命令后 .git/config 文件内容变成了：
```
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = https://github.com/Bwar/Nebula.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
        remote = origin
        merge = refs/heads/master
[remote "mirror"]
    url = https://gitee.com/Bwar/Nebula.git
    fetch = +refs/heads/*:refs/remotes/mirror/*
```
&emsp;&emsp; 此时已经是一个本地仓库，两个远程仓库。使用下面的命令可以分别从两个远程仓库拉取和推送到两个远程仓库。
```
git pull origin master 
git pull mirror master
git push origin master 
git push mirror master
```
### 一条命令同时更新多个远程仓库
&emsp;&emsp;目前我的开源项目只有我一个 contributor （计划2018年12月开始引入其他contributor），主要push比较少pull，输入多条命令我都觉得麻烦，一条命令将当前分支同时更新到两个远程仓库才能让我满意。于是改变一下，不用上面的mirror做法，直接在origin中添加一个url来实现一个本地仓库多个远程仓库。
```
git remote set-url --add origin https://gitee.com/Bwar/Nebula.git
```
&emsp;&emsp;执行这条命令后 .git/config 内容变成：
```
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = https://github.com/Bwar/Nebula.git
        url = https://gitee.com/Bwar/Nebula.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
        remote = origin
        merge = refs/heads/master
[remote "mirror"]
    url = https://gitee.com/Bwar/Nebula.git
    fetch = +refs/heads/*:refs/remotes/mirror/*
```
&emsp;&emsp;之前添加的“ mirror ”留着或删掉都没关系，这时候我们一条命令即可更新两个远程仓库：
```
git push origin master
```

### 免输入密码操作远程仓库
&emsp;&emsp;执行远程仓库操作需要输入密码是件比较麻烦的事情，在配置文件的url里配上用户名和密码即可免掉这样的麻烦，提高操作效率。免输密码操作远程仓库还可以通过ssh方式实现，下面只给出https方式的免输密码配置：
```
url = https://${user}:${password}@github.com/Bwar/Nebula.git
```
&emsp;&emsp;把上面配置中的“ ${user} ”和“ ${password} ”用你的远程仓库用户名和密码代入即可。

### 直接修改 git 配置文件实现多个远程仓库
&emsp;&emsp;上面通过 git remote 命令完成一个本地仓库多个远程仓库配置，这些命令实际上都是通过修改.git/config实现的，其实直接修改配置文件可能会更快，我就是直接修改配置文件完成。最后我的多个远程仓库配置如下：
```
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = https://${user}:${password}@github.com/Bwar/Nebula.git
        url = https://${user}:${password}@gitee.com/Bwar/Nebula.git
        url = https://${user}:${password}@git.coding.net/Bwar/NebulaBootstrap.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "master"]
        remote = origin
        merge = refs/heads/master
```
&emsp;&emsp;完毕。

