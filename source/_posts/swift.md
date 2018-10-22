---
title: swift
date: 2018-01-31 14:20:49
updated: 2018-03-02 09:46:35
tags: [swift]
---
# swift4.0 使用错误示例及解决办法

1. `'HGL_TitleViewStyle' initializer is inaccessible due to 'internal' protection level`
	
	```
public class HGL_TitleViewStyle {
    
    public var titleViewHeight : CGFloat = 44
    public var normalColor : UIColor = UIColor.blue
    public var selectColor : UIColor = UIColor.orange
    public var titleSize : CGFloat = 17
    public var isScrollEnable = false
    public var itemMargin : CGFloat = 20
}
```
> 创建类时候没有继承类,可修改为 `public class HGL_TitleViewStyle : NSObject`

2. `cannot invoke initializer for type with an argument list of type swift`

	> 由于使用了没有权限的初始化方法,导致编译器找不到该方法,将该方法暴露,使用`public`关键字即可
	
3. `No such module ''

	> 由于使用的是podspec模式,在写子模块时候,调用到了其他的模块时候,不能再podfile中直接使用`pod install`,而应该是在spec中添加`dependency`
	
4. `Command /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/bin/swiftc failed with exit code 1`
	> 这个问题是由于在写子组件时候,物理删除了文件,但是在pod里面还有应用就报错,将pod中的引用删除即可
5. ` Use of unresolved identifier 'HD_TSShoppingCellModel'

	> 在编写pod字库时候,在本地文件已经存在,但是在项目的pod里面并未引用到的,需要重新 `pod install`即可
	
6. 引用pod字库中添加的图片资源
	> 我使用的是`pod lib create name` 的方式创建的模板工程,自带有`Assets` 和 `Classes`文件夹.
	当使用这个模式来创建字库时候,在spec中引用图片资源,应该使用:
	
	```
 s.resource_bundles = {
   'HD_AccountModule' => ['HD_AccountModule/Assets/HD_AccountModule.bundle/*']
 }
	```
	这样在宿主工程中引用到的就是在`source/HD_AccountModule.bundle`这个目录下的.
	> 当使用常用模式,在模板目录(HD_AccountModule/)下直接添加资源文件夹时候,引用图片资源应该使用:
	
	```
	  s.resources = 'HD_AccountModule/HD_AccountModule.bundle'
	```
7. pod子库中使用xib,连线后属性为nil
	> 因为xib的实例化默认是在`mainBundle`中去寻找,当宿主工程引用子库时,子库对应的`bundle`已经不是`mainBundle`了,而应该是对应的.
	解决办法:
	在使用xib的View或者VC中重写方法,并指定bundle
	
	```
 override init(nibName nibNameOrNil: String?, bundle nibBundleOrNil: Bundle?) {
        let bundle = Bundle.init(for: HD_LoginVC.self)
        super.init(nibName: "HD_LoginVC", bundle: bundle)
    }
	```
	8. `[!] The `HD_PassExperience_Example [Release]` target overrides the `SWIFT_VERSION` build setting defined in `Pods/Target Support Files/Pods-HD_PassExperience_Example/Pods-HD_PassExperience_Example.release.xcconfig'. This can lead to problems with the CocoaPods installation     - Use the `$(inherited)` flag, or     - Remove the build settings from the target.`,在使用pod时候安装第三方(我安装的是WCDB.swift)时候,运行`pod install`时候,终端报错.
	
	```
	这里说的是pod Pods-HD_PassExperience_Example.debug 这个文件中的 SWIFT_VERSION 与 target中的设置有冲突,虽然能够安装,但是编译时候回报错:
	在 target中setting,将SWIFT_VERSION删除(即设置成默认的)即可解决.
	```
	

# nav 是否隐藏问题

在自定义导航栏时候滑动滑返回时候由于导航栏是否隐藏需要使用 `setNavigationBarHidden`来设置,在设置`setNavigationBarHidden`之后,会关闭系统默认的右滑手势,需要手动开启,使用`navigationController?.interactivePopGestureRecognizer?.delegate = self`方法将滑动权限交由控制器本身**animated必须是TRUE,否则不起作用!!!**

```
override func viewDidLoad() {
    super.viewDidLoad()      
navigationController?.interactivePopGestureRecognizer?.delegate = self

}


override func viewWillAppear(_ animated: Bool) {
   super.viewWillAppear(animated)
   navigationController?.setNavigationBarHidden(true, animated: false)
}
    
override func viewWillDisappear(_ animated: Bool) {
   super.viewWillAppear(animated)          	navigationController?.setNavigationBarHidden(false, animated: false)
}
    
extension HD_TSSearchVC : UIGestureRecognizerDelegate{
    
}
```

# 在违常理的方法调用顺序上,有可能是方法循环调用

```
系统的右滑手势取消后还会调用viewWillDisappear
viewDidDisappear这两个方法
 override func viewWillDisappear(_ animated: Bool) {
       navigationController?.setNavigationBarHidden(false, animated: true)
        super.viewWillAppear(animated)
        print("viewWillDisappear")
    }
```
这个是原因是由于调用super时候调错了方法,在`viewWillDisappear`中调用了`super.viewWillAppear`,导致错误如下:

```
viewWillDisappear 第一次调用,正常
viewWillDisappear 由于调用了super.viewWillAppear后,当前方法是viewWillDisappear,导致按照生命周期流程继续往下调用,直到viewDidDisappear
viewDidDisappear

UIKit 是个黑盒，这一套生命周期的流程只是一套承诺，一套抽象，他承诺在 view 将会消失的时候调用 viewWillAppear，但你在错误的地方调用了错误的方法，会发生什么就只取决于它的具体实现，这一套实现有可能在 iOS 11 会像你说的这样，但在 iOS 9 或者 iOS 12 里也许就会完全不一样
```

