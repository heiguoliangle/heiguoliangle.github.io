---
title: runtime普及
date: 2018-02-09 10:58:07
update: 2018-02-23 14:38:46
tags: [oc]
---

# runtime基本知识
每次都有人问,什么是`id`,`class`,`object`,`ivar`...这里做一个总结啦..
## class

```
/// An opaque type that represents an Objective-C class.
typedef struct objc_class *Class;
```
`class`是`objc_class`这个结构体的指针

## id

```
typedef struct objc_object *id;
```
`id`是`objc_object`这个结构体的指针
## objc_object

```
/// Represents an instance of a class.
struct objc_object {
    Class isa  OBJC_ISA_AVAILABILITY;
};
```

## objc_class

```
struct objc_class : objc_object {
    // Class ISA;
    Class superclass;
    cache_t cache;             // formerly cache pointer and vtable
    class_data_bits_t bits;    // class_rw_t * plus custom rr/alloc flags // 实例方法链表

    class_rw_t *data() { // 主要的运行数据
        return bits.data();
    }
   ...下面一大堆方法          
```

## ivar

```
/// An opaque type that represents an instance variable.
typedef struct objc_ivar *Ivar;
```
ivar 是指向objc_ivar结构体的指针.
## objc_ivar

```
struct objc_ivar {
    char *ivar_name                                          
    char *ivar_type                                         
    int ivar_offset                                          
    int space                                                
} 
```
`objc_ivar` 这个是记录属性的名字,类型,地址偏移量,大小

> 综合以上可以得出:
![借用霜神大图](23_6.png)

## 总结
> 通过上面这几个定义,可以理解成:
> 1. `class` 只是一个指向 `objc_class`的指针
> 2. `objc_class` 是继承至`objc_object`的结构体,除了继承了父类的`    // Class ISA;`成员变量,还增加了其他的成员变量
> 3. `objc_object` 一个只有`Class ISA;`成员变量的结构体**这个是一个基类**

例如当声明一个`Person`实例对象时,实际上是创建了一个指向`objc_class`结构体的指针.

## 更新
1. `objc_object` 被typedef成为id类型.
2. `objc_class`继承自objc_object,这里包含了isa_t类型的结构体isa,还有父类指针(superclass),方法缓冲(cache),实例方法链表(bits).
3. `object` 类和NSObject类都包含一个`objc_class`的isa.

### 类,对象方法调用
- 当一个对象的实例方法被调用的时候,会通过isa找到对应的类,然后在该类的class_data_bits_t(即bits)中去查找方法.class_data_bits_t(即bits)是指向了类对象的数据区域,在这里查找相应方法的对应实现.
- 当调用的是类方法时,引入了元类的概念(meta-class).
- 对象的实例方法调用时,通过对象的isa在类获取方法实现,类对象的类方法调用时,通过类的isa获取**元类**中的方法实现.
- 在`meta-class`中,存储了一个类中所有的类方法.每个类都有一个单独的`meta-class`.

![对象,类,元类之间的关系](23_7.png)
*借用霜神的描述*
实线是superclass的指针,虚线是isa指针
1. root class 实际就是NSObject,NSObject没有超类,所以他的superclass为nil
2. 每一个Class都有一个唯一指向的Metaclass.
3. rootclass(meta)的superclass指向rootclass(class),即NSObject,形成回路.
4. 每一个metaclass的isa都指向rootClass(meta).
5. **类对象和元类对象都是唯一的,对象可以在运行时创建无数个.在main函数之前,类对象和元类对象已经被创建出来**



# 访问一个类的ivar

首先在回到`objc_class`来,因为这`objc_class`中才能够拿到ivar.
`objc_class`中有一个`    class_rw_t *data() { `的成员变量,这里保存的是// 主要的运行数据,他是`class_rw_t`这个类型的:

```
struct class_rw_t {
    uint32_t flags;
    uint32_t version;

    const class_ro_t *ro;//程序运行时对程序二进制读取的副本数据

    union {
        method_list_t **method_lists;  // RW_METHOD_ARRAY == 1
        method_list_t *method_list;    // RW_METHOD_ARRAY == 0
    };
    struct chained_property_list *properties;
    const protocol_list_t ** protocols;

    Class firstSubclass;
    Class nextSiblingClass;

    char *demangledName;

    void setFlags(uint32_t set) 
    {
        OSAtomicOr32Barrier(set, &flags);
    }

    void clearFlags(uint32_t clear) 
    {
        OSAtomicXor32Barrier(clear, &flags);
    }

    // set and clear must not overlap
    void changeFlags(uint32_t set, uint32_t clear) 
    {
        assert((set & clear) == 0);

        uint32_t oldf, newf;
        do {
            oldf = flags;
            newf = (oldf | set) & ~clear;
        } while (!OSAtomicCompareAndSwap32Barrier(oldf, newf, (volatile int32_t *)&flags));
    }
};

```

在`class_rw_t`这个结构体中,有一个`class_ro_t`这里放的是运行数据的副本.

```
struct class_ro_t {
    uint32_t flags;
    uint32_t instanceStart;
    uint32_t instanceSize;
#ifdef __LP64__
    uint32_t reserved;
#endif

    const uint8_t * ivarLayout;
    
    const char * name;
    const method_list_t * baseMethods;
    const protocol_list_t * baseProtocols;
    const ivar_list_t * ivars; // 存放ivar指针的列表

    const uint8_t * weakIvarLayout;
    const property_list_t *baseProperties;
};
```

这里有一个`ivar_list_t`,这里存放的就是当前类中所有成员变量的ivar.


```
struct ivar_list_t {
    uint32_t entsize;
    uint32_t count;
    ivar_t first; //表示每一个ivar
};
```
这里的`ivar_t`就是每个ivar


```
struct ivar_t {
    int32_t *offset;
    const char *name;
    const char *type;
    // alignment is sometimes -1; use alignment() instead
    uint32_t alignment_raw;
    uint32_t size;

    uint32_t alignment() {
        if (alignment_raw == ~(uint32_t)0) return 1U << WORD_SHIFT;
        return 1 << alignment_raw;
    }
};
```

通过以上的结论,想要访问`ivar`,需要多次的指针转义:

`
class -> class.(class_rw_t *data()) -> class.(class_rw_t *data()).(class_ro_t *ro) -> class.(class_rw_t *data()).(class_ro_t *ro).(ivar_list_t * ivars) -> class.(class_rw_t *data()).(class_ro_t *ro).(ivar_list_t * ivars).first[n]
`
## 使用简单案例试验

```
#import <Foundation/Foundation.h>

typedef void(^MyBlock)(void);

@interface MyObject : NSObject
@property (nonatomic) NSUInteger haha;
@property (nonatomic, copy) MyBlock block;

- (void)inits;

@end

@implementation MyObject
- (void)inits
{
    self.block = ^{
        _haha = 5;
    };
}
@end
int main(int argc, char * argv[]) {
    
    MyObject *object = [MyObject new];
    [object inits];
}
```

将代码转成cpp,整理后如下:

```
typedef void(*MyBlock)(void);


#ifndef _REWRITER_typedef_MyObject
#define _REWRITER_typedef_MyObject
typedef struct objc_object MyObject;
typedef struct {} _objc_exc_MyObject;
#endif

extern "C" unsigned long OBJC_IVAR_$_MyObject$_haha;
extern "C" unsigned long OBJC_IVAR_$_MyObject$_block;
struct MyObject_IMPL {
	struct NSObject_IMPL NSObject_IVARS;
	NSUInteger _haha;
	MyBlock _block;
};

// @property (nonatomic) NSUInteger haha;
// @property (nonatomic, copy) MyBlock block;

// - (void)inits;

/* @end */


// @implementation MyObject

struct __MyObject__inits_block_impl_0 {
  struct __block_impl impl;
  struct __MyObject__inits_block_desc_0* Desc;
  MyObject *self;
  __MyObject__inits_block_impl_0(void *fp, struct __MyObject__inits_block_desc_0 *desc, MyObject *_self, int flags=0) : self(_self) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
static void __MyObject__inits_block_func_0(struct __MyObject__inits_block_impl_0 *__cself) {
  MyObject *self = __cself->self; // bound by copy

        (*(NSUInteger *)((char *)self + OBJC_IVAR_$_MyObject$_haha)) = 5;
    }
static void __MyObject__inits_block_copy_0(struct __MyObject__inits_block_impl_0*dst, struct __MyObject__inits_block_impl_0*src) {_Block_object_assign((void*)&dst->self, (void*)src->self, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static void __MyObject__inits_block_dispose_0(struct __MyObject__inits_block_impl_0*src) {_Block_object_dispose((void*)src->self, 3/*BLOCK_FIELD_IS_OBJECT*/);}

static struct __MyObject__inits_block_desc_0 {
  size_t reserved;
  size_t Block_size;
  void (*copy)(struct __MyObject__inits_block_impl_0*, struct __MyObject__inits_block_impl_0*);
  void (*dispose)(struct __MyObject__inits_block_impl_0*);
} __MyObject__inits_block_desc_0_DATA = { 0, sizeof(struct __MyObject__inits_block_impl_0), __MyObject__inits_block_copy_0, __MyObject__inits_block_dispose_0};

static void _I_MyObject_inits(MyObject * self, SEL _cmd) {
    ((void (*)(id, SEL, MyBlock))(void *)objc_msgSend)((id)self, sel_registerName("setBlock:"), ((void (*)())&__MyObject__inits_block_impl_0((void *)__MyObject__inits_block_func_0, &__MyObject__inits_block_desc_0_DATA, self, 570425344)));
}

static NSUInteger _I_MyObject_haha(MyObject * self, SEL _cmd) { return (*(NSUInteger *)((char *)self + OBJC_IVAR_$_MyObject$_haha)); }
static void _I_MyObject_setHaha_(MyObject * self, SEL _cmd, NSUInteger haha) { (*(NSUInteger *)((char *)self + OBJC_IVAR_$_MyObject$_haha)) = haha; }

static void(* _I_MyObject_block(MyObject * self, SEL _cmd) )(){ return (*(MyBlock *)((char *)self + OBJC_IVAR_$_MyObject$_block)); }
extern "C" __declspec(dllimport) void objc_setProperty (id, SEL, long, id, bool, bool);

static void _I_MyObject_setBlock_(MyObject * self, SEL _cmd, MyBlock block) { objc_setProperty (self, _cmd, __OFFSETOFIVAR__(struct MyObject, _block), (id)block, 0, 1); }
// @end
int main(int argc, char * argv[]) {

    MyObject *object = ((MyObject *(*)(id, SEL))(void *)objc_msgSend)((id)objc_getClass("MyObject"), sel_registerName("new"));
    ((void (*)(id, SEL))(void *)objc_msgSend)((id)object, sel_registerName("inits"));
}
```

- 从上面的代码可以看到,对于每一个成员变量,都会有一个对应的全局变量:

```
extern "C" unsigned long OBJC_IVAR_$_MyObject$_haha;
extern "C" unsigned long OBJC_IVAR_$_MyObject$_block;
```
- blcok_invoke对应的实现通过对象本事作为基地址+全局变量偏移对变量的ivar进行赋值的

```
static void __MyObject__inits_block_func_0(struct __MyObject__inits_block_impl_0 *__cself) {
  MyObject *self = __cself->self; // bound by copy

        (*(NSUInteger *)((char *)self + OBJC_IVAR_$_MyObject$_haha)) = 5;
    }
```
- block的构造函数确实捕捉了`self`

```
struct __MyObject__inits_block_impl_0 {
  struct __block_impl impl;
  struct __MyObject__inits_block_desc_0* Desc;
  MyObject *self;
  __MyObject__inits_block_impl_0(void *fp, struct __MyObject__inits_block_desc_0 *desc, MyObject *_self, int flags=0) : self(_self) {
    impl.isa = &_NSConcreteStackBlock;
    impl.Flags = flags;
    impl.FuncPtr = fp;
    Desc = desc;
  }
};
```


# objc_msgSend
## - (id)perform:(SEL)aSelector with:anObject
源码如下:

```
- (id)perform:(SEL)aSelector with:anObject
{
	if (aSelector)
		return ((id(*)(id, SEL, id))objc_msgSend)(self, aSelector, anObject); 
	else
		return [self error:_errBadSel, sel_getName(_cmd), aSelector];
}
```
在调用`objc_msgSend`时候,需要提前指定好返回值,调用者,方法名,传入参数:
> ((id(*)(id, SEL, id)) //(id(*)返回值;(id, SEL, id)调用者,方法名,传入参数类型
> objc_msgSend) // 调用的方法为`objc_msgSend`
> (self, aSelector, anObject) // 调用者,方法名,传入参数

# 参考链接
 [霜神博客](https://halfrost.com/objc_runtime_isa_class/)
 [YY大神](https://blog.ibireme.com/2013/11/25/objc-object/)

