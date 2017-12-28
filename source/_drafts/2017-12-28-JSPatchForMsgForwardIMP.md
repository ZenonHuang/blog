# 方法替换之 Special Struct -- 给 JSPatch 提PR

## JSPatch 是什么



## 方法替换是什么

     Class class = aspect_hookClass(object);
     Method targetMethod = class_getInstanceMethod(class, selector);
     IMP targetMethodIMP = method_getImplementation(targetMethod);


不太了解的，可以推荐去看看:
[Objective-C Method Swizzling](http://yulingtianxia.com/blog/2017/04/17/Objective-C-Method-Swizzling/)


## _objc_msgForward_stret 和 _objc_msgForward 的不同

[objc_msgSend_stret](http://sealiesoftware.com/blog/archive/2008/10/30/objc_explain_objc_msgSend_stret.html)


### _objc_msgForward 返回值直接存储在寄存器上

函数在执行完后，返回值也会保存在寄存器上，取这个寄存器的值就是返回值。

### _objc_msgForward_stret 返回值存在指针中

对于某些架构，某些 struct。寄存器要腾出一个位置放这个指针，返回的时候，读取指针写入的数据。


#### 哪些 CPU 架构需要判断

A function result can be returned in registers or in memory, depending on the data type of the function’s return value.

> 放在相同的寄存器中。When the return value of the called function would be passed in registers, if it were passed as a parameter in a function call, the called function places its return value in the same registers

> 指向内存中。Otherwise, the function places its result at the location pointed to by GPR3。


##### ppc32 is trivial: structs never return in registers.

PowerPC32 结构体从来都不在寄存器中返回。

##### i386 is straightforward: structs with sizeof exactly equal to 1, 2, 4, or 8 return in registers

i386 是清晰明了的，寄存器大小等于 1,2,4,8 的结构。

##### x86_64 is more complicated, including rules for returning floating-point struct fields in FPU registers, and ppc64's rules and exceptions will make your head spin

x86_64 和 PowerPC64 的返回规则是非常复杂和让人头疼。


首先判断返回值类型，如果是结构体，就进一步判断.

到苹果官方文档查看
[method_getTypeEncoding](https://developer.apple.com/documentation/objectivec/1418488-method_gettypeencoding)，说明：
>Returns a string describing a method's parameter and return types。

返回值具体字符串代表什么，可以查看 [Type Encodings](https://developer.apple.com/library/content/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Articles/ocrtTypeEncodings.html#//apple_ref/doc/uid/TP40008048-CH100)  

因为不是所有的结构体，都需要用 _objc_msgForward_stret 去存储。

##### IA-32

##### 32bit-PowerPC

##### 64bit-PowerPC

## JSPatch 对 IMP 的实现


    static void overrideMethod(Class cls, NSString *selectorName, JSValue *function, BOOL isClassMethod, const char *typeDescription)
    {
    SEL selector = NSSelectorFromString(selectorName);
    
    if (!typeDescription) {
        Method method = class_getInstanceMethod(cls, selector);
        typeDescription = (char *)method_getTypeEncoding(method);
    }
    
    IMP originalImp = class_respondsToSelector(cls, selector) ? class_getMethodImplementation(cls, selector) : NULL;
    
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

    if (class_getMethodImplementation(cls, @selector(forwardInvocation:)) != (IMP)JPForwardInvocation) {
        IMP originalForwardImp = class_replaceMethod(cls, @selector(forwardInvocation:), (IMP)JPForwardInvocation, "v@:@");
        if (originalForwardImp) {
            class_addMethod(cls, @selector(ORIGforwardInvocation:), originalForwardImp, "v@:@");
        }
    }

    [cls jp_fixMethodSignature];
    if (class_respondsToSelector(cls, selector)) {
        NSString *originalSelectorName = [NSString stringWithFormat:@"ORIG%@", selectorName];
        SEL originalSelector = NSSelectorFromString(originalSelectorName);
        if(!class_respondsToSelector(cls, originalSelector)) {
            class_addMethod(cls, originalSelector, originalImp, typeDescription);
        }
    }
    
    NSString *JPSelectorName = [NSString stringWithFormat:@"_JP%@", selectorName];
    
    _initJPOverideMethods(cls);
    _JSOverideMethods[cls][JPSelectorName] = function;
    
    // Replace the original selector at last, preventing threading issus when
    // the selector get called during the execution of `overrideMethod`
    class_replaceMethod(cls, selector, msgForwardIMP, typeDescription);
    }

其中在非 arm64 使用了如下代码，来判断 Special Struct:

    if ([methodSignature.debugDescription rangeOfString:@"is special struct return? YES"].location != NSNotFound)
    
可以说是一个非常 trick 的做法了.

## Aspects 对 IMP 的实现

### Commit 对比

2014年5月4日 ，Add test and support for struct return types ：

        const char *typeSignature = method_getTypeEncoding(targetMethod);
        BOOL isStruct = (*typeSignature == '{') ? YES : NO;
        class_replaceMethod(class, selector, isStruct ? (IMP)_objc_msgForward_stret : _objc_msgForward, typeSignature);

Improve debug, rename class, better detection if we need _objc_msgForward_stret ：

            // As an ugly internal runtime implementation detail, we need to determine of the method we hook returns a struct or anything larger than double.
        // https://developer.apple.com/library/mac/documentation/DeveloperTools/Conceptual/LowLevelABI/000-Introduction/introduction.html
        NSMethodSignature *signature = [object methodSignatureForSelector:selector];
        BOOL useStret = *typeEncoding == '{' || signature.methodReturnLength > sizeof(double);
        class_replaceMethod(class, selector, useStret ? (IMP)_objc_msgForward_stret : _objc_msgForward, typeEncoding);

make it work on arm64：

        IMP msgForwardIMP = _objc_msgForward;
        #if !defined(__arm64__)
                NSMethodSignature *signature = [object methodSignatureForSelector:selector];
                if (*typeEncoding == '{' || signature.methodReturnLength > sizeof(double)) {
                    msgForwardIMP = (IMP)_objc_msgForward_stret;
                }
        #endif
        class_replaceMethod(class, selector, msgForwardIMP, typeEncoding);

further arm64 fixes:

            if (targetMethodIMP != _objc_msgForward
            #if !defined(__arm64__)
                && targetMethodIMP != (IMP)_objc_msgForward_stret
            #endif
             ) 
             

1.0.0 Release


2014年5月6日 ，Improve detection when to use _objc_msgForward vs _objc_msgForward_stret :

            static BOOL aspect_isMsgForwardIMP(IMP impl) {
                 return impl != _objc_msgForward
            #if !defined(__arm64__)
                 && impl != (IMP)_objc_msgForward_stret
            #endif
                ;
            }
            
            static IMP aspect_getMsgForwardIMP(NSObject *self, SEL selector) {
                IMP msgForwardIMP = _objc_msgForward;
            #if !defined(__arm64__)
             // As an ugly internal runtime implementation detail in the 32bit runtime, we need to determine of the method we hook returns a struct or anything larger than id.
             // https://developer.apple.com/library/mac/documentation/DeveloperTools/Conceptual/LowLevelABI/000-Introduction/introduction.html
            // https://github.com/ReactiveCocoa/ReactiveCocoa/issues/783
            // http://infocenter.arm.com/help/topic/com.arm.doc.ihi0042e/IHI0042E_aapcs.pdf (Section 5.4)
            NSMethodSignature *signature = [self methodSignatureForSelector:selector];
            Method method = class_getInstanceMethod(self.class, selector);
            const char *typeSignature = method_getTypeEncoding(method);
            if ((*typeSignature == _C_STRUCT_B) || signature.methodReturnLength > sizeof(double)) {
                 msgForwardIMP = (IMP)_objc_msgForward_stret;
            }
            #endif
            return msgForwardIMP;
            }


            class_replaceMethod(klass, selector, aspect_getMsgForwardIMP(self, selector), typeEncoding);

negate logic to actually do what the function implies :

             static BOOL aspect_isMsgForwardIMP(IMP impl) {

             return impl == _objc_msgForward
            #if !defined(__arm64__)

            || impl == (IMP)_objc_msgForward_stret
             #endif
                ;
            }
            
               // if (!aspect_isMsgForwardIMP(targetMethodIMP)) {


2014年5月10日，Better stret detection：

            static IMP aspect_getMsgForwardIMP(NSObject *self, SEL selector) {
            BOOL methodReturnsStructValue = NO;
            #if !defined(__arm64__)
            // As an ugly internal runtime implementation detail in the 32bit runtime, we need to determine of the method we hook returns a struct or anything larger than id.
            // https://developer.apple.com/library/mac/documentation/DeveloperTools/Conceptual/LowLevelABI/000-Introduction/introduction.html
            // https://github.com/ReactiveCocoa/ReactiveCocoa/issues/783
            // http://infocenter.arm.com/help/topic/com.arm.doc.ihi0042e/IHI0042E_aapcs.pdf (Section 5.4)
               const char *encoding = method_getTypeEncoding(method);
           methodReturnsStructValue = encoding[0] == _C_STRUCT_B;
           if ( methodReturnsStructValue ) {
              @try {
                  NSUInteger valueSize = 0;
                  NSGetSizeAndAlignment(encoding, &valueSize, NULL);
       
                 if (valueSize == 1 || valueSize == 2 || valueSize == 4 || valueSize == 8) {
                      methodReturnsStructValue = NO;
                   }
              } @catch (NSException *e) {}
            }
            #endif
                 return methodReturnsStructValue ? (IMP)_objc_msgForward_stret : _objc_msgForward;
            }
        
2014年5月11日 ,Fix 64 bit usage:

## 提交PR,测试失败更进一步研究

[测试结果](https://travis-ci.org/bang590/JSPatch/builds/262684614?utm_source=github_status&utm_medium=notification):
>Xcode7.3 iPhone4s(8.1) 成功
>Xcode7.3 iPhone6s(9.3) 失败

增加一次[提交](https://github.com/bang590/JSPatch/pull/795/commits/59f2cc394eff29dde91b58e0f90f2f2fb900082e):
     
     if (valueSize == 1 || valueSize == 2 || valueSize == 4 || valueSize == 8 || valueSize == 16) {
                      methodReturnsStructValue = NO;
     }
  
[测试结果](https://travis-ci.org/bang590/JSPatch/builds/262684614?utm_source=github_status&utm_medium=notification):
>Xcode7.3 iPhone4s(8.1) 失败
>Xcode7.3 iPhone6s(9.3) 成功

What a fucking day! 表情包


## 通过PR,完善发现

再增加一次[提交](https://github.com/bang590/JSPatch/pull/795/commits/fa2523c33727048771b4e555cce1edae06ca67f4):

    #if defined(__LP64__) && __LP64__
              if (valueSize == 16) {
                 methodReturnsStructValue = NO;
              }
    #endif
    

终于通过了[测试](https://travis-ci.org/bang590/JSPatch/builds/264604490?utm_source=github_status&utm_medium=notification)，完美 : )

表情包

## 原理

### LP64 

__LP64__则是由预处理器定义的宏，代表当前操作系统是64位。

LP64意思是 long 和 pointer 是 64 位，ILP64 指 int，long，pointer 是 64位，LLP 指 long long 和 pointer 是 32-bit 的。ILP32 指 int，long 和 pointer 是32位的，LP32 指 long 和 pointer 是32位的。

[](http://blog.csdn.net/wyywatdl/article/details/4683762)

32位环境涉及"ILP32"数据模型，是因为C数据类型为32位的int、long、指针。而64位环境使用不同的数据模型，此时的long和指针已为64位，故称作"LP64"数据模型。

### 32-bit PowerPC Function Calling Conventions

当函数（例程）调用其他函数（子例程）时，可能需要将 arguments 传递给被调用的函数。

被调用的函数接受这些 arguments 作为 parameters。

与传递下去相反，一些函数会返回一个结果或返回值，给他们的调用者。

使用 32位架构的寄存器 还是 runtime stack(堆的英文是heap；栈的英文是stack) 来存储方法的参数和结果，取决于涉及的数据类型。

为了在调用者和被调用者之间很好的传递参赛和结果，GCC 遵循严格的规则来生成程序的对象代码。

这篇文章描述了一些可以用来操作方法调用的参数和结果的数据类型，函数怎么去传递参数给其它函数，以及函数怎么返回结果给它的调用者。并列出了一些在 32 位 cpu 架构下可用的寄存器 以及它们的值在函数调用之后是否会被保存。

#### Data Types and Data Alignment

为你的变量使用正确的数据类型 以及 为你的数据设置适当的 data alignment  可以最大化您的程序的性能和可移植性。data alignment 指定如何在内存中放置数据。

列出了 ANSI C scalar data types。以及这些 data types 在 32位 cpu 环境的大小和它们的默认对齐方式。



**Table 1**  Size and natural alignment of the scalar data types

| Data type|  Size and natural alignment (in bytes) |
| --- | --- | --- |
|  `_Bool`, `bool`|  4 |
|  `unsigned char`|  1 |
|  `char`, `signed char` |  1 |
|  `unsigned short` |  2 |
|  `signed short` |  2|
|  `unsigned int` |  4 |
|  `signed int` |  4 |
|  `unsigned long` |  4 |
|  `signed long` |  4 |
|  `unsigned long long` |  8 |
|  `signed long long` |  8 |
|  `float`|  4 |
|  `double` |  8 |
|  `long double` |  16*|
|  pointer |  4|




(*) In OS X v10.4 and later and GCC 4.0 and later, the size of the `long double` extended precision data type is 16 bytes (it’s made up of two 8-byte doubles). In earlier versions of OS X and GCC, `long double` is equivalent to `double`. You should not use the `long double` type when you use GCC 4.0 or later to develop or in programs targeted at OS X versions earlier than 10.4.

(*) 在 `OS X` v10.4 和 `GCC` 4.0 之后版本中，`long double` 被扩展的精度数据类型的大小是 16 字节(由2个8字节的 `double`组成)。在早期的 `OS X` 和 `GCC` 中，`long double` 相当于 `double`.当您使用 `GCC` 4.0 和之后版本做开发，或在 10.4 以前的 `OS X` 版本中使用的程序时，不应使用 `long double` 类型。

### 标量

### 结构体

[结构体](https://zhuanlan.zhihu.com/p/32310843)

### 内存对齐

处理器并不是按1字节来存取内存的.它一般会以2字节,4字节,8字节,16字节甚至32字节 作为单位来存取内存.

我们将上述这些 存取单位 称为 内存存取粒度 .

现在的我们说常见 32位/64位 处理器，就是分别以 4字节/8字节 作为内存存取粒度。

#### 为什么要有内存对齐

上面说了一个内存存取粒度的概念，不同的处理器有不同的存取粒度。

大的存取粒度，能大幅减少读取的时间， 16 字节的数据，32位处理器需要读取 4 次，64位处理器只需要 2 次，少了一半。

##### 性能问题

有些平台每次读都是从偶地址开始，这时候使用4字节为存取粒度，从地址 1 读取 4 字节数据。
地址从 0 开始，读取数据占据的地址是 1-4，过程就是：
第1次: 0-3.
第2次：4-7.

本来一次可以读完的数据，却花费了 2 次读取，造成了不必要的开销。

而在对齐的内存地址上,4字节处理器能直接 1 次读取完成。这就是我们所说的内存对齐。

##### Q:cpu通过寻址去内存存取数据的时候，为什么一定要从0开始呢？比如说：取数据的时候，取一个int型的数据，肯定是知道了它的起始地址a和长度l的，然后从这个它的起始地址a，往下取l个字节不就对了么？为什么要从0开始取？然后还要分开几次取？

有些CPU只能对字长倍数的内存地址进行访问.

CPU 的访问粒度不仅仅是大小限制，地址上也有限制。也就是说，CPU 只能访问对齐地址上的固定长度的数据。
以四字节对齐为例，就是只能访问 0x0 - 0x3，0x4 - 0x7, 0x8 - 0xc 这样的（闭）区间，不能跨区间访问。
如果真正需要访问的数据并没有占据那个区间的全部字节范围，还有另外的信号线来指出具体操作哪几个字节，类似于掩码的作用。好像也有些架构干脆就不允许这种部分访问，强制要求按粒度访问。

如果一个数据跨越了两个这样的区间，那么就只能将这个数据的操作拆分成两部分，分别执行，效率当然就低了。

硬件不支持一次访问就读到0x02～0x05.

##### 平台原因

各个硬件平台对存储空间的处理上有很大的不同。一些平台对某些特定类型的数据只能从某些特定地址开始存取。这时候就需要按照各自平台的规则做对齐了。


[Data Alignment](http://blog.csdn.net/lgouc/article/details/8235471)

#### 对齐的规则

每个特定平台上的编译器都有自己的默认“对齐系数”。

## 总结

说起来对 JSPatch 的原理解读文章，应该没有谁写的比作者本人还好了，里面介绍了项目实际遇到的各种难点。读完下来，相信一定会非常有收获 :[JSPatch 实现原理详解](https://github.com/bang590/JSPatch/wiki/JSPatch-%E5%AE%9E%E7%8E%B0%E5%8E%9F%E7%90%86%E8%AF%A6%E8%A7%A3)，其中关于 `super关键字` 的解读，给了我另一个项目的灵感。

# 参考

[Introduction to OS X ABI Function Call Guide](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/LowLevelABI/000-Introduction/introduction.html#//apple_ref/doc/uid/TP40002437-SW1)

[32-bit PowerPC Function Calling Conventions](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/LowLevelABI/100-32-bit_PowerPC_Function_Calling_Conventions/32bitPowerPC.html#//apple_ref/doc/uid/TP40002438-SW20)


[64-bit PowerPC Function Calling Conventions](https://developer.apple.com/library/content/documentation/DeveloperTools/Conceptual/LowLevelABI/110-64-bit_PowerPC_Function_Calling_Conventions/64bitPowerPC.html#//apple_ref/doc/uid/TP40002471-SW14)

[objc_msgSend_stret](http://sealiesoftware.com/blog/archive/2008/10/30/objc_explain_objc_msgSend_stret.html)

