---
title: 创建私有Pod
date: 2015-012-2 14:03:40
tags: 
  - Pod
categories: 
  - 随记
---



### 创建私有Pod 

**需要有两个Git仓库， 个作为私有库的索引，另 个存放私有库的代码。**
### 创建索引库
在命令  输  pod repo 可以查看本地的Pod索引库，默认只有 CocoaPods官 的索引库 master 。

在Git服务 上创建 个私有仓库，执 以下命令创建 个本地私有索引 库，再执  pod repo 就会有两个索引库 。
`pod repo add [Private Repo Name] [git clone URL]`

<!-- more -->
### 创建私有Pod`pod lib cerate [PrivatePodLibrary]`
创建的时候可以选择Pod是基于OC还是Swift的，是否包含Demo等。然 后添加代码，编写pod spec，验证Pod
`pod lib lint --allow-warnings --verbose`
验证通过之后把push pod刷新索引库
`pod repo push [Private Repo Name] [Pod Name.spec] --allo w-warnings`       
### 使用私有Pod 
 
 在Podfile 加 私有索引库的URL，应该是下 这样的，第  是CocoaPods官 索引库，第  是私有索引。然后 `pod install` 。

```
source 'https://github.com/CocoaPods/Specs.git'source [Privete Repo URL]
```