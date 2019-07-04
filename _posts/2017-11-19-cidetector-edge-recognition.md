---
layout: post
title: CIDetector 边缘识别
date: 2017-11-19
Author: madaoCN
categories: 
tags: [源码分享]
comments: true
---

`CoreImage `下`CIDetector.h`自带了四种识别功能
```swift
/ * 人脸识别 */
CORE_IMAGE_EXPORT NSString* const CIDetectorTypeFace NS_AVAILABLE(10_7, 5_0);

/ * 矩形边缘识别 */
CORE_IMAGE_EXPORT NSString* const CIDetectorTypeRectangle NS_AVAILABLE(10_10, 8_0);

/* 二维码识别 */
CORE_IMAGE_EXPORT NSString* const CIDetectorTypeQRCode NS_AVAILABLE(10_10, 8_0);

/* 文本识别 */
#if __OBJC2__
CORE_IMAGE_EXPORT NSString* const CIDetectorTypeText NS_AVAILABLE(10_11, 9_0);
```

接下来用`CIDetectorTypeRectangle `对图片的矩形状边缘进行识别，效果图如下 （__Demo链接文章底部已给出__）

![识别边缘](http://upload-images.jianshu.io/upload_images/1749699-85a3e205b3d691d4.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

![截取](http://upload-images.jianshu.io/upload_images/1749699-849079b90f206f95.PNG?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)

### 部分代码：
* 初始化一个高精度的识别器
```swift
// 高精度边缘识别器
- (CIDetector *)highAccuracyRectangleDetector
{
    static CIDetector *detector = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^
                  {
                      detector = [CIDetector detectorOfType:CIDetectorTypeRectangle context:nil options:@{CIDetectorAccuracy : CIDetectorAccuracyHigh}];
                  });
    return detector;
}
```

* 调用照相机捕获摄像头图像
```swift
NSArray *possibleDevices = [AVCaptureDevice devicesWithMediaType:AVMediaTypeVideo];
    AVCaptureDevice *device = [possibleDevices firstObject];
    if (!device) return;
    
    _imageDedectionConfidence = 0.0;
    
    AVCaptureSession *session = [[AVCaptureSession alloc] init];
    self.captureSession = session;
    [session beginConfiguration];
    self.captureDevice = device;
    
    NSError *error = nil;
    AVCaptureDeviceInput* input = [AVCaptureDeviceInput deviceInputWithDevice:device error:&error];
    session.sessionPreset = AVCaptureSessionPresetPhoto;
    [session addInput:input];
    
    AVCaptureVideoDataOutput *dataOutput = [[AVCaptureVideoDataOutput alloc] init];
    [dataOutput setAlwaysDiscardsLateVideoFrames:YES];
    [dataOutput setVideoSettings:@{(id)kCVPixelBufferPixelFormatTypeKey:@(kCVPixelFormatType_32BGRA)}];
    [dataOutput setSampleBufferDelegate:self queue:dispatch_get_main_queue()];
    [session addOutput:dataOutput];
    
    self.stillImageOutput = [[AVCaptureStillImageOutput alloc] init];
    [session addOutput:self.stillImageOutput];
    
    AVCaptureConnection *connection = [dataOutput.connections firstObject];
    [connection setVideoOrientation:AVCaptureVideoOrientationPortrait];
```

* 还需要有个显示已捕获图像的容器
```swift
 self.context = [[EAGLContext alloc] initWithAPI:kEAGLRenderingAPIOpenGLES2];
    GLKView *view = [[GLKView alloc] initWithFrame:self.bounds];
    view.autoresizingMask = UIViewAutoresizingFlexibleWidth | UIViewAutoresizingFlexibleHeight;
    view.translatesAutoresizingMaskIntoConstraints = YES;
    view.context = self.context;
    view.contentScaleFactor = 1.0f;
    view.drawableDepthFormat = GLKViewDrawableDepthFormat24;
    [self insertSubview:view atIndex:0];
    _glkView = view;
    glGenRenderbuffers(1, &_renderBuffer);
    glBindRenderbuffer(GL_RENDERBUFFER, _renderBuffer);
    
    //图像将绘制进_coreImageContext内
    _coreImageContext = [CIContext contextWithEAGLContext:self.context];
    [EAGLContext setCurrentContext:self.context];
```

* 遵循`AVCaptureVideoDataOutputSampleBufferDelegate`代理，捕获到图像之后，会调用以下方法
```swift
-(void)captureOutput:(AVCaptureOutput *)captureOutput didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection
```
调用`CIDetector `进行识别，并且获取最大不规则四边形
```swift
// 从缓冲区中获取CIImage
CVPixelBufferRef pixelBuffer = (CVPixelBufferRef)CMSampleBufferGetImageBuffer(sampleBuffer);
CIImage *image = [CIImage imageWithCVPixelBuffer:pixelBuffer];
 // 用高精度边缘识别器 识别特征
NSArray <CIFeature *>*features = [[self highAccuracyRectangleDetector] featuresInImage:image];
// 选取特征列表中最大的不规则四边形
 _borderDetectLastRectangleFeature = [self biggestRectangleInRectangles:features];
```

* 识别到边缘之后使用`CAShapeLayer`将边缘绘制并显示
```swift
// 绘制边缘检测图层
- (void)drawBorderDetectRectWithImageRect:(CGRect)imageRect topLeft:(CGPoint)topLeft topRight:(CGPoint)topRight bottomLeft:(CGPoint)bottomLeft bottomRight:(CGPoint)bottomRight
{
    
    if (!_rectOverlay) {
        _rectOverlay = [CAShapeLayer layer];
        _rectOverlay.fillRule = kCAFillRuleEvenOdd;
        _rectOverlay.fillColor = [UIColor colorWithRed:73/255.0 green:130/255.0 blue:180/255.0 alpha:0.4].CGColor;
        _rectOverlay.strokeColor = [UIColor whiteColor].CGColor;
        _rectOverlay.lineWidth = 5.0f;
    }
    if (!_rectOverlay.superlayer) {
        self.layer.masksToBounds = YES;
        [self.layer addSublayer:_rectOverlay];
    }

    // 将图像空间的坐标系转换成uikit坐标系
    TransformCIFeatureRect featureRect = [self transfromRealRectWithImageRect:imageRect topLeft:topLeft topRight:topRight bottomLeft:bottomLeft bottomRight:bottomRight];
    
    // 边缘识别路径
    UIBezierPath *path = [UIBezierPath new];
    [path moveToPoint:featureRect.topLeft];
    [path addLineToPoint:featureRect.topRight];
    [path addLineToPoint:featureRect.bottomRight];
    [path addLineToPoint:featureRect.bottomLeft];
    [path closePath];
    // 背景遮罩路径
    UIBezierPath *rectPath  = [UIBezierPath bezierPathWithRect:CGRectMake(-5,
                                                                          -5,
                                                                          self.frame.size.width + 10,
                                                                          self.frame.size.height + 10)];
    [rectPath setUsesEvenOddFillRule:YES];
    [rectPath appendPath:path];
    _rectOverlay.path = rectPath.CGPath;
}
```
即可显示出实时识别的效果了

* 最后拍照之后的裁剪，使用该滤镜将识别出的不规则四边形转换成矩形，即可转换成正正方方的矩形了
```swift
/// 将任意四边形转换成长方形
- (CIImage *)correctPerspectiveForImage:(CIImage *)image withFeatures:(CIRectangleFeature *)rectangleFeature
{
    NSMutableDictionary *rectangleCoordinates = [NSMutableDictionary new];
    rectangleCoordinates[@"inputTopLeft"] = [CIVector vectorWithCGPoint:rectangleFeature.topLeft];
    rectangleCoordinates[@"inputTopRight"] = [CIVector vectorWithCGPoint:rectangleFeature.topRight];
    rectangleCoordinates[@"inputBottomLeft"] = [CIVector vectorWithCGPoint:rectangleFeature.bottomLeft];
    rectangleCoordinates[@"inputBottomRight"] = [CIVector vectorWithCGPoint:rectangleFeature.bottomRight];
    return [image imageByApplyingFilter:@"CIPerspectiveCorrection" withInputParameters:rectangleCoordinates];
}
```

### // TODO : 识别出边缘之后，可以手动设置边缘范围
![手动设置边缘范围](http://upload-images.jianshu.io/upload_images/1749699-ed2a6863d3ec84db.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/640)



> Demo地址： [github传送门：https://github.com/madaoCN/MADRectDetect  好心人点下Star呗](https://github.com/madaoCN/MADRectDetect)