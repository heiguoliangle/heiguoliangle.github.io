---
title: hook方法
date: 2018-01-24 18:54:57
tags: [reverse]
---


# CydiaSubstrate Hook
> CydiaSubstrate，iOS7越狱之前名为 MobileSubstrate（简称为MS或MS框架），作者为大名鼎鼎的Jay Freeman(saurik)。

`MobileSubstrate` 这个工具是`control`的默认依赖库,包含了`MobileHooker`,`MobileLoader`,`Safe mode`.
`MobileHooker` 对 C OC 均有效.包含了`MSHookMessageEx`,`MSHookFunction`两个函数,其中`MSHookMessageEx`是对OC函数,实际采用的是runtime机制.
`MSHookFunction`针对C/C++函数,对于C函数是在函数的开头修改了汇编指令，使其跳转到新的实现，执行完成后再返回执行原指令.

## MSHookMessageEx

```
void MSHookMessageEx(Class _class, SEL message, IMP hook, IMP *old);
```
> _class需要hook的oc函数的类名;message 需要hook的oc的函数sel;hook新的hook函数的具体实现的imp;*old旧的函数的地址,官方示例:


```
NSString *(*oldDescription)(id self, SEL _cmd);

// implicit self and _cmd are explicit with IMP ABI
NSString *newDescription(id self, SEL _cmd) {
    NSString *description = (*oldDescription)(self, _cmd);
    description = [description stringByAppendingString:@"!"];
    return description;
}

MSHookMessageEx(
    [NSObject class], @selector(description),
    &newDescription, &oldDescription
);
```

[网上教程](https://www.jianshu.com/p/3479f9632a6f)



