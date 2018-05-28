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
  
#### 1.模板匹配类头文件
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

#### 2.模板匹配类实现
##### 2.1首先是模板图片属性的实现。
模板图片属性的设置并不是存储一张图片那么简单。OpenCV的模板匹配功能有个缺陷，就是如果模板图片和大图的比例相差太大，则无法匹配到。比如我们要在1000\*1000像素的图片中，查找大概100\*100像素大小的物体，但提供的模版图片只有30\*30，那么匹配到的概率会大大降低。所以我们需要缩放模板图片来提高匹配概率。这里我们使用C++标准库容器来存放各个缩放等级的模板图片,和平方函数来计算各个等级缩放比例。类实现部分的开头如下：

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


模板图片的设置方法如下：

* * *

### 四. 绘制矩形提示匹配到的位置。

本文对应[Demo源码][2]下载。  
原创文章，如需转载，请注明出处([https://blog.happyyun.com][1])，非常感谢！

 [1]: https://blog.happyyun.com/2018/04/17/opencv-template-matching/
 [2]: https://github.com/chenyun122/TemplateMatching