---
title: iOS中常见集合
date: 2018-08-08 15:39:51
tags:
---



> iOS开发中说道集合,你能想到多少呢?
> `NSArray`,`NSDictionary`,`NSCache`,`NSSet`,`NSHashTable`,`NSMapTable`
	以及它们的可变集.

<text style='color:green;font:14;text-align:center;text-decoration:underline;display:block'> 这里就开始看看它们的具体区别 </text>

# 1. NSCache

先看看官网的一段解释:

```
The NSCache class incorporates various auto-eviction policies, which ensure that a cache doesn’t use too much of the system’s memory. If memory is needed by other applications, these policies remove some items from the cache, minimizing its memory footprint.
You can add, remove, and query items in the cache from different threads without having to lock the cache yourself.
Unlike an NSMutableDictionary object, a cache does not copy the key objects that are put into it.
```
这个大致意思:
1. NSCache类包各种内存清理机制，确保缓存不占用系统内存太多,如果接到内存警告或者其他内存问题时候,会清除占用的多余内存.
2. 不会像 `NSMutableDictionary`一样,对`key`进行copy.
3. 这个是线程安全的.

这里在使用时候需要注意,当内存吃紧时候,NSCache所拥有的值被释放,再次使用时候需要重新进行计算或取值.
## NSCache属性,方法

### 属性

```
/// 当前cache 的名称,可以理解为标识符.
@property (copy) NSString *name;
/// 当前cache 的代理, 
// 1. 当接受到内存警告; 
// 2. 手动调用`removeObjectForKey` | `removeAllObjects` 会进入代理方法.
// 3. 设置属性`countLimit`后,缓冲数量大于所设置的,会按照FIFO规则进行调用.
// 4. 设置属性`totalCostLimit`后,超出时候,会调用.
@property (nullable, assign) id<NSCacheDelegate> delegate;
/// 可以存储的最大内存字节数,并非严格限制 limits are imprecise/not strict
@property NSUInteger totalCostLimit; 
/// 设置对象数量 .limits are imprecise/not strict
@property NSUInteger countLimit;
/// 如果设置YES,value为实现了`NSDiscardableContent`协议的对象,这回进入这个协议中,并将他移除出去.反之不会 丢弃.
@property BOOL evictsObjectsWithDiscardedContent;
```

### 方法

```
/// 设置和移除key value 方法
- (nullable ObjectType)objectForKey:(KeyType)key;
- (void)setObject:(ObjectType)obj forKey:(KeyType)key; // 0 cost
- (void)setObject:(ObjectType)obj forKey:(KeyType)key cost:(NSUInteger)g;
- (void)removeObjectForKey:(KeyType)key;

- (void)removeAllObjects;

/// NSCacheDelegate <NSObject> 将要被清除时候的方法.
- (void)cache:(NSCache *)cache willEvictObject:(id)obj;

```
## NSCache的使用.
在使用过程中,需要有个注意点,这里存储的值有可能会释放,在取值时候需要谨慎使用.
这里参考SD中的实现来说:

```

@interface SDMemoryCache <KeyType, ObjectType> : NSCache <KeyType, ObjectType>

@end


@interface SDMemoryCache <KeyType, ObjectType> ()

@property (nonatomic, strong, nonnull) NSMapTable<KeyType, ObjectType> *weakCache; // strong-weak cache
@property (nonatomic, strong, nonnull) dispatch_semaphore_t weakCacheLock; // a lock to keep the access to `weakCache` thread-safe

@end

@implementation SDMemoryCache
- (void)dealloc {
    [[NSNotificationCenter defaultCenter] removeObserver:self name:UIApplicationDidReceiveMemoryWarningNotification object:nil];
}

- (instancetype)init {
    self = [super init];
    if (self) {
          self.weakCache = [[NSMapTable alloc] initWithKeyOptions:NSPointerFunctionsStrongMemory valueOptions:NSPointerFunctionsWeakMemory capacity:0];
        self.weakCacheLock = dispatch_semaphore_create(1);
        [[NSNotificationCenter defaultCenter] addObserver:self                  selector:@selector(didReceiveMemoryWarning:)              name:UIApplicationDidReceiveMemoryWarningNotification            object:nil];
    }
    return self;
}
- (void)didReceiveMemoryWarning:(NSNotification *)notification {
    [super removeAllObjects];
}
- (void)setObject:(id)obj forKey:(id)key cost:(NSUInteger)g {
    [super setObject:obj forKey:key cost:g];
    if (key && obj) {
        // Store weak cache
        LOCK(self.weakCacheLock);
        [self.weakCache setObject:obj forKey:key];
        UNLOCK(self.weakCacheLock);
    }
}

- (id)objectForKey:(id)key {
    id obj = [super objectForKey:key];
    if (key && !obj) {
        LOCK(self.weakCacheLock);
        obj = [self.weakCache objectForKey:key];
        UNLOCK(self.weakCacheLock);
        if (obj) {
            NSUInteger cost = 0;
            if ([obj isKindOfClass:[UIImage class]]) {
                cost = SDCacheCostForImage(obj);
            }
            [super setObject:obj forKey:key cost:cost];
        }
    }
    return obj;
}

- (void)removeObjectForKey:(id)key {
    [super removeObjectForKey:key];
    if (key) {
        LOCK(self.weakCacheLock);
        [self.weakCache removeObjectForKey:key];
        UNLOCK(self.weakCacheLock);
    }
}

- (void)removeAllObjects {
    [super removeAllObjects];
    LOCK(self.weakCacheLock);
    [self.weakCache removeAllObjects];
    UNLOCK(self.weakCacheLock);
}
@end

```

简单的大致一看,SDMemoryCache 继承自 NSCache ,内部持有了一个`NSMapTable` 类型的weakCache缓冲,负责类对外界需要缓冲的对象进行弱引用.
这里我们的关注点只要是释放问题.
1. 在初始化时候接受了一个`UIApplicationDidReceiveMemoryWarningNotification`通知,来确保收到内存警告时候能够及时的将cache清除.
2. 在收到内存警告时候并未将对象持有的weakCache清除,只是将`[super removeAllObjects];`而已,将自己持有的缓冲对象清空了.这样能够保证不产生循环引用的情况下载内存中保持着一个备选方案,避免了在cache被清除时候每次从磁盘重新缓冲的浪费.
3. 在`removeAllObjects` 时候对weakCache同时也清除了.

## NSCache 总结
相比于其他的容器,NSCache的有点和缺点很明显:
优点: 
能够一直保证内存处于平缓转态,不会爆增爆减.
缺点:
在使用时候不知道key 对应的 value 是否已被删除,需要时刻进行处理.

# 2. NSMapTable
再来看看官网:

```
A collection similar to a dictionary, but with a broader range of available memory semantics.
The map table is modeled after NSDictionary with the following differences:
Keys and/or values are optionally held “weakly” such that entries are removed when one of the objects is reclaimed.
Its keys or values may be copied on input or may use pointer identity for equality and hashing.
It can contain arbitrary pointers (its contents are not constrained to being objects).
```
简单来说:
1. 类似于dictionary,但是拥有更多的内存可控性.
2. Keys and/or values 可以是`weak`或`strong` 引用的,如果源地址被回收了,该条目也会相应的删除.
3. 他的包含内容不局限与对象,可以是指针类型.

> 与dictionary相比,最大的区别就是对key及value内存的持有关系,可以通过`NSPointerFunctionsOptions`进行各个中关系的匹配.

## NSMapTable属性及方法(仅显示iPhone平台的)
### 属性


```
/// 若创建方式是`initWithKeyPointerFunctions`形式,则返回初始时的对象.
@property (readonly, copy) NSPointerFunctions *keyPointerFunctions;
@property (readonly, copy) NSPointerFunctions *valuePointerFunctions;
/// 这个count不是当前NSMapTable所持有的键值对,而是总共加过的键值对数量.
@property (readonly) NSUInteger count;
```
### 方法

#### 初始化方法
```
- (instancetype)initWithKeyOptions:(NSPointerFunctionsOptions)keyOptions valueOptions:(NSPointerFunctionsOptions)valueOptions capacity:(NSUInteger)initialCapacity NS_DESIGNATED_INITIALIZER;
- (instancetype)initWithKeyPointerFunctions:(NSPointerFunctions *)keyFunctions valuePointerFunctions:(NSPointerFunctions *)valueFunctions capacity:(NSUInteger)initialCapacity NS_DESIGNATED_INITIALIZER;
```
这里有两个初始化方法.
1. 使用内存管理枚举进行创建,可以指定key , value 的内存管理方式.
2. 使NSPointerFunctions 对象进行创建,可以针对key , value 进行详细的内存管理.

平时使用第一种完全没有问题.其中枚举`NSPointerFunctionsOptions`,主要分为三大类:

1. 内存管理
	- NSPointerFunctionsStrongMemory：默认值，在 CG 和 MRC 下强引用成员
	- NSPointerFunctionsZeroingWeakMemory：已废弃，在 GC 下，弱引用指针，防止悬挂指针
	- NSPointerFunctionsMallocMemory 与 NSPointerFunctionsMachVirtualMemory： 用于 Mach 的虚拟内存管理
	- NSPointerFunctionsWeakMemory：在 CG 或者 ARC 下，弱引用成员
2. 特性，用于标明对象判等方式
	- NSPointerFunctionsObjectPersonality：hash、isEqual、对象描述
	- NSPointerFunctionsOpaquePersonality：pointer 的 hash 、直接判等
	- NSPointerFunctionsObjectPointerPersonality：pointer 的 hash、直接判等、对象描述
	- NSPointerFunctionsCStringPersonality：string 的 hash、strcmp 函数、UTF-8 编码方式的描述
	- NSPointerFunctionsStructPersonality：内存 hash、memcmp 函数
	- NSPointerFunctionsIntegerPersonality：值的 hash
3. 内存标识
	- NSPointerFunctionsCopyIn：根据第二类的选择，来具体处理。
如果是 NSPointerFunctionsObjectPersonality，则根据 NSCopying 来拷贝。

这里可以见常用的 `NSPointerFunctionsWeakMemory`,`NSPointerFunctionsCopyIn`,`NSPointerFunctionsStrongMemory`等枚举值.
以上这两个是`NS_DESIGNATED_INITIALIZER`方法,剩下一下扩展初始化方法.

#### 对象方法

```

- (nullable ObjectType)objectForKey:(nullable KeyType)aKey;

- (void)removeObjectForKey:(nullable KeyType)aKey;
- (void)setObject:(nullable ObjectType)anObject forKey:(nullable KeyType)aKey;   // add/replace value (CFDictionarySetValue, NSMapInsert)
- (NSEnumerator<KeyType> *)keyEnumerator;
- (nullable NSEnumerator<ObjectType> *)objectEnumerator;

- (void)removeAllObjects;

- (NSDictionary<KeyType, ObjectType> *)dictionaryRepresentation;  // create a dictionary of contents
```
这里基本上和NSDictory是没有区别的.其中提供了一个获取所有key 和value 的方法,返回值是一个`NSEnumerator`类型的对象.这里在调用了`NSEnumerator`的`nextObject`方法后,返回的`allObjects` 属性将会是按照enumer的顺序来单个展示.
还有一个将mapTable转换成`NSDictionary`的实例方法.

## NSMapTable的使用
参考一下官方的例子:

```
 NSMapTable *mt = [[NSMapTable alloc] 
                            initWithKeyOptions: (NSPointerFunctionsStrongMemory | NSPointerFunctionsCStringPersonality | NSPointerFunctionsCopyIn)
                            valueOptions:(NSPointerFunctionsWeakMemory   | NSPointerFunctionsObjectPersonality)
                            capacity:0];
       ...
       // given a C string, look it up and, if not found, create and save the NSString version in the table
       const char *cString = ...;
       NSString *result = [mt objectForKey:(id)cString];
       if (!result) {
            result = [NSString stringWithCString:cString];
            [mt setObject:result forKey:(id)cString];
        }
        return result;
```
这里是创建了一个key 是strong类型的强用用,并且使用了NSCoping的字符串的选项,value 是weak引用的对象,长度暂为0(后期自动变化)的mapTable.

## 总结
NSMapTable 与 NSDictory最大的区别就是对key和value 的内存管理,在合理选择情况下,可以规避掉循环引用关系;也可以在对象内存释放后同时对容器进行同步更新.


# NSSet
## NSSet 和 NSArray
NSSet : A static unordered collection of unique objects.
NSArray : A static ordered collection of objects.

通过官网的描述可简单显示 : 
1. set 是一个无序并且是唯一的对象的集合;array 是一个有序集合,但是可以包含相同对象.
2. 在对某个元素进行搜索时候,NSSet是高于NSArray 的,因为NSSet 是hash算法,存储一个hash值,定位查找,而NSArray这是遍历;NSSet是无序的,不能使用下标进行查找

## NSSet 增加方法.


```
/// 至少有一个set 在原set中存在就返回YES
- (BOOL)intersectsSet:(NSSet<ObjectType> *)otherSet;
/// 完全匹配返回YES
- (BOOL)isEqualToSet:(NSSet<ObjectType> *)otherSet;
/// 原set 是otherSet 中的一部分.
- (BOOL)isSubsetOfSet:(NSSet<ObjectType> *)otherSet;
```






