---
title: svn命令
date: 2018-01-31 11:28:01
tags: [SVN]
---


# 使用问题

## 在svn add 文件夹 命令后如果有文件被删除要commit时候会报错


1. `svn st` 
	
```
	?       异步队列测试
```

2. `svn add 异步队列测试/`

```
A         异步队列测试
A         异步队列测试/异步队列测试.xcodeproj
A         异步队列测试/异步队列测试.xcodeproj/project.pbxproj
A         异步队列测试/异步队列测试
A         异步队列测试/异步队列测试/ViewController.m
A         异步队列测试/异步队列测试/Assets.xcassets
A         异步队列测试/异步队列测试/Assets.xcassets/AppIcon.appiconset
A         异步队列测试/异步队列测试/Assets.xcassets/AppIcon.appiconset/Contents.json
A         异步队列测试/异步队列测试/Base.lproj
A         异步队列测试/异步队列测试/Base.lproj/Main.storyboard
A         异步队列测试/异步队列测试/Base.lproj/LaunchScreen.storyboard
A         异步队列测试/异步队列测试/main.m
A         异步队列测试/异步队列测试/AppDelegate.h
.....
```

> 在add之后将本地的某个文件删除后....

3. `svn commit -m "234"`

```
svn: E155010: Commit failed (details follow):
svn: E155010: '/Users/guolianghao/Desktop/HD/Fakao-lib/publicLib/trunk/异步队列测试/异步队列测试.xcodeproj' is scheduled for addition, but is missing
``` 

这个报错是由于`/Users/guolianghao/Desktop/HD/Fakao-lib/publicLib/trunk/异步队列测试/异步队列测试.xcodeproj`这个本地被删除了,提交SVN时候不能够识别的**解决办法**是

```
svn revert 异步队列测试 --depth infinity
```
> 这里的*异步队列测试*是将这个add的文件夹脱离SVN的控制,需要重新add
> 如果在本地删除的仅仅是某个文件只需要使用 `svn revert 异步队列测试`命令即可,使用`--depth infinity`是将整个文件深度处理.

