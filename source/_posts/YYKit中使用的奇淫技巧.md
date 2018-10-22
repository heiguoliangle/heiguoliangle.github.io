---
title: YYKit中使用的奇淫技巧
layout: post
date: 2018-01-23 13:28:03
comments: true
tags: [OC]
keywords: YYKit
---



# 变量的修饰 
1. @package 本包内使用，跨包不可以 `YYTextLayout`
2. @protected 该类和所有子类中的方法可以直接访问这样的变量。
3. @private 该类中的方法可以访问，子类不可以访问。
4. @public   可以被所有的类访问
5. `UNAVAILABLE_ATTRIBUTE`告诉编译器该方法不可用，如果强行调用编译器会提示错误。比如某个类在构造的时候不想直接通过init来初始化，只能通过特定的初始化方法，就可以将init方法标记为unavailable。 
6. `NS_DESIGNATED_INITIALIZER` 告诉编译器使用的构造方法最终会调用的是该方法,但是这个方法实现时候也需要调用父类的designated_initiallizer方法,
	
```
// 实现 UIView 的 designed initializer
- (instancetype)initWithCoder:(NSCoder *)aDecoder {
    return [self initWithVideoID:@0];
}

// 实现 UIView 的 designed initializer
- (instancetype)initWithFrame:(CGRect)frame {
    return [self initWithVideoID:@0];
}

// 实现自己的 designed initializer
- (instancetype)initWithVideoID:(NSNumber *)videoID {
    // 这里在调用 super 的初始化方法时，就不能调用 init，因为 init 不是 UIView 的 designed initializer
    self = [super initWithFrame:CGRectZero];
    if (self) {
        self.videoID = videoID;
        [self setupUI];
    }
    return self;
}
```

# 关键字
1. `__VA_ARGS__` 
		
		例如 `#define debug(...) printf(__VA_ARGS__)`,使用保留名 `__VA_ARGS__` 把参数传递给宏
2. `__FLT_MAX__` float 的最大值
3. `__kindof`	规定参数为UITableViewCell这个类或者其子类。比如说一个NSArray < UIView* >*,如果不加__kindof，这个数组只能有UIView,即便是其子类也不行。而加了的话，传入UIImageView或者UIButton之类的不会有问题。

		`__typeof__(var)` 是gcc对C语言的一个扩展保留字，用于声明变量类型,var可以是数据类型（int， char*..),也可以是变量表达式。
```
__typeof__(int *) x; //It   is   equivalent   to  'int  *x';
```
4. `inline` 内联函数,
5. `round` 四舍五入函数, 例如 : `round(1.499900) is 1.000000`,取整数
6. `fabs`绝对值 求浮点数x的绝对值,计算|x|, 当x不为负时返回 x，否则返回 -x
7. `extern` \ `static`,使用static修饰，该函数只能被当前源文件中的其它函数调用。使用extern修饰，该函数可以被任何源文件中的其它函数调用。
8. `NS_ASSUME_NONNULL_BEGIN` `NS_ASSUME_NONNULL_END`,这两个关键词中间所有的指针类型的参数都不能为空
9. `FOUNDATION_EXPORT`等同于 `#define`
	
```
在.h中这么定义,
FOUNDATION_EXPORT NSString * const kMyConstantString; 
在.m中实现
NSString * const kMyConstantString = @"Hello";
等同于 `#define kMyConstantString @"Hello"` 
使用时候可以 (stringInstance == MyFirstConstant)而define则使用的是这种.([stringInstance isEqualToString:MyFirstConstant])
```
	
# 方法
## CoreGraphics方法

1. `CGRectUnion` 返回能过包含两个矩形的最小矩形
3. `CGContextSaveGState` 把之前对上下文的操作拷贝一份推到堆栈上
4. `CGContextRestoreGState` 把之前拷贝一份推到堆栈上的上下文取出来
4. `CGContextSetFillColorWithColor` 定制要填充路径的颜色
5. `CGContextSetStrokeColorWithColor` 定制要描边路径的颜色
6. `CGContextSetLineWidth`
7. `CGContextSetTextDrawingMode` 每个符号的绘制方式,绘制文字的方式
8. `CGContextAddEllipseInRect` 画椭圆
9. `CGContextSetTextPosition` 需要绘制文本的位置
10. `CGContextSetLineDash` 设置图形上下文中的虚线的模式。

## C语言函数

1. `calloc`在内存动态存储区中分配n块长度为“size”字节的连续区域。函数的返回值为该区域的首地址。(类型说明符*)用于强制类型转换。calloc函数与malloc 函数的区别仅在于一次可以分配n块区域。例如： ps=(struet stu*) calloc(2,sizeof (struct stu)); 其中的sizeof(struct stu)是求stu的结构长度。因此该语句的意思是：按stu的长度分配2块连续区域，强制转换为stu类型，并把其首地址赋予指针变量ps。
2. `malloc` 它允许从空间内存池中分配内存,malloc()的参数是一个指定所需字节数的整数.
例如:P=(int*)malloc(n*sizeof(int));
  colloc与malloc类似,但是主要的区别是存储在已分配的内存空间中的值默认为0,使用malloc时,已分配的内存中可以是任意的值.
  colloc需要两个参数,第一个是需要分配内存的变量的个数,第二个是每个变量的大小.
例如:P=(int*)colloc(n,colloc(int));


# 其他问题
## 忽略警告

```
#pragma clang diagnostic push
#pragma clang diagnostic ignored "-Wcovered-switch-default"
switch (style) {
    case UITableViewCellStyleDefault:
    case UITableViewCellStyleValue1:
    case UITableViewCellStyleValue2:
    case UITableViewCellStyleSubtitle:
        // ...
    default:
        return;
}
#pragma clang diagnostic pop
```







