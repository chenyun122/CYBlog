  在视频或计算机视觉方面的应用中，有时需要识别追踪视频中的特定物体。比如科幻片《头号玩家》中，反派的无人机在寻找主角车辆时，通过匹配之前拍摄的车辆特征图片来识别，并追踪打击。在新的iOS版本中，可以利用CoreML+Vision根据训练好的模型来识别，但此文介绍的是利用OpenCV库的Template Matching来识别，以应付一些简单的场合。我们最终要实现的是在视频中识别苹果Logo（这个Logo是事先拍好的），效果如下动图所示：

<p align="center" >
  <img src="http://p9f3h0583.bkt.clouddn.com/tp3.gif" alt="template matching" title="template matching" width="325px"/>
</p>


  整个项目主要通过以下几个步骤实现：  
**  一. 集成OpenCV最新的iOS版本Framework；  
  二. 使用AVFoundation获取视频流；  
  三. 利用OpenCV进行模版匹配(Template Matching)；  
  四. 绘制矩形提示匹配到的位置。**

想直接看源码的同学可以访问Github中的[项目源码][2]。为了突出重点和关键步骤，本文可能省略一些不重要的细节。请查看项目来了解全部。
* * *

### 一. 集成OpenCV最新的iOS版本Framework

  先在Github上下载OpenCV的最新Release版本：<https://github.com/opencv/opencv/releases>。iOS请选择文件opencv-3.4.1-ios-framework.zip(版本号可能更新)并解压。然后新建一个名为TamplateMatching的新iOS项目，因为需要以C++方式调用OpenCV的函数，所以语言选择Objective-C更加方便一点。最后将解压好的opencv2.framework文件拖到项目中，选择“Copy items if needed”。由于OpenCV部分宏定义和OC的同名，容易冲突，需要在引入iOS库之前`#import <opencv2/opencv.hpp>`,所以建立PCH文件并在其顶部添加：

    #ifdef __cplusplus
    #import <opencv2/opencv.hpp>
    #endif
    

之后将项目编译通过，准备工作完成。

* * *

### 二. 使用AVFoundation获取视频流

  使用AVFoundation获取视频流的方法网上已有很多，这里不做详述。从系统层面看，我们获取视频数据主体的步骤为：1.摄像头捕获视频流；2.输入到系统；3.系统再输出到我们的类中处理。相关的类：先通过AVCaptureDevice类提供用于视频输入的设备，然后绑定到AVCaptureDeviceInput类来提供具体的输入。视频流捕获行为由AVCaptureSession类管理，将AVCaptureDeviceInput对象加入到AVCaptureSession对象中来为Session提供输入。最后由AVCaptureVideoDataOutput类来提供视频的输出，输出的数据就由我们的类来处理。
  在上面建立的项目中的默认ViewController中，我们直接进行视频拍摄。下面代码以尽量简洁的方式获取视频流：

    #import "ViewController.h"
    #import <AVFoundation/AVFoundation.h>
    
    
    @interface ViewController () <AVCaptureVideoDataOutputSampleBufferDelegate> {
    }
    
    @property (nonatomic,strong) AVCaptureSession *captureSession;
    @property (nonatomic,strong) AVCaptureVideoDataOutput *videoOutput;
    @property (nonatomic,strong) AVCaptureVideoPreviewLayer *videoPreviewLayer;
    
    @end
    
    
    @implementation ViewController
    
    - (void)viewDidLoad {
        [super viewDidLoad];
    
        //检查视频权限，有授权则开始视频捕获
        [self checkAuthorization];
    }
    
    //检查视频权限
    - (void)checkAuthorization {
        if ([AVCaptureDevice authorizationStatusForMediaType:AVMediaTypeVideo] == AVAuthorizationStatusAuthorized) {
            [self setupCaptureSession];
        }
        else{
            [AVCaptureDevice requestAccessForMediaType:AVMediaTypeVideo completionHandler:^(BOOL granted) {
                if (granted) {
                    [self setupCaptureSession];
                } else {
                    UIAlertController *alertController = [UIAlertController alertControllerWithTitle:@"未授权拍摄视频" message:@"请前往系统设置开放授权" preferredStyle:UIAlertControllerStyleAlert];
                    UIAlertAction *okAction = [UIAlertAction actionWithTitle:@"好的" style:UIAlertActionStyleCancel handler:^(UIAlertAction * _Nonnull action) {}];
                    [alertController addAction:okAction];
                    [self presentViewController:alertController animated:YES completion:nil];
                }
            }];
        }
    }
    
    //配置视频并开始捕获
    - (void)setupCaptureSession {
        NSArray *possibleDevices = [AVCaptureDevice devicesWithMediaType:AVMediaTypeVideo];
        AVCaptureDevice *videoDevice = [possibleDevices firstObject];
        if (!videoDevice) return;
    
        // 创建Session
        AVCaptureSession *session = [[AVCaptureSession alloc] init];
        [session beginConfiguration];
    
        // 添加输入设备
        NSError *error = nil;
        AVCaptureDeviceInput* input = [AVCaptureDeviceInput deviceInputWithDevice:videoDevice error:&error];
        [session addInput:input];
    
        // 添加输出
        self.videoOutput = [[AVCaptureVideoDataOutput alloc] init];
        //此项设置很重要，在我们处理视频帧图片，需要32BGRA格式。而系统默认的为JPEG格式，所以需要在此设置。
        [self.videoOutput setVideoSettings:@{(id)kCVPixelBufferPixelFormatTypeKey:@(kCVPixelFormatType_32BGRA)}];
        dispatch_queue_t queue = dispatch_queue_create("SampleBufferQueue", NULL);
        [self.videoOutput setSampleBufferDelegate:self queue:queue];
        [session addOutput:self.videoOutput];
    
        // 完成配置
        [session commitConfiguration];
        self.captureSession = session;
    
        // 添加视频预览层
        self.videoPreviewLayer = [AVCaptureVideoPreviewLayer layer];
        self.videoPreviewLayer.frame = self.view.layer.bounds;
        self.videoPreviewLayer.session = session;
        [self.view.layer addSublayer:self.videoPreviewLayer];
    
        // 开始视频
        [session startRunning];
    }
    
    
    #pragma mark - AVCaptureVideoDataOutputSampleBufferDelegate
    - (void)captureOutput:(AVCaptureOutput *)output didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection {
        //我们在这里处理视频数据，用于识别特定物体
    }
    
    @end
    
完成这一步，运行APP，授予摄像权限，便可以看到视频拍摄画面。
  

* * *

### 三. 利用OpenCV进行模板匹配(Template Matching)
  OpenCV进行模版匹配的工作主要是我们传给它一张大图，一张小的模板图，然后它在大图中找到模板图片的内容，并给出匹配到的位置。对于视频的每一帧图片，我们都需要进行一次匹配，确定视频中是否存在要找的物体。我们建立一个类来专门负责这些匹配工作，向项目添加一个继承自`NSObject`的Cocoa Touch Class,命名为`TemplateMatch`。它与其他类的交互如下：
  

<p align="center" >
  <img src="http://p9f3h0583.bkt.clouddn.com/tp.png" alt="template matching" title="template matching" />
</p>

  由于需要以C++方式调用OpenCV的相关函数，所以将.m文件的扩展重命名为.mm来进行Objective C++编码。
  
#### 1. 模板匹配类头文件
头文件很简单，就是用于设置模板图片，和提供模板匹配的方法：
```
#import <UIKit/UIKit.h>
#import <CoreMedia/CoreMedia.h>


@interface TemplateMatch : NSObject

@property(nonatomic,strong) UIImage *templateImage;     //模板图片。由于匹配方法会被多次调用，所以模板图片适合单次设定。

//在Buffer中匹配预设的模板，如果成功则返回位置以及区域大小。
//这里返回的Rect基于AVCapture Metadata的坐标系统，即值在0.0-0.1之间，方便AVCaptureVideoPreviewLayer类进行转换。
- (CGRect)matchWithSampleBuffer:(CMSampleBufferRef)sampleBuffer;

@end
```
由于视频由不同的帧组成，而模板图片则固定不变。所以我们提供一个属性来设置模板图片，一次设置即可。而匹配行为则由一个方法实现，提供给外部多次调用。

#### 2. 模板匹配类实现
##### 2.1 准备工作
OpenCV的模板匹配功能有个缺陷，就是如果模板图片和大图的比例相差太大，则无法匹配到。比如我们要在1000\*1000像素的图片中，查找大概100\*100像素大小的物体，但提供的模版图片只有30\*30，那么匹配到的概率会大大降低。所以我们需要缩放模板图片来提高匹配概率。这里我们使用C++标准库容器来存放各个缩放等级的模板图片,和平方函数来计算各个等级缩放比例。类实现部分的开头如下：

```
#import "TemplateMatch.h"
#include <vector>
#include <math.h>

using namespace cv;
using namespace std;

@interface TemplateMatch() {
    UIImage *_templateImage;
    vector<Mat> _scaledTempls; //各个缩放等级的模板图片矩阵
}

@end
```

在类实现的开头部分我们定义一些常量，以便集中调整计算参数：
```
@implementation TemplateMatch

static const float resizeRatio = 0.35;     //原图缩放比例，越小性能越好，但识别度越低
static const int maxTryTimes = 4;          //未达到预定识别度时，再尝试的次数限制
static const float acceptableValue = 0.7;  //达到此识别度才被认为正确
static const float scaleRation = 0.75;     //当模板未被识别时，尝试放大/缩小模板。 指定每次模板缩小的比例

//......

@end
```


##### 2.2 图片数据的转换
OpenCV用矩阵类`Mat`来代表所处理的图片，所以需要将`UIImage`对象和`CMSampleBufferRef`数据转换为`Mat`对象，并将颜色调整为灰度以更好地适配OpenCV的一些函数。在类实现中添加以下方法：
```
//UIImage转为OpenCV灰图矩阵
- (Mat)cvMatGrayFromUIImage:(UIImage *)image {
    Mat img;
    Mat img_color = [self cvMatFromUIImage:image];
    cvtColor(img_color, img, CV_BGR2GRAY);
    
    return img;
}

//UIImage转为OpenCV矩阵
- (Mat)cvMatFromUIImage:(UIImage *)image {
    CGColorSpaceRef colorSpace = CGImageGetColorSpace(image.CGImage);
    CGFloat cols = image.size.width;
    CGFloat rows = image.size.height;
    
    Mat cvMat(rows, cols, CV_8UC4); // 8位图, 4通道 (颜色 通道 + alpha)
    
    CGContextRef contextRef = CGBitmapContextCreate(cvMat.data,                 // 数据来源
                                                    cols,                       // 宽
                                                    rows,                       // 高
                                                    8,                          // 8位
                                                    cvMat.step[0],              // 每行字节
                                                    colorSpace,                 // 颜色空间
                                                    kCGImageAlphaNoneSkipLast |
                                                    kCGBitmapByteOrderDefault); // Bitmap图信息
    
    CGContextDrawImage(contextRef, CGRectMake(0, 0, cols, rows), image.CGImage);
    CGContextRelease(contextRef);
    
    return cvMat;
}

//Buffer转为OpenCV矩阵
- (Mat)cvMatFromBuffer:(CMSampleBufferRef)buffer {
    CVImageBufferRef pixelBuffer = CMSampleBufferGetImageBuffer(buffer);
    CVPixelBufferLockBaseAddress( pixelBuffer, 0 );
    
    //取得高宽，以及数据起始地址
    int bufferWidth = (int)CVPixelBufferGetWidth(pixelBuffer);
    int bufferHeight = (int)CVPixelBufferGetHeight(pixelBuffer);
    unsigned char *pixel = (unsigned char *)CVPixelBufferGetBaseAddress(pixelBuffer);
    
    //转为OpenCV矩阵
    Mat mat = Mat(bufferHeight,bufferWidth,CV_8UC4,pixel,CVPixelBufferGetBytesPerRow(pixelBuffer));
    
    //结束处理
    CVPixelBufferUnlockBaseAddress( pixelBuffer, 0 );
    
    //转为灰度图矩阵
    Mat matGray;
    cvtColor(mat, matGray, CV_BGR2GRAY);
    
    return matGray;
}
```

##### 2.3 模板图片属性的设置
模板图片一次设置，多次复用。一些准备性工作也在设置时完成，比如各个缩放比例的版本。另外，对于一张iPhone拍摄的原图，OpenCV模板匹配的性能其实并不高，通常会占用5-10秒的时间。这在视频捕获的应用场景中延时太高了，所以我们需要对模板图片和原图进行同比例压缩以提高性能。这里还有一个图片横屏竖屏问题，AVFoundation提供的图片帧默认是横屏方式，相当于iPhone Home键在右侧时拍摄那样。如下图所示：

<p align="center" >
  <img src="http://p9f3h0583.bkt.clouddn.com/tp_portrait.png" alt="coordinates" title="coordinates" />
</p>
所以应该把模板图片逆时针旋转90度。在实际应用中，应该在ViewController中根据手机的旋转情况，对模板图片做相应的旋转，然后提供给TemplateMatch类。这里先简单处理下，就默认为竖屏方式。模板图片属性设置过程如下：

```
//设置模板图片
//由于拍摄会存在拉远拉近的行为，所以需要建立不同大小的模板图片，进行多次匹配
- (void)setTemplateImage:(UIImage *)templateImage {
    //保存默认模板图，并取得模板矩阵
    _templateImage = templateImage;
    Mat templUp = [self cvMatGrayFromUIImage:templateImage];
    
    //本例子默认采用竖屏拍摄，而AVFoundation提供的数据为横屏模式，所以需要将模板图逆时针旋转90度
    //更好的方式，是在ViewController中根据屏幕方向动态旋转模板图,并重新赋值。这里暂时简化处理。
    Mat templ;
    cv::rotate(templUp, templ, ROTATE_90_COUNTERCLOCKWISE);
    
    //设置新模板，需清空旧模板
    _scaledTempls.clear();
    
    //为了提高性能，模板图和原图进行同比列压缩
    Mat templResized;
    resize(templ, templResized, cv::Size(0, 0), resizeRatio, resizeRatio);
    _scaledTempls.push_back(templResized); //默认模板图也存放于模板数组中，以便循环匹配
    
    //由于模板图和原图大小比例不一致，需要放大缩小模板图，来多次比较。所以建立不同比例的模板图。
    for(int i=0;i<maxTryTimes;i++) {
        //放大模板图
        float powIncreaRation = pow(2 - scaleRation, i+1);
        resize(templ, templResized, cv::Size(0, 0), resizeRatio * powIncreaRation, resizeRatio * powIncreaRation);
        _scaledTempls.push_back(templResized); //由于push_back方法执行值拷贝，所以可以复用templResized变量。
        
        //缩小模板图
        float powReduceRation = pow(scaleRation, i+1);
        resize(templ, templResized, cv::Size(0, 0), resizeRatio * powReduceRation, resizeRatio * powReduceRation);
        _scaledTempls.push_back(templResized);
    }
}
```
##### 2.4 进行匹配
匹配时，我们接受`AVFoundation`提供的`CMSampleBufferRef`视频帧数据，然后将它转换成`Mat`矩阵。同模板图片一样，对矩阵进行压缩以提高性能。之后，在一个循环里依次取出各个缩放等级的模板图和大图进行匹配。由于AVCapture Metadata的坐标系统的值范围在(-1,1)之间，所以在匹配到位置后，要按位置在图中的所处的比例来进行换算。这也正好省略了将位置还原的到图压缩前的值。代码如下：
```
//接受Buffer进行匹配
- (CGRect)matchWithSampleBuffer:(CMSampleBufferRef)sampleBuffer {
    Mat img = [self cvMatFromBuffer:sampleBuffer]; //Buffer转换到矩阵
    
    //如果图片为空，则返回空值
    if (resizeRatio <= 0 || img.cols <= 0 || img.rows <= 0) {
        return CGRectZero;
    }
    
    //为了提高性能，将原图缩小。模板图也已同比例缩小。
    Mat imgResized = Mat();
    resize(img, imgResized, cv::Size(0, 0), resizeRatio, resizeRatio);
    
    //进行匹配
    cv::Rect rect = [self matchWithMat:imgResized];

    //除以行列数得到点位置在全图中的比例，转为AVCapture Metadata的坐标系统
    CGPoint point = CGPointMake(rect.x / CGFloat(imgResized.cols), rect.y  / CGFloat(imgResized.rows));
    CGSize templSize = CGSizeMake(rect.width / CGFloat(imgResized.cols), rect.height / CGFloat(imgResized.rows));
    
    return CGRectMake(point.x, point.y, templSize.width, templSize.height);
}

//调用OpenCV进行匹配
//此方法具体解释参考OpenCV官方文档: https://docs.opencv.org/3.2.0/de/da9/tutorial_template_matching.html
- (cv::Rect)matchWithMat:(Mat)img {
    double minVal;
    double maxVal;
    cv::Point minLoc;
    cv::Point maxLoc;

    //匹配不同大小的模板图
    for (int i=0; i < _scaledTempls.size(); i++) {
        Mat templ = _scaledTempls[i];
        
        //创建结果矩阵，用于存放单次匹配到的位置信息(单次会匹配到很多，后面根据不同算法取最大或最小值)
        int result_cols = img.cols - templ.cols + 1;
        int result_rows = img.rows - templ.rows + 1;
        Mat result;
        result.create(result_rows, result_cols, CV_32FC1);
        
        //OpenCV匹配
        matchTemplate(img, templ, result, TM_CCOEFF_NORMED);
        
        //整理出本次匹配的最大最小值
        minMaxLoc(result, &minVal, &maxVal, &minLoc, &maxLoc, Mat());
        
        //TM_CCOEFF_NORMED算法，取最大值为最佳匹配
        //当最大值符合要求，认为匹配成功
        if (maxVal >= acceptableValue) {
            NSLog(@"matched point:%d,%d maxVal:%f, tried times:%d",maxLoc.x,maxLoc.y,maxVal,i + 1);
            return cv::Rect(maxLoc,cv::Size(templ.rows,templ.cols));
        }
    }
    
    //未匹配到，则返回空区域
    return cv::Rect();
}
```
#### 3. 回到ViewController
在TemplateMatch类完成后，我们就可以ViewController进行调用了。先是引入头文件、声明对象、初始化等等，如下代码(只列出了增加的代码)：
```
//......
#import "TemplateMatch.h"

@interface ViewController () <AVCaptureVideoDataOutputSampleBufferDelegate> {
    TemplateMatch *templateMatch;
}

//......

@end

@implementation ViewController

- (void)viewDidLoad {
    //......

    //初始化模板匹配对象，并设置模板图
    templateMatch = [[TemplateMatch alloc] init];
    templateMatch.templateImage = [UIImage imageNamed:@"apple"];

    //.....
}
```
接下来就是视频捕获代理方法中，进行调用了：
```
- (void)captureOutput:(AVCaptureOutput *)output didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection {
    CGRect rect = [templateMatch matchWithSampleBuffer:sampleBuffer]; //将buffer提交给OpenCV进行模板匹配
    dispatch_async(dispatch_get_main_queue(), ^{
        if (!CGRectEqualToRect(rect,CGRectZero)) { //匹配成功，则绘制标识框

        }
        else{ //未匹配到，则隐藏标识框

        }
    });
}
```
就这么几行代码，是不是使用起来So easy? :D
* * *

### 四. 绘制矩形提示匹配到的位置。
最后，要将匹配到位置区域绘制到屏幕上。这里我们就简单增加一个CALayer, 然后设置好红色边框，再根据位置的不同调整下Frame, 就可以有一个动态红框框了。先声明一个对象：
```
@interface ViewController () <AVCaptureVideoDataOutputSampleBufferDelegate> {
    //......
    CALayer *rectangleLayer;
}
```
再定义绘制方法：
```
// 绘制标识框
- (void)drawRectangleAtPoint:(CGRect)rect {
    if (rectangleLayer == nil) {
        rectangleLayer = [CALayer layer];
        rectangleLayer.frame = CGRectMake(0, 0, templateMatch.templateImage.size.width, templateMatch.templateImage.size.height);
        [rectangleLayer setBorderWidth:2.0];
        [rectangleLayer setBorderColor:[UIColor.redColor CGColor]];
        [self.view.layer addSublayer:rectangleLayer];
    }
    
    rectangleLayer.frame = CGRectMake(rect.origin.x, rect.origin.y, rect.size.width, rect.size.height);
    rectangleLayer.hidden = NO;
}

// 隐藏标识框
- (void)hideRectangle {
    rectangleLayer.hidden = YES;
}
```
终于到了世界的尽头，我们再完善下视频捕获代理方法：
```
- (void)captureOutput:(AVCaptureOutput *)output didOutputSampleBuffer:(CMSampleBufferRef)sampleBuffer fromConnection:(AVCaptureConnection *)connection {
    CGRect rect = [templateMatch matchWithSampleBuffer:sampleBuffer]; //将buffer提交给OpenCV进行模板匹配
    dispatch_async(dispatch_get_main_queue(), ^{
        if (!CGRectEqualToRect(rect,CGRectZero)) { //匹配成功，则绘制标识框
            [self drawRectangleAtPoint:[self.videoPreviewLayer rectForMetadataOutputRectOfInterest:rect]]; //由于视频的尺寸和屏幕宽高比不一定一致，所以对于视频中的一个点坐标，需要转换到屏幕的对应位置中。
        }
        else{ //未匹配到，则隐藏标识框
            [self hideRectangle];
        }
    });
}
```
这样一个关于Template Matching的项目就完成了，跑起来看看效果吧。

本文对应[Demo源码][2]下载。  
原创文章，如需转载，请注明出处([https://blog.happyyun.com][1])，非常感谢！

 [1]: https://blog.happyyun.com/2018/04/17/opencv-template-matching/
 [2]: https://github.com/chenyun122/TemplateMatching