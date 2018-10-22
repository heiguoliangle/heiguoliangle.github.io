---
title: NSString中copy&strong
date: 2018-02-28 16:20:20
tags: [oc]
---

# @property参数
1. 通过`property`声明的属性会自动生成setter/getter方法.
2. `nonatomic` <--> `atomic`
3. `readwrite` <--> `readonly`
4. `strong` / `copy` / `assign`


## synthesize
有些代码可能会使用`@synthesize`关键字(iOS5之前).**iOS5之后,property已经优化了该属性,自动生成setter/getter方法**

```
Person.h

@property (nonatomic, assign) NSInteger age;
//  当编译器编译到上面这行代码的时候，会自动生成name的setter and getter方法的声明
- (void)setAge:(NSInteger)age;
- (NSInteger)age;
Person.m
@implementation Person
@synthesize age = _age;
    
-(void)setAge:(NSInteger)age{
    _age = age;
}
-(NSInteger)age{
    return _age;
}

//  当编译器编译到上面的代码，会在Person.m里生成一个私有的 _name 实例变量，并自动生成setter and getter的实现部分
```
	使用了`synthesize`关键字后,就是告诉编译器要自己生成两个方法,其中
`@synthesize age = _age;`这句话的意思就是`age`属性给`_age`的成员变量生成setter/getter方法,在setter中实际操作的是`_age`.
> 通过使用`synthesize`关键字,可以在赋值操作时候重新定义一个接受值的成员变量,使得命名更清晰.


## `atomic` / `nonatomic`
atomic: 原子属性, 生成的setter和getter方法是一个原子操作。如果有多个线程同时调用setter的话，不会出现某一个线程执行setter全部语句之前，另一个线程开始执行setter的情况，相当于方法头尾加了锁一样。虽然安全性高，但是会导致程序特别的卡（开发中一般不用这个属性）
nonatomic : 非原子属性，多线程的情况下数据可能会有问题，但是会提高性能。

## `readwrite` / `readonly`
readwrite:默认属性，系统会自动生成setter 和 getter方法的声明与实现.
readonly:只读属性，只会生成getter不会生成setter
## `assign`  
这个属性一般处理基本数据类型，比如int,char,float等，assign是默认的，可以不加这个属性。并且不会更改引用计数。
# copy / strong 
## NSString赋值操作
无图无真相:

```
@interface ViewController ()
    
@property(nonatomic,copy)NSString * copyedStr;
@property(nonatomic,strong)NSString * strongStr;

@end
```
首先声明了一个`copy`和`strong`类型的字符串.
```
NSString * originStr = @"9999";
    self.copyedStr = originStr;
    self.strongStr = originStr;
    NSLog(@"originStr: %p . address : %p",originStr,&originStr);
    NSLog(@"_copyedStr: %p . address : %p",_copyedStr,&_copyedStr);
    NSLog(@"_strongStr: %p . address : %p",_strongStr,&_strongStr);
```
对`copy`及`strong`类型的属性进行赋值**这里使用的是setter方法,而不是成员变量**,输出结果:

```
2018-02-28 16:59:49.368224+0800 copystrong[60607:2244516] originStr: 0x10c8570a0 . address : 0x7ffee33a7af8
2018-02-28 16:59:49.368350+0800 copystrong[60607:2244516] _copyedStr: 0x10c8570a0 . address : 0x7fb15bf0da70
2018-02-28 16:59:49.368465+0800 copystrong[60607:2244516] _strongStr: 0x10c8570a0 . address : 0x7fb15bf0da78
```
从上可以看出来,不管是`copy`还是`strong`都指向了同一个地址.

```
(lldb) memory read 0x10c8570a0
0x10c8570a0: e0 5f ac 0d 01 00 00 00 c8 07 00 00 00 00 00 00  ._..............
0x10c8570b0: d9 63 85 0c 01 00 00 00 04 00 00 00 00 00 00 00  .c..............
(lldb) p (char*) 0x10c8563d9
(char *) $0 = 0x000000010c8563d9 "9999"
(lldb) 
```
由此可查看出来在指向地址的高一位时,地址上存储的值是对变量的值.

在此之后对原来的str进行重新赋值,看看有什么变化.

```
originStr = @"newStr";
```
之后重新输出:

```
2018-02-28 17:04:48.902472+0800 copystrong[60607:2244516] originStr: 0x10c857120 . address : 0x7ffee33a7af8
```
此时`originStr`的指针不变,但是指向的地址生变化,重新申请了一份内存.继续查看内存上的地址:

```
(lldb) memory read 0x10c857120
0x10c857120: e0 5f ac 0d 01 00 00 00 c8 07 00 00 00 00 00 00  ._..............
0x10c857130: 37 64 85 0c 01 00 00 00 06 00 00 00 00 00 00 00  7d..............
(lldb) p (char *)0x10c856437
(char *) $1 = 0x000000010c856437 "newStr"
```
同样的,在该地址的高一位找到了我们赋值的值.再看看通过`copy`和`strong`修改的值:

```
2018-02-28 17:08:00.954336+0800 copystrong[60607:2244516] _copyedStr: 0x10c8570a0 . address : 0x7fb15bf0da70
2018-02-28 17:08:00.954810+0800 copystrong[60607:2244516] _strongStr: 0x10c8570a0 . address : 0x7fb15bf0da78
```
还是第一次赋值的地址,且内容也没有变化.
总结一下哈: 
1. 通过多次赋值操作,字符串地址的变化大致得出结论,还有待证明(如果有大佬指出问题所在,请斧正!!!)
![多次赋值地址变化](Snip20180228_20.png)
2. 针对使用`NSString`对`copy`或`strong`修饰的属性没有任何影响.

## NSMutableString赋值操作
没代码还说个🐓

```
NSMutableString *originStr = [NSMutableString stringWithFormat:@"heiguo"];
    
    self.copyedStr = originStr;
    self.strongStr = originStr;
    NSLog(@"originStr: %p . address : %p",originStr,&originStr);
    NSLog(@"_copyedStr: %p . address : %p",_copyedStr,&_copyedStr);
    NSLog(@"_strongStr: %p . address : %p",_strongStr,&_strongStr);
```
同样的操作,使用`setter`方法进行赋值.结果如下:

```
2018-02-28 17:14:23.468627+0800 copystrong[60820:2254432] originStr: 0x60400044d290 . address : 0x7ffee67e6af0
2018-02-28 17:14:23.468759+0800 copystrong[60820:2254432] _copyedStr: 0xa006f75676965686 . address : 0x7f9aa86385a0
2018-02-28 17:14:23.468834+0800 copystrong[60820:2254432] _strongStr: 0x60400044d290 . address : 0x7f9aa86385a8
```
在这里细看:
1. originStr 和 strongStr 地址是一样的.
2. copyStr 地址是一个很大的值(此时这个地址在堆上)

直接使用`po`:

```
(lldb) po 0xa006f75676965686
heiguo
```
此刻及证明第二点.
同样的进行重新赋值操作:

```
[originStr appendFormat:@"append"];
```
继续输出:

```
2018-02-28 17:17:48.128857+0800 copystrong[60820:2254432] originStr: 0x60400044d290 . address : 0x7ffee67e6af0
2018-02-28 17:17:48.129178+0800 copystrong[60820:2254432] _copyedStr: 0xa006f75676965686 . address : 0x7f9aa86385a0
2018-02-28 17:18:04.687150+0800 copystrong[60820:2254432] _strongStr: 0x60400044d290 . address : 0x7f9aa86385a8
```

结合第一次输出:
1. 在修改originStr 后,`strong`修饰的变量值跟着改变为新的地址(同新的originStr一致).
2. 即使修改originStr后, `copy`修饰的变量一样使用第一次赋值的值.

# 回归主题 --> `copy` / `strong`
借用网上众多深拷贝,浅拷贝之说:
> 1. 深拷贝: 开辟一块新内存地址,使用指针指向这块地址.对原来的对象引用计数不变.
> 2. 浅拷贝: 对原内容增加一个指针指向,可以理解成引用计数器+1.
> 3. 在使用`copy`修饰时,深拷贝还是浅拷贝,这个和被copy的对象是什么类型有关:[借大佬图](https://www.cnblogs.com/beckwang0912/p/7212075.html)
![拷贝图](1118933-20170720203748536-592203434.png)

**水平有限,内容若有不妥,请斧正!!!**

