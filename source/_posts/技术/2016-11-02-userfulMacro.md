---
layout: post
date: 2016-11-02 10:50:46
title: iOS的宏
category: 技术
tags: 资料
keywords: iOS
description: 整理最近几个在开发时有特定用处的宏
---
# NSParameterAssert
NSParameterAssert故名思义,用于检测参数,是一个封装了NSAssert的宏,定义如下:

```
#define NSParameterAssert(condition) NSAssert((condition), @"Invalid parameter not satisfying: %@", @#condition)
```

## code 

```

- (void)assertWithPara:(NSString *)str
{
    NSParameterAssert(str); //只需要一个参数,如果参数存在程序继续运行,如果参数为空,则程序停止打印日志
    //further code ...
}

```

## NSAssert

当然也可以用 ` NSAssert` ,通过条件表达式来断定是否属于Bug，满足条件返回`真值`，程序继续运行;如果返回`假值`，则抛出异常，并且可以自定义异常描述。

## code 

```
int a = 1;
//第一个参数是条件,如果第一个参数不满足条件,就会记录并打印后面的字符串
NSCAssert(a == 2, @"a must equal to 2"); 
```

# __has_include

用来检查Frameworks是否引入某个类，

如YYWebImage已经集成YYCache,如果导入过YYWebImage则无需重新导入YYCache

## code

```
#if __has_include(<YYCache/YYCache.h>)
#import <YYCache/YYMemoryCache.h>
#import <YYCache/YYDiskCache.h>
#import <YYCache/YYKVStorage.h>
#elif __has_include(<YYWebImage/YYCache.h>)
#import <YYWebImage/YYMemoryCache.h>
#import <YYWebImage/YYDiskCache.h>
#import <YYWebImage/YYKVStorage.h>
#else
#import "YYMemoryCache.h"
#import "YYDiskCache.h"
#import "YYKVStorage.h"
#endif
```

# NS_ASSUME_NONNULL_BEGIN/NS_ASSUME_NONNULL_END

接口中 nullable 的是少数,一般都为nonnull,为了防止写一大堆 nonnull，Foundation供了一对宏NS_ASSUME_NONNULL_BEGIN、NS_ASSUME_NONNULL_END，包在里面的对象默认加 nonnull 修饰符，如果是nullable的,只需要把 nullable 的指出来就行

## code 

```
NS_ASSUME_NONNULL_BEGIN
@interface HZUserDefaults : NSObject
...
+ (nullable instancetype)hz_objectForKey:(nullable NSString *)key;
...
@end
NS_ASSUME_NONNULL_END
```

# UNAVAILABLE_ATTRIBUTE

unavailable告诉编译器该方法失效.

在封装单例或初始化某个类前必须做一些事时,对一些方法禁用是非常不错的选择.

如果需要,还可以使用 ` __attribute__((unavailable("your tips")));` 这个宏来给出具体的提示

## code

```
- (instancetype)init UNAVAILABLE_ATTRIBUTE;

+ (instancetype)new __attribute__((unavailable("new方法不可用，请用initWithName:")));
```


