---
title: xcode常见问题
date: 2018-02-27 15:11:07
tags: [Xcode]
---

# 平常使用中xcode常见问题汇总
> xcode版本: xcode9.0
> Mac版本: 10.13 Beta (17A362a)

1. `App Installation failed, No code signature found.`
	- 解决办法: 到Xcode.app中（ /Applications/Xcode.app/Contents/Developer/Platforms/iPhoneOS.platform/Developer/SDKs/iPhoneOS.sdk ）修改SDKSettings.plist 文件中的一项CODE_SIGNING_REQUIRED的值，从NO改为YES，再重启XCode即可。
	- [Apple论坛](https://forums.developer.apple.com/message/170207#170207)
2. `build operations are disabled : 'Pods.xcodeproj' has changed and is reloading`
	- ![错误截图](Snip20180226_11.png)
	- 解决办法: 删除workspace,lock,pods


