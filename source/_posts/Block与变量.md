---
title: Block与变量
date: 2018-03-18 18:10:57
tags:
---
# block 的定义


首先看一下block的定义(具体定义还得找找找了)
```
struct Block_layout {  
    void *isa;
    int flags;
    int reserved;
    void (*invoke)(void *, ...);
    struct Block_descriptor *descriptor;
    /* Imported variables. */
};

struct Block_descriptor {  
    unsigned long int reserved;
    unsigned long int size;
    void (*copy)(void *dst, void *src);
    void (*dispose)(void *);
};
```

其中isa 这个可已在 `usr/include/Block.h` 找到,主要是`_NSConcreteGlobalBlock`和`_NSConcreteStackBlock`,还有`_NSConcreteMallocBlock`这三种.

# block 如何捕获外部变量
## 外部变量的分类
 1. 全局变量
 2. 全局静态变量
 3. 静态变量
 4. 普通变量(在函数定义的变量)
 
## 外部变量的捕获
- 测试代码1(在普通变量不在block中修改) .

```
#import <Foundation/Foundation.h>
int global_i = 1;
static int static_global_j = 2;

int main(int argc, char * argv[]) {
    
    
    static int static_k = 3;
    int var = 4;
    void (^myBlock)(void) = ^{
        global_i ++;
        static_global_j ++;
        static_k ++;
         NSLog(@"在block内部 打印: global_i = : %d . static_global_j = %d . static_k = %d . var = %d ",global_i,static_global_j,static_k,var);
    };
    global_i ++;
    static_global_j ++;
    static_k ++;
    var ++;
     NSLog(@"在block外部 打印: global_i = : %d . static_global_j = %d . static_k = %d . var = %d ",global_i,static_global_j,static_k,var);
    myBlock();
    return 0 ;
}
```

运行结果: 

```
2018-03-18 18:25:42.965883+0800 BlockTest[366:1601746] 在block外部 打印: global_i = : 2 . static_global_j = 3 . static_k = 4 . var = 5 
2018-03-18 18:25:42.968159+0800 BlockTest[366:1601746] 在block内部 打印: global_i = : 3 . static_global_j = 4 . static_k = 5 . var = 4 
```
总结: 

1. 全局变量,全局静态变量,静态变量均被改变了值.
2. 普通变量的值并未改变.

内部是如何实现的呢,看看cpp.
使用`clang -rewrite-objc main.m` 将代码转换成cpp

## 捕获变量的实现(未使用__block)

将代码转换成cpp后,文件会扩大很多吧,它会将你导入的头文件内容也写进去.出去头文件,看看代码实现如下:

```
int global_i = 1;
static int static_global_j = 2;


struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int *static_k;
  int var;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_static_k, int _var, int flags=0) : static_k(_static_k), var(_var) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int *static_k = __cself->static_k; // bound by copy
  int var = __cself->var; // bound by copy

        global_i ++;
        static_global_j ++;
        (*static_k) ++;

         NSLog((NSString *)&__NSConstantStringImpl__var_folders_zc_l488fsfj15x593g3x5d0xs1c0000gn_T_main_1ec1fe_mi_0,global_i,static_global_j,(*static_k),var);
    }

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0)};
int main(int argc, char * argv[]) {


    static int static_k = 3;
    int var = 4;
    void (*myBlock)(void) = ((void (*)())&__main_block_impl_0(
          (void *)__main_block_func_0, &__main_block_desc_0_DATA,
              &static_k,
                  var)
                             );
    
    global_i ++;
    static_global_j ++;
    static_k ++;
    var ++;

     NSLog((NSString *)&__NSConstantStringImpl__var_folders_zc_l488fsfj15x593g3x5d0xs1c0000gn_T_main_1ec1fe_mi_2,global_i,static_global_j,static_k,var);
    ((void (*)(__block_impl *))((__block_impl *)myBlock)->FuncPtr)((__block_impl *)myBlock);

    return 0 ;
}
```

根据结果来看内容.
1. 全局变量和全局静态变量的值改变了,这个是由于他们作用域很广,直接被block捕获,并且修改了值.
2. 静态变量的值改变,它已经被block捕获,而普通变量并未被捕获.

具体实现:  (由于是cpp代码,所以每一个`struct`其实对应的就是oc中的`Class`)
1. 首先从block的定义开始看:
	
	```
	void (*myBlock)(void) = ((void (*)())&__main_block_impl_0(
          (void *)__main_block_func_0, &__main_block_desc_0_DATA,
              &static_k,
                  var)
                             );
	```
定义block时候,实际是调用了一个`__main_block_impl_0` 函数,并且传入了`(void *)__main_block_func_0`,`&__main_block_desc_0_DATA`,`&static_k`,`var`这四个参数.

接着就看 `__main_block_impl_0` 方法.

2. `__main_block_impl_0`
	在cpp代码中可以看到这个类(可以理解为oc中的类)
	
	```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  int *static_k;
  int var;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, int *_static_k, int _var, int flags=0) : static_k(_static_k), var(_var) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
	```
	在调用方法时候,使用的第一个参数`void *fp` --> `impl.FuncPtr = fp;`,可以看出来是将`(void *)__main_block_func_0`这个参数赋值给block的实现方法,即这个block的实现应该是去看`__main_block_func_0`方法.
	看第二个参数是`struct __main_block_desc_0 *desc` --> `Desc = desc;`, 这里是将`&__main_block_desc_0_DATA`这个的地址赋值给了Desc这里包含了一写block的大小信息.
	看第三,四个参数是`int *_static_k, int _var`,这里个其实就是在block中捕获的两个外部变量,并且将他们作为了参数来使用.**其中`_static_k` 这里的取地址,而`_var`直接就是取值.**
	看第五个参数是`int flags=0` ,这里给一个默认值,可类似理解成swift中的default参数,外界可以不用传入.通过这个flag,可以将block在做copy操作时候_NSConcreteStackBlock 赋值到_NSConcreteMallocBlock.具体方法在`_Block_copy_internal`这个方法中(文档暂时没找到)
	之后还有`static_k(_static_k), var(_var)`,这个是cpp的代码风格,其实就是指定了`_static_k` 这个形参,并且将`_static_k`的值赋值给类的属性`static_k`,相当于是oc中的一个成员变量(暂做这么理解).
	当在block的调用之前,可以将block的初始化理解为:
	
	```
	impl.isa = &_NSConcreteStackBlock;
    impl.Flags = 0;
    impl.FuncPtr = (void *)__main_block_func_0;
    Desc = desc;
    *_statick_k = 4;
    var = 4
	```

3. `__main_block_func_0` block的具体实现.
	
	```
	static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  int *static_k = __cself->static_k; // bound by copy
  int var = __cself->var; // bound by copy

        global_i ++;
        static_global_j ++;
        (*static_k) ++;

         NSLog((NSString *)&__NSConstantStringImpl__var_folders_zc_l488fsfj15x593g3x5d0xs1c0000gn_T_main_1ec1fe_mi_0,global_i,static_global_j,(*static_k),var);
    }
	```
	这里通过默认给的注释可以看出,`bound by copy`,在调用block的实现的时候,是通过copy的方式将外部变量保存到block的实体类中的.同时这里好保证了自去copy block中使用到的外部变量,而不是全部的.
	这里将`static_k`,`var`取出,并进行 ++ 操作,由于`static_k`是取得指针,所以在外部进行++操作之后,在block中也会进行加一操作.而`var`只是值传递,不会进行操作.也就导致了在bloc中是不能够修改`var`的值.如果强行修改户报错`Variable is not assignable(missing __block type specifier)`

4. 小范围总结一下:
	想要在block中修改外部参数的值可以通过两种方式: 
	- 传递参数的内存地址到block
	- 修改参数的存储位置,比如改成global等.
5. 验证 `传递参数的内存地址到block`
	
	```
int main(int argc, char * argv[]) {
    NSMutableString * str = [[NSMutableString alloc]initWithString:@"hei"];
    void (^myBlock)(void) = ^{
        [str appendFormat:@"guo"];
        NSLog(@"在block内部 打印:%@",str);
    };
    
    NSLog(@"在block外部 打印:%@",str);
    myBlock();
    return 0 ;
}
	```
	打印结果:
	
	```
	2018-03-18 20:07:26.840293+0800 BlockTest[654:1664438] 在block外部 打印:hei
2018-03-18 20:07:26.842053+0800 BlockTest[654:1664438] 在block内部 打印:heiguo
	```
	
	换成cpp: 
	
```
struct __main_block_impl_0 {
  struct __block_impl impl;
  struct __main_block_desc_0* Desc;
  NSMutableString *str;
  __main_block_impl_0(void *fp, struct __main_block_desc_0 *desc, NSMutableString *_str, int flags=0) : str(_str) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __main_block_func_0(struct __main_block_impl_0 *__cself) {
  NSMutableString *str = __cself->str; // bound by copy

        ((void (*)(id, SEL, NSString *, ...))(void *)objc_msgSend)((id)str, sel_registerName("appendFormat:"), (NSString *)&__NSConstantStringImpl__var_folders_zc_l488fsfj15x593g3x5d0xs1c0000gn_T_main_c66468_mi_1);
        NSLog((NSString *)&__NSConstantStringImpl__var_folders_zc_l488fsfj15x593g3x5d0xs1c0000gn_T_main_c66468_mi_2,str);
    }
static void __main_block_copy_0(struct __main_block_impl_0*dst, struct __main_block_impl_0*src) {_Block_object_assign((void*)&dst->str, (void*)src->str, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static void __main_block_dispose_0(struct __main_block_impl_0*src) {_Block_object_dispose((void*)src->str, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static struct __main_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __main_block_impl_0*, struct __main_block_impl_0*);
  void (*dispose)(struct __main_block_impl_0*);
} __main_block_desc_0_DATA = { 0, sizeof(struct __main_block_impl_0), __main_block_copy_0, __main_block_dispose_0};
int main(int argc, char * argv[]) {
    NSMutableString * str = ((NSMutableString *(*)(id, SEL, NSString *))(void *)objc_msgSend)((id)((NSMutableString *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("NSMutableString"), sel_registerName("alloc")), sel_registerName("initWithString:"), (NSString *)&__NSConstantStringImpl__var_folders_zc_l488fsfj15x593g3x5d0xs1c0000gn_T_main_c66468_mi_0);
    void (*myBlock)(void) = ((void (*)())&__main_block_impl_0((void *)__main_block_func_0, &__main_block_desc_0_DATA, str, 570425344));
    NSLog((NSString *)&__NSConstantStringImpl__var_folders_zc_l488fsfj15x593g3x5d0xs1c0000gn_T_main_c66468_mi_3,str);
    ((void (*)(__block_impl *))((__block_impl *)myBlock)->FuncPtr)((__block_impl *)myBlock);
    return 0 ;
}

```
	
	
首先看结果:
在`__main_block_impl_0`中确实是捕获了地址:    NSMutableString *str;证明第一种说法.
	 其中可以看到这里多了两个方法`__main_block_copy_0`,`__main_block_dispose_0`,这两个方法在`usr/include/Block.h`文件中也是可以看到的.
	 
###  __main_block_copy_0与 __main_block_dispose_0


