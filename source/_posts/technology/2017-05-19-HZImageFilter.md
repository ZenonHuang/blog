---
layout: post
date: 2017-05-19 10:50:46
title:  CoreImage滤镜实践
category: 技术
tags: 
- 资料

keywords: 滤镜
description: 使用Core Image完成基础的滤镜功能，滤镜链，视频实时滤镜，人脸贴图
---
# CoreImage滤镜实践

最近因为项目需求,接触了Core Image,实现了实时抠图,人脸贴纸。

决定把这次实践分享给大家,作为对开源社区的回馈。

相关代码上传在 [GitHub](https://github.com/ZernonHuang/HZImageFilter)

## CoreImage介绍

以下是[苹果官方文档](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/CoreImaging/ci_intro/ci_intro.html)给出 Core Image 的介绍：

>Core Image is an image processing and analysis technology designed to provide near real-time processing for still and video images. It operates on image data types from the Core Graphics, Core Video, and Image I/O frameworks, using either a GPU or CPU rendering path. Core Image hides the details of low-level graphics processing by providing an easy-to-use application programming interface (API). You don’t need to know the details of OpenGL, OpenGL ES, or Metal to leverage the power of the GPU, nor do you need to know anything about Grand Central Dispatch (GCD) to get the benefit of multicore processing. Core Image handles the details for you.

翻译过来的大意是：

Core Image 是一个被设计用来给图片和视频提供实时的图像处理和分析技术.它可以使用 GPU 或 CPU 渲染，操作来自 Core Graphics, Core Video,以及Image I/O 框架的图片数据类型. Core Image 隐藏了底层图形处理的细节，提供简单易用的应用程序 API 接口.你不需要知道关于OpenGL, OpenGL ES,或者 Metal 的细节来利用 GPU 的能力，也不需要你知道任何关于 GCD 的知识来进行多线程处理的优化。



### CIFilter

CIFilter,是用来表示 Core Image 的各种滤镜。通过提供对应的键值对，来设置滤镜的输入参数。这些值设置好，CIFilter就可以用来生成新的CIImage输出图像。

### CIImage

 CIImage 是 Core Image 框架中最基本代表图像的对象。可以通过以下几种方式来创建 CIImage:
 
```
 //通过 NSURL 创建
 CIImage*image=[CIImage imageWithContentsOfURL:myURL]; 
 
 //通过 NSData 创建 
 CIImage*image=[CIImage imageWithData:myData];  
 
 //通过 CGImage 创建
 CIImage*image=[CIImage imageWithCGImage:myCgimage]; 
  
 //通过 CVPixelBuffer 创建
 CIImage*image=[CIImage imageWithCVPixelBuffer：CVBuffer]; 
 
```

### CIContext

所有Core Image的处理流程都通过 CIContext 来进行。

CIContext 可以是基于 CPU 的，也可以是基于 GPU 的，这两种渲染的区别是：

-  CPU 渲染方式会采用 GCD 多线程来对图像进行渲染，这保证了 CPU 渲染在大部分情况下更可靠，他可以在后台实现渲染过程
-  GPU 渲染方式可以使用 OpenGLES 或者 Metal 来渲染图像，这种方式 CPU 完全无负担，应用程序的运行循环不会受到图像渲染的影响，而且他渲染比 CPU 渲染更快。缺点是 GPU 渲染无法在后台运行。

```
     //创建基于 GPU 的 CIContext 对象
     //内部的渲染器会根据设备最优选择。依次为 Metal，OpenGLES，CoreGraphics。
CIContext *context = [CIContext contextWithOptions: nil];

     //创建基于 GPU 的 CIContext 对象
     //支持实时渲染，渲染始终在 GPU 上进行，不会复制回 CPU 存储器上，这就保证了更快的渲染速度和更好的性能。
EAGLContext *eaglctx = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
CIContext *context = [CIContext contextWithEAGLContext:eaglctx];
        
     //创建基于CPU的CIContext对象
CIContext *context = [CIContext contextWithOptions:[NSDictionary dictionaryWithObject:[NSNumber numberWithBool:YES] forKey:KCIContextUseSoftWareRenderer]];
```

一般采用第一种基于GPU的，因为效率要比CPU高很多.


## 基础滤镜使用

你可以这样来使用它

```
        //将UIImage转换成CIImage
        CIImage *ciImage = [[CIImage alloc] initWithImage:self.originImage];
        
        //创建滤镜
        CIFilter *filter = [CIFilter filterWithName:filterName 
                                      keysAndValues:kCIInputImageKey,ciImage, nil];
                                      
        //已有的值不改变，其他的设为默认值
        [filter setDefaults];
        
        //创建一个CIContext对象
        CIContext *context = [CIContext contextWithOptions:nil];
        
        //滤镜渲染并输出一个CIImage
        CIImage *outputImage = [filter outputImage];
        
        //创建CGImage句柄
        CGImageRef cgImage = [context createCGImage:outputImage fromRect:[outputImage extent]];
        
        //获取图片
        UIImage *image = [UIImage imageWithCGImage:cgImage];
        
        //释放CGImage句柄
        CGImageRelease(cgImage);
        
        //得到处理后的image后赋值
        self.imageView.image = image;
```

## 滤镜链

滤镜链的使用也非常简单，将滤镜A的输出图像，当作滤镜B的输入图像，就可以把滤镜效果叠加起来。
如：

```
  //得到要处理的图片
  CIImage *ciImage=[[CIImage alloc] initWithImage:self.originImage];
  //创建滤镜A
  CIFilter *filterA = [CIFilter filterWithName:filterNameA 
  //得到滤镜A的输出图像                          keysAndValues:kCIInputImageKey,ciImage, nil];
  CIImage *ciImageA=filterA.outputImage;
  //创建滤镜B，将滤镜A的输出图像做为输入图像 
  CIFilter *filterB = [CIFilter filterWithName:filterNameB 
                                      keysAndValues:kCIInputImageKey,ciImageA, nil];
   //得到最后的输出图像
  CIImage *ciImageB=filterB.outputImage;

```

## 实时滤镜

由于视频本身是由一帧帧的图像组成的，所以我们在添加滤镜的时候，只要处理每一帧的图片就好了。

所以在 AVCaptureVideoDataOutputSampleBufferDelegate
对应的方法里做处理就ok了：

```
-(void)captureOutput:(AVCaptureOutput *)captureOutput didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection {
    
    //得到要处理的图片
    CVPixelBufferRef pixelBuffer = (CVPixelBufferRef)CMSampleBufferGetImageBuffer(sampleBuffer);
    CIImage *image = [CIImage imageWithCVPixelBuffer:pixelBuffer];
  
    image=[self inputCIImageForDetector:image];
    
     /** 使用 CISourceOverCompositing 滤镜，制定前后的图片关系，可以把两张图片组合**/
    CIImage *resulImage = [[CIFilter filterWithName:@"CISourceOverCompositing" 
                                      keysAndValues:kCIInputImageKey,image,kCIInputBackgroundImageKey,self.bgImage,nil] 
                           valueForKey:kCIOutputImageKey];
  
  
    [self.glkView bindDrawable];
    if( !(self.glContext == [EAGLContext currentContext])) {
        [EAGLContext setCurrentContext:self.glContext];
    }
    // clear eagl view to grey
    glClearColor(0.5, 0.5, 0.5, 1.0); 
    glClear(GL_COLOR_BUFFER_BIT);
    
    // set the blend mode to "source over" so that CI will use that
    glEnable(GL_BLEND);  
    glBlendFunc(GL_ONE, GL_ONE_MINUS_SRC_ALPHA);
    [self.ciContext drawImage:resulImage
                       inRect:CGRectMake(0, 0, ScreenWidth*2, ScreenHeight*2) 
                     fromRect:CGRectMake(0, 0, [image extent].size.width,[image extent].size.height) ];
    [self.glkView display];
    
}

```
## 人脸检测
Core Image 已经提供了 CIDetector 来做特征检测，里面包含了 CIDetectorTypeFace 的特征类型来做人脸识别。并且它已被优化过，使用起来也很容易。

```
//创建一个CIDetector对象
CIDetector *faceDetector = [CIDetector detectorOfType:CIDetectorTypeFace context:context options:@{CIDetectorAccuracy: CIDetectorAccuracyHigh}];
//给出要识别的图片，得到一个人脸结果的数组
NSArray *faces = [faceDetector featuresInImage:image];
```

数组 faces 中保存着 CIFaceFeature 类的实例。通过CIFaceFeature 的实例，不仅可以得到脸部的位置信息，也可以知道眼睛和嘴巴的位置。

```

    for (CIFeature *f in faces){
            CIFaceFeature *faceFeature=(CIFaceFeature *)f;
            // 判断是否有左眼位置
            if(faceFeature.hasLeftEyePosition){
                  
            }
            // 判断是否有右眼位置
            if(faceFeature.hasRightEyePosition){
           
            }
            // 判断是否有嘴位置
            if(faceFeature.hasMouthPosition){
            
            }
            
                        
    }

```

在这里需要注意的是，Core Image 和 UIKit 使用了不同的坐标系: Core Image 是以左下角作为原点，UIKit 是以左上角为原点。需要用 仿射变换 将 Core Image 坐标转换成了 UIKit 坐标。

参考资料:

[Core Image Programming Guide](https://developer.apple.com/library/content/documentation/GraphicsImaging/Conceptual/CoreImaging/ci_intro/ci_intro.html#//apple_ref/doc/uid/TP30001185)

[在iOS上用 Core Image 实现人脸检测](http://swift.gg/2017/05/11/face-detection-core-image/)

[Core Image 你需要了解的那些事~](http://colin1994.github.io/2016/10/21/Core-Image-OverView/)

