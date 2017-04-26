---
title: Build Version自增长
date: 2016-06-26 14:44:21
tags: 
  - Xcode
categories: 
  - 随记
  - iOS
---

参考：[Xcode自动生成版本号与根据版本号获取Git提交记录哈希值](http://wonderffee.github.io/blog/2015/09/24/auto-build-number/)

以git提交次数作为Build版本号，在每次Build的时候跑以下脚本。

<!-- more -->

```python
## Set the build number to the current git commit count. # If we're using the Dev scheme, then we'll suffix the b uild# number with the current branch name, to make collision s# far less likely across feature branches.# Based on: http://w3facility.info/question/how-do-i-for ce-xcode-to-rebuild-the-info-plist-file-in-my-project-ev ery-time-i-build-the-project/#git=`sh /etc/profile; which git`appBuild=`"$git" rev-list --all |wc -l`if [ $CONFIGURATION = "Debug" ]; thenbranchName=`"$git" rev-parse --abbrev-ref HEAD` /usr/libexec/PlistBuddy -c "Set :CFBundleVersion $appBui ld-$branchName" "${TARGET_BUILD_DIR}/${INFOPLIST_PATH}" else/usr/libexec/PlistBuddy -c "Set :CFBundleVersion $appBui ld" "${TARGET_BUILD_DIR}/${INFOPLIST_PATH}"fiecho "Updated ${TARGET_BUILD_DIR}/${INFOPLIST_PATH}"
```

但是可能不同机器上打包出来的Build版本号不一样。因为git clone的时候可以全量clone，也可以shallow clone，导致下面这条命令得出来的数字不一样。

``git rev-list --all | wc -l
``

可以用下面这条命令替换

``git rev-list --count HEAD
``


