# ZLStartPageDemo
动态启动页跳过

这里介绍一下自己在做这种动画时的一种方案。
启动图依然是加载的，只不过是一闪而过，这时候我想到的是拿到当前的LaunchImage图片，然后进行处理，造成一种改变了LaunchImage动画的假象。

我们都知道 APP启动时加载的是LaunchImage 这张静态图。现在好多应用启动时都是动态的,并且右上角可选择跳过。截图如下:
![](hhttps://github.com/ZLFighting/ZLStartPageDemo/blob/master/ZLStartPageDemo/截图.png)


> 思路如下:
根据UIBezierPath和CAShapeLayer自定义倒计时进度条，适用于app启动的时候设置一个倒计时关闭启动页面。可以设置进度条颜色，填充颜色，进度条宽度以及点击事件等。

一.设置跳过按钮
二. 启动页
三. 动态启动页的显示代码放在AppDeleate中.

### 一. 设置跳过按钮

ZLDrawCircleProgressBtn.h :
```
//  跳过按钮

#import <UIKit/UIKit.h>

typedef void(^DrawCircleProgressBlock)(void);

@interface ZLDrawCircleProgressBtn : UIButton

//set track color
@property (nonatomic, strong) UIColor    *trackColor;

//set progress color
@property (nonatomic, strong) UIColor    *progressColor;

//set track background color
@property (nonatomic, strong) UIColor    *fillColor;

//set progress line width
@property (nonatomic, assign) CGFloat    lineWidth;

//set progress duration
@property (nonatomic, assign) CGFloat    animationDuration;

/**
*  set complete callback
*
*  @param lineWidth line width
*  @param block     block
*  @param duration  time
*/
- (void)startAnimationDuration:(CGFloat)duration withBlock:(DrawCircleProgressBlock )block;

@end
```

ZLDrawCircleProgressBtn.m :
**初始化相关跳过按钮及进度圈**
```
#import "ZLDrawCircleProgressBtn.h"

#define degreesToRadians(x) ((x) * M_PI / 180.0)

@interface ZLDrawCircleProgressBtn ()

// 底部进度条圈
@property (nonatomic, strong) CAShapeLayer *trackLayer;
// 表层进度条圈
@property (nonatomic, strong) CAShapeLayer *progressLayer;
@property (nonatomic, strong) UIBezierPath *bezierPath;
@property (nonatomic, copy)   DrawCircleProgressBlock myBlock;

@end

@implementation ZLDrawCircleProgressBtn

- (instancetype)initWithFrame:(CGRect)frame
{
if (self == [super initWithFrame:frame]) {
self.backgroundColor = [UIColor clearColor];

[self.layer addSublayer:self.trackLayer];

}
return self;
}

- (UIBezierPath *)bezierPath {
if (!_bezierPath) {

CGFloat width = CGRectGetWidth(self.frame)/2.0f;
CGFloat height = CGRectGetHeight(self.frame)/2.0f;
CGPoint centerPoint = CGPointMake(width, height);
float radius = CGRectGetWidth(self.frame)/2;

_bezierPath = [UIBezierPath bezierPathWithArcCenter:centerPoint
radius:radius
startAngle:degreesToRadians(-90)
endAngle:degreesToRadians(270)
clockwise:YES];
}
return _bezierPath;
}

- (CAShapeLayer *)trackLayer {
if (!_trackLayer) {
_trackLayer = [CAShapeLayer layer];
_trackLayer.frame = self.bounds;
// 圈内填充色
_trackLayer.fillColor = self.fillColor.CGColor ? self.fillColor.CGColor : [UIColor clearColor].CGColor ;
_trackLayer.lineWidth = self.lineWidth ? self.lineWidth : 2.f;
// 底部圈色
_trackLayer.strokeColor = self.trackColor.CGColor ? self.trackColor.CGColor : [UIColor colorWithRed:197/255.0 green:159/255.0 blue:82/255.0 alpha:0.3].CGColor ;
_trackLayer.strokeStart = 0.f;
_trackLayer.strokeEnd = 1.f;

_trackLayer.path = self.bezierPath.CGPath;
}
return _trackLayer;
}

- (CAShapeLayer *)progressLayer {

if (!_progressLayer) {
_progressLayer = [CAShapeLayer layer];
_progressLayer.frame = self.bounds;
_progressLayer.fillColor = [UIColor clearColor].CGColor;
_progressLayer.lineWidth = self.lineWidth ? self.lineWidth : 2.f;
_progressLayer.lineCap = kCALineCapRound;
// 进度条圈进度色
_progressLayer.strokeColor = self.progressColor.CGColor ? self.progressColor.CGColor  : [UIColor  colorWithRed:197/255.0 green:159/255.0 blue:82/255.0 alpha:1].CGColor;
_progressLayer.strokeStart = 0.f;

CABasicAnimation *pathAnimation = [CABasicAnimation animationWithKeyPath:@"strokeEnd"];
pathAnimation.duration = self.animationDuration;
pathAnimation.fromValue = @(0.0);
pathAnimation.toValue = @(1.0);
pathAnimation.removedOnCompletion = YES;
pathAnimation.delegate = self;
[_progressLayer addAnimation:pathAnimation forKey:nil];

_progressLayer.path = _bezierPath.CGPath;
}
return _progressLayer;
}
```
**设置代理回调**
```
#pragma mark -- CAAnimationDelegate
- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag {
if (flag) {
self.myBlock();
}
}

#pragma mark ---
- (void)startAnimationDuration:(CGFloat)duration withBlock:(DrawCircleProgressBlock )block {
self.myBlock = block;
self.animationDuration = duration;
[self.layer addSublayer:self.progressLayer];
}
```

### 二. 启动页
ZLStartPageView.h :
**露出 显示引导页面方法**
```
//  启动页

#import <UIKit/UIKit.h>

#define kscreenWidth [UIScreen mainScreen].bounds.size.width

@interface ZLStartPageView : UIView

/**
*  显示引导页面方法
*/
- (void)show;

@end
```
ZLStartPageView.m :
**1. 初始化启动页**
```
#import "ZLStartPageView.h"
#import "ZLDrawCircleProgressBtn.h"

@interface ZLStartPageView ()

// 启动页图
@property (nonatomic,strong) UIImageView *imageView;

// 跳过按钮
@property (nonatomic, strong) ZLDrawCircleProgressBtn *drawCircleBtn;

@end

// 倒计时时间
static int const showtime = 3;

@implementation ZLStartPageView

- (instancetype)initWithFrame:(CGRect)frame {

if (self = [super initWithFrame:frame]) {

// 1.启动页图片
_imageView = [[UIImageView alloc]initWithFrame:frame];
_imageView.contentMode = UIViewContentModeScaleAspectFill;
_imageView.image = [UIImage imageNamed:@"LaunchImage_667h"];
[self addSubview:_imageView];

// 2.跳过按钮
ZLDrawCircleProgressBtn *drawCircleBtn = [[ZLDrawCircleProgressBtn alloc]initWithFrame:CGRectMake(kscreenWidth - 55, 30, 40, 40)];
drawCircleBtn.lineWidth = 2;
[drawCircleBtn setTitle:@"跳过" forState:UIControlStateNormal];
[drawCircleBtn setTitleColor:[UIColor  colorWithRed:197/255.0 green:159/255.0 blue:82/255.0 alpha:1] forState:UIControlStateNormal];
drawCircleBtn.titleLabel.font = [UIFont systemFontOfSize:14];

[drawCircleBtn addTarget:self action:@selector(removeProgress) forControlEvents:UIControlEventTouchUpInside];
[self addSubview:drawCircleBtn];
self.drawCircleBtn = drawCircleBtn;

}
return self;
}
```
**2. 显示启动页且完成时候回调移除**
```
- (void)show {

//  progress 完成时候的回调
__weak __typeof(self) weakSelf = self;
[weakSelf.drawCircleBtn startAnimationDuration:showtime withBlock:^{
[weakSelf removeProgress];
}];

UIWindow *window = [UIApplication sharedApplication].keyWindow;
[window addSubview:self];
}
```
**3. 移除启动页面**
```
// 移除启动页面
- (void)removeProgress {

self.imageView.transform = CGAffineTransformMakeScale(1, 1);
self.imageView.alpha = 1;

[UIView animateWithDuration:0.3 animations:^{
self.drawCircleBtn.hidden = NO;
self.imageView.alpha = 0.05;
self.imageView.transform = CGAffineTransformMakeScale(5, 5);
} completion:^(BOOL finished) {

self.drawCircleBtn.hidden = YES;
[self.imageView removeFromSuperview];
}];
}
```

### 三. 动态启动页的显示代码放在AppDeleate中.

```
- (BOOL)application:(UIApplication *)application didFinishLaunchingWithOptions:(NSDictionary *)launchOptions {

self.window = [[UIWindow alloc] initWithFrame:[UIScreen mainScreen].bounds];
self.window.rootViewController = [[UINavigationController alloc] initWithRootViewController:[[HomeViewController alloc] init]];
[self.window makeKeyAndVisible];

[self setupStartPageView];

return YES;
}

/**
*  设置启动页
*/
- (void)setupStartPageView {

ZLStartPageView *startPageView = [[ZLStartPageView alloc] initWithFrame:self.window.bounds];
[startPageView show];
}
```

这个时候可以可以测试喽,效果如下:

![动态启动页.gif](https://github.com/ZLFighting/ZLStartPageDemo/blob/master/ZLStartPageDemo/动态启动页%20下午7.34.54.gif)


如果需要启动页加载广告这种启动页方案 ,请移步: [iOS-APP启动页加载广告](https://github.com/ZLFighting/ZLAdvertDemo)

您的支持是作为程序媛的我最大的动力, 如果觉得对你有帮助请送个Star吧,谢谢啦
