---
layout: post
date: 2017-12-31 14:25:00
title: 你真的会判断 _objc_msgForward_stret 吗
category: 技术
keywords: iOS
description: 从 JSPatch/Aspects 的代码里，学习探究 _objc_msgForward_stret 的判断
---

# 前言

>本文需要对消息转发机制有了解，建议阅读 [Objective-C 消息发送与转发机制原理](http://yulingtianxia.com/blog/2016/06/15/Objective-C-Message-Sending-and-Forwarding/)

恰巧在 8 月学习 Method Swizzling ，阅读了 Aspects 和 JSPatch 做方法替换的处理，注意到了我们这次介绍的主角 --`_objc_msgForward_stret `.
>[JSPatch](https://github.com/bang590/JSPatch) 目前的 Star 数已经破万，知名度可见一斑。面试时，也会经常被提及。[Aspects](https://github.com/steipete/Aspects)也是一个在 AOP 方面非常著名的库。


在消息转发时，我们根据方法返回值的类型，来决定 IMP 使用 `_objc_msgForward` 或者 `_objc_msgForward_stret`.

根据苹果的文档描述，使用 `_objc_msgForward_stret` 的肯定是一个结构体:
>Sends a message with a data-structure return value to an instance of a class.

然而，不同 CPU 架构下，判断 `_objc_msgForward_stret` 的规则也有差异。下面就来看看两个著名开源库的做法。

## JSPatch 的判断 

首先我们来看 JSPatch , 在 JPEngine.m 文件里的 overrideMethod 方法,是如何去判断是否使用 `_objc_msgForward_stret`:


    IMP msgForwardIMP = _objc_msgForward;
    #if !defined(__arm64__)
        if (typeDescription[0] == '{') {
            //In some cases that returns struct, we should use the '_stret' API:
            //http://sealiesoftware.com/blog/archive/2008/10/30/objc_explain_objc_msgSend_stret.html
            //NSMethodSignature knows the detail but has no API to return, we can only get the info from debugDescription.
            NSMethodSignature *methodSignature = [NSMethodSignature signatureWithObjCTypes:typeDescription];
            if ([methodSignature.debugDescription rangeOfString:@"is special struct return? YES"].location != NSNotFound) {
                msgForwardIMP = (IMP)_objc_msgForward_stret;
            }
        }
    #endif
    

上面的代码，第一判断在非 `arm64` 下，第二判断是否为 `union` 或者 `struct` (详见[Type Encodings](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100)  )。

最后，通过判断方法签名的 debugDescription 是不是包含特定字符串-`is special struct return? YES`，进而决定是否使用 `_objc_msgForward_stret` .可以说是一个非常 trick 的做法了.

关于 `Special Struct` ，JSPatch 作者自己也在 [JSPatch 实现原理详解](https://github.com/bang590/JSPatch/wiki/JSPatch-%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3) 中提到了原因.文章说明在非 arm64 下都会存在 `Special Struct` 这样的问题。而具体判断的规则，苹果并没有提供给我们，所以使用到了这样的方法进行判断也是无奈之举。

好在经过大量项目运行以来，证明这个方法还是靠谱的。


## Aspects 的判断

Aspects 同样也是一个非常有名的开源项目,查看 Aspects 中与 `_objc_msgForward_stret` 相关的 commit,作者对  `Special Struct` 的判断颇下功夫，前后修改了很多次。

最后的版本是这样的:

    static IMP aspect_getMsgForwardIMP(NSObject *self, SEL selector) {
    IMP msgForwardIMP = _objc_msgForward;
    #if !defined(__arm64__)
    // As an ugly internal runtime implementation detail in the 32bit runtime, we need to determine of the method we hook returns a struct or anything larger than id.
    // https://developer.apple.com/library/mac/documentation/DeveloperTools/Conceptual/LowLevelABI/000-Introduction/introduction.html
    // https://github.com/ReactiveCocoa/ReactiveCocoa/issues/783
    // http://infocenter.arm.com/help/topic/com.arm.doc.ihi0042e/IHI0042E_aapcs.pdf (Section 5.4)
    Method method = class_getInstanceMethod(self.class, selector);
    const char *encoding = method_getTypeEncoding(method);
    BOOL methodReturnsStructValue = encoding[0] == _C_STRUCT_B;
    if (methodReturnsStructValue) {
        @try {
            NSUInteger valueSize = 0;
            NSGetSizeAndAlignment(encoding, &valueSize, NULL);

            if (valueSize == 1 || valueSize == 2 || valueSize == 4 || valueSize == 8) {
                methodReturnsStructValue = NO;
            }
        } @catch (__unused NSException *e) {}
    }
    if (methodReturnsStructValue) {
        msgForwardIMP = (IMP)_objc_msgForward_stret;
    }
    #endif
     return msgForwardIMP;
    }

与 JSPatch 相同，只对非 arm64 判断使用 `_objc_msgForward_stret`. 

最大的不同，在于 `Aspects` 是判断方法返回值的内存大小，来决定是否使用`_objc_msgForward_stret`。

根据代码上的注释，作者参考了 苹果的 `OS X ABI Function Call Guide` ,以及 ARM 遵循的标准  `Procedure Call Standard for the
ARM® Architecture`.

## 官方文档的解释

抱着搞明白的心态，我也去看了上述文档里面关于 `Return Values` 的说明:
> 一般来说，函数的返回值，和函数存储在同一个寄存器当中。但是有些 **Special Struct** 太大了，超出了寄存器能存储的范围，就只能放置一个指针在存储器上，指向内存中返回值所在的地址。

专门查阅了 `System V Application Binary Interface` 中的 [Intel386 Architecture
Processor Supplement](http://www.sco.com/developers/devspecs/abi386-4.pdf) ，这里对于返回值为结构体类型的存储，有一个比较清楚的界定:
> Structures. The called function returns structures according to their aligned size.
> - Structures 1 or 2 bytes in size are placed in EAX.
> - Structures 4 or 8 bytes in size are placed in: EAX and EDX.
> - Structures of other sizes are placed at the address supplied by the caller. For example, the C++ language occasionally forces the compiler to return a value in memory when it would normally be returned in registers. See Passing Arguments for more information.

IA-32 说明 1,2,4,8 字节大小的结构体，被存储在寄存器中。其它大小的结构体,被放置的在寄存器中的，则是结构体的指针。

Aspects 中的判断，应该就是基于这个的。我心想，靠谱了。

仿佛学习到姿势的我，兴冲冲的去对 JSPatch 提了一个 PR .

事实证明，我还是太年轻 ，[测试结果](https://travis-ci.org/bang590/JSPatch/builds/262684614?utm_source=github_status&utm_medium=notification) 是这样的:
>Xcode7.3 iPhone4s(8.1) 成功
>
>Xcode7.3 iPhone6s(9.3) 失败

发现是在 64 位底下，一些结构体判断失败了。因为在 IA-32 下，寄存器是 32 位的。而新的机型，比如这里测试的 6s 模拟器，则属于 x86-64 ,寄存器是 64 位的。

所以需要增加对 16 字节的判断。

于是，本地对 6s 进行测试通过后，又增加了一次提交:
     
     if (valueSize == 1 || valueSize == 2 || valueSize == 4 || valueSize == 8 || valueSize == 16) {
                      methodReturnsStructValue = NO;
     }
  
然而....[测试结果](https://travis-ci.org/bang590/JSPatch/builds/262684614?utm_source=github_status&utm_medium=notification) 还是不行:
>Xcode7.3 iPhone4s(8.1) 失败
>
>Xcode7.3 iPhone6s(9.3) 成功


上面说了，16 字节的判断是在 64 位机型情况下做的，所以在的 32 位的机型上， 也对 16 字节进行处理，继续使用 `_objc_msgForward` 是会 Crash 的。 

再增加一次[提交](https://github.com/bang590/JSPatch/pull/795/commits/fa2523c33727048771b4e555cce1edae06ca67f4):

    #if defined(__LP64__) && __LP64__
              if (valueSize == 16) {
                 methodReturnsStructValue = NO;
              }
    #endif
    

终于通过了[测试](https://travis-ci.org/bang590/JSPatch/builds/264604490?utm_source=github_status&utm_medium=notification)，完美 : )

这里说明一下，为什么寄存器所能存储的结构体，是本身处理器位数的 2 倍的问题:

比如，在 `x86-64` 中，`RAX` 通常用于存储函数调用的返回结果，但同时也在乘法和除法指令中。在 imul  指令中，2 个 64 位的乘法最多会产生 128 位的结果，就需要 `RAX` 与 `RDX` 共同存储乘法结果. `IA-32` 也是同样的道理.

在苹果描述 `32bit-PowerPC` 函数规则的文档里，关于 `Returning Results` 的也有 2 个寄存器共同存储 1 个返回值情况的描述:
>Values of type long long are returned in the high word of GPR3 and the low word of GPR4.

按照上述结果，Aspects 是有问题的，果然经过测试，在 64 位模拟器上返回结构体 Crash 了，这里是我提供的 [`复现过程`](https://github.com/steipete/Aspects/pull/127)。

## ARM 中的处理

说完了模拟器中的处理，再来看看真机的规则。

查看苹果的  [iOS ABI Function Call Guide](https://developer.apple.com/library/content/documentation/Xcode/Conceptual/iPhoneOSABIReference/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009020-SW1) ， ARMv6 ,ARMv7 等遵循的规则一致。找到   ` Procedure Call Standard for the ARM Architecture (AAPCS)`。有一份在线的 PDF，里面对 `Result Return` 有比较完整的说明:

> The manner in which a result is returned from a function is determined by the type of that result.
For the base standard:
> - A Half-precision Floating Point Type is returned in the least significant 16 bits of r0.
> - A Fundamental Data Type that is smaller than 4 bytes is zero- or sign-extended to a word and returned in r0.
> - A word-sized Fundamental Data Type (e.g., int, float) is returned in r0.
> -  A double-word sized Fundamental Data Type (e.g., long long, double and 64-bit containerized vectors) is
returned in r0 and r1.
> - A 128-bit containerized vector is returned in r0-r3.
> - A Composite Type not larger than 4 bytes is returned in r0. The format is as if the result had been stored in
memory at a word-aligned address and then loaded into r0 with an LDR instruction. Any bits in r0 that lie
outside the bounds of the result have unspecified values.
> - A Composite Type larger than 4 bytes, or whose size cannot be determined statically by both caller and
callee, is stored in memory at an address passed as an extra argument when the function was called (§5.5,
rule A.4). The memory to be used for the result may be modified at any point during the function call.

上面总结我们要的关键信息，大于 4 字节的复合类型返回值，会被存储在内存中的地址上，作为一个额外的参数传递。

### ARM64 为什么没有大小限制？

前面一直判断的，都属于非 ARM64 的，我也很好奇，为什么 ARM64 就没有问题？

查看关于 `ARM64` 调用规则文档里的 `Result Return ` 说明:

The manner in which a result is returned from a function is determined by the type of that result:
- If the type, T, of the result of a function is such that:
```
void func(T arg)
```
would require that arg be passed as a value in a register (or set of registers) according to the rules in §5.4
Parameter Passing, then the result is returned in the same registers as would be used for such an argument.
- Otherwise, the caller shall reserve a block of memory of sufficient size and alignment to hold the result. The
address of the memory block shall be passed as an additional argument to the function in x8. The callee may
modify the result memory block at any point during the execution of the subroutine (there is no requirement for
the callee to preserve the value stored in x8).


上面说到 x8 寄存器。查看关于返回值的寄存器功能的说明，如下:

|寄存器 | 功能 |
| --- | --- | 
|r0…r7 | Parameter/result registers|
| r8|Indirect result location register|


第一条说的，就是返回值会和参数存在一样的寄存器，也就是 x0-x7 中。

第二条说的，除了第一条的情况之外，调用者就会为这个函数预留一各足够大小和对齐的内存块，存在 x8 寄存器中。

由于第二条规则，我们可以知道，只要返回的不是 `void`. arm64 上存储的返回值都是通过指向内存的指针来做的。我也拿了一个非常大的结构体进行验证：

```
typedef struct {
    CGRect rect;
    CGSize size;
    CGPoint orign;
}TestStruct;

typedef struct {
    TestStruct struct1;
    TestStruct struct2;
    TestStruct struct3;
    TestStruct struct4;
    TestStruct struct5;
    TestStruct struct6;
    TestStruct struct7;
    TestStruct struct8;
    TestStruct struct9;
    TestStruct struct10;
}TestBigStruct;
```

测试 `TestBigStruct` ，打印它的方法签名的 `debugDescription`,包含的内容是 `is special struct return? NO`. valueSize 却是 640，寄存器肯定是存放不下,使用的指针指向内存。


### 规则汇总

判定`返回值`的`Special Struct`的条件:
|机器 | 条件 |
| --- | --- | 
|i386 | 大小非 1，2，4，8 字节|
|x86-64| 大小非 1，2，4，8，16 字节|
|arm-32|大于4字节|
|arm-64|不存在的 ：）|

当然，还有判断方法签名的 `debugDescription` 是否含有 `is special struct return? YES` 的方法。

## 总结

因为包含有 PowerPC-32／PowerPC-64／IA-32／X86-64／ARMv6／ARMv7／ARM64 这么多个体系的英文说明，很多还是关于寄存器和汇编的，可以说看的我非常痛苦了。同时收获也是非常大的。

说起来对 `JSPatch` 的原理解读文章，应该没有谁写的比作者本人还好了，里面介绍了项目实际遇到的各种难点。读完下来，其中关于 `super关键字` 的解读，给了我另一个问题的灵感。

建议大家学习新知识时，不妨结合知名的项目进行借鉴，能学到许多知识 : )

这次的探究过程，完全属于本菜鸟自己瞎摸索的，如果有不对的地方，希望大家多拍砖，让我进一步学习。

# 参考

[Introduction to OS X ABI Function Call Guide](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/LowLevelABI/000-Introduction/introduction.html#//apple_ref/doc/uid/TP40002437-SW1)


[PowerPC 体系结构开发者指南](https://www.ibm.com/developerworks/cn/linux/l-powarch/index.html)

[Intel386 Architecture
Processor Supplement](http://www.sco.com/developers/devspecs/abi386-4.pdf)

[iOS ABI Function Call Guide](https://developer.apple.com/library/content/documentation/Xcode/Conceptual/iPhoneOSABIReference/Introduction/Introduction.html#//apple_ref/doc/uid/TP40009020-SW1)

[Procedure Call Standard for the
ARM 64-bit Architecture](http://infocenter.arm.com/help/topic/com.arm.doc.ihi0055b/IHI0055B_aapcs64.pdf)

[Procedure Call Standard for the
ARM® Architecture](http://infocenter.arm.com/help/topic/com.arm.doc.ihi0042f/IHI0042F_aapcs.pdf)

 [JSPatch 实现原理详解](https://github.com/bang590/JSPatch/wiki/JSPatch-%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3) 

[objc_msgSend_stret](http://sealiesoftware.com/blog/archive/2008/10/30/objc_explain_objc_msgSend_stret.html)

[重识 Objective-C Runtime - 看透 Type 与 Value](http://blog.sunnyxx.com/2016/08/13/reunderstanding-runtime-1/)

[什么是-x86、i386、ia32等等](http://blog.csdn.net/dagnet/article/details/6162168)


