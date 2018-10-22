---
title: '@synthesize与@dynamic'
date: 2018-03-29 18:14:45
tags: [OC]
---

> @synthesize will generate getter and setter methods for your property. @dynamic just tells the compiler that the getter and setter methods are implemented not by the class itself but somewhere else (like the superclass or will be provided at runtime).

# property

iOS6之后才出来的,这里 `property`, 包含
1. 自动生成getter,setter方法
2. 生成带下划线的成员变量
可以这么看 : 
`property` = `synthesize` + `_属性名`


# @synthesize
具体什么时候不知道...
1. 只生成getter,setter方法
2. 不能生成下划线的成员变量
**当使用了synthesize修饰后,不能够访问下划线变量,如果要访问,可以另外定义一个,赋值;或者直接将**

```
@synthesize age = _age; /// 这个意思使用_age和访问age是同样的值,就是可以理解为别名.但是这个_age并不是成员变量.
@synthesize age = newAge; /// 将age赋值给一个新的属性或者成员变量.使用self. 和新值同样的效果
```

# @dynamic
这个是告诉编译器,不要生成getter或者setter方法,由程序员手动来写,或者在运行时来生成.

```
@dynamic names;
```
使用@dynamic 时候,可以对父类中的属性进行重新定义,加入父类是只读,之类中添加@dynamic 后,可将其改为可读可写属性.

```
@interface Person : NSObject
@property(nonatomic,strong,readonly)NSArray * names;
@end

@interface Student : Person
@property(nonatomic,strong,readwrite)NSArray * names;
@end

@implementation Student
@dynamic names;
    
@end
```
这时候names 在子类中是可读可写


