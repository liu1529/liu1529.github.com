---
title: iOS Drawing
layout: post
tags: [iOS,Drawing]
comments: true
---

## UIImage和UIImageView

最简单的方法，使用`imageNamed:`方法创建UIImage，它搜索Asset catalog（首先搜索）和Top level of app bundle。
或者通过`imageWithContentsOfFile`方法直接搜索`[NSBundle mainBundle]`。

UIIamgeView的contentMode属性决定了图像在view中怎么显示。UIViewContentModeScaleToFill
意味着缩放当这个view的大小显示，UIViewContentModeCenter表示显示在view的中间，且不缩放。

当你代码中创建一个UIImageView时，contentMode属性默认为UIViewContentModeScaleToFill，
但是image不会初始缩放，而是view会缩放到image的尺寸，你需要将UIImageView放在superview正确
的地方。

{% highlight objc %}

UIImageView* iv = [[UIImageView alloc] initWithImage:[UIImage imageNamed:@"Mars"]];
[mainview addSubview: iv];
iv.center = CGPointMake(CGRectGetMidX(iv.superview.bounds),
                            CGRectGetMidY(iv.superview.bounds));
iv.frame = CGRectIntegral(iv.frame);

{% endhighlight %}

当你给一个已经存在的UIImageView赋值Image时，它的尺寸取决于它使用的autolayout。如果未使用
autolayout，或image尺寸小于view尺寸，view的尺寸不会改变。但是在autolayout下，image的
尺寸会赋值给view的`intrinsicContentSize`，所以view的尺寸会适配image的尺寸，除非有其他
的constrain。

## Resizable Images

UIImage可以调用`resizableImageWithCapInsets:resizingMode: `方法可以被转换为resizable image。


## Image Rendering Mode

在iOS的有些地方，image被当做transparentmask，也被称为template。这意味着image的颜色改变
被忽略，只有alpha和tint color共同起作用。在iOS7中，在image的rederingMode模式下，image被
当做图像。这个属性是只读的，要想改变它，通过调用`imageWithRenderingMode:`新建图像，rederingMode
有如下值：

* UIImageRenderingModeAutomatic（默认值）
* UIImageRenderingModeAlwaysOriginal
* UIImageRenderingModeAlwaysTemplate

iOS7中，每个view都有一个tintColor，用于重新染色它包含的任何template image。而且，subview会
继承superview的tintColor，因此改变window的tintColor可以很方便的改变整个app的颜色（如果你使用
main storyboard，使用File inspector设置）。每个view重写tintColor，便可以拥有自己的tintColor。

## Graphics Contexts

graphic context是在代码中唯一能绘制（Drawing）的地方，有很多方法可以获得graphic
context，在这章中主要介绍两种：

* 创建一个image context

通过函数`UIGraphicsBeginImageContextWithOptions`创建一个可用于产生image的
graphic context。完成以后，通过`UIGraphicsGetImageFromCurrentImageContext`
获得一个UIImage，最后调用`UIGraphicsEndImageContext`。

* Cocoa给你一个graphics context

你子类化UIView然后实现`drawRect:`。在你的`drawRect:`中cocoa已经给你创建了一个
graphic context，在函数中任何绘制的东西都将显示在界面上。（你子类化CALayer并实现
`drawRect:`，或某个对象有layer的委托并实现`drawRect:`时，有所不同）

此外，在任何时候要么是当前graphics context要么不是：

* `UIGraphicsBeginImageContextWithOptions`创建一个image context，并使这个context
为当前graphic context。

* `drawRect:`被调用的时候，已经是当前graphic context。

* `context:`的回调函数参数不会使任何context变为当前graphic context，这个参数这是
graphic context的引用。

Drawing有两套独立的绘制工具：

* UIKit

很多object-C类知道怎么绘制自己，包括NSString(绘制文本)，UIBezierPath（绘制形状），
和UIColor。它们提供了很多方便的方法来绘制，在大多数情况下，你只需要使用UIKit。
使用UIKit，你必须在当前context中使用，这在`UIGraphicsBeginImageContextWithOptions`
或`drawRect:`当然没有问题。但是当使用`context:`参数时，你必须使用`UIGraphicsPushContext`
将其变为当前graphic context，确保在使用完成后调用`UIGraphicsPopContext`。

* Core Graphics

这是一个完全的drawing api。Core Graphics通常称为Quartz（或Quartz 2D），它是
iOS所有绘制的基础，所以它是一个底层一系列C函数，UIKit在其上层封装。这章会使你熟悉
一些基本的绘制原理，请参考apple的Quartz 2D Programming Guide。
在Core Graphics中，在使用的每个函数中，你必须制定一个Graphics context（CGContexRef）。
在`UIGraphicsBeginImageContextWithOptions`或`drawRect:`中，你需要调用
`UIGraphicsGetCurrentContext`获得一个context引用。



##  UIImage Drawing

UIImage提供在当前context绘制自己的方法。

{% highlight objc %}

UIImage* mars = [UIImage imageNamed:@"Mars"];
CGSize sz = mars.size;
UIGraphicsBeginImageContextWithOptions(
        CGSizeMake(sz.width*2, sz.height), NO, 0);
[mars drawAtPoint:CGPointMake(0,0)];
[mars drawAtPoint:CGPointMake(sz.width,0)];
UIImage* im = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();

{% endhighlight %}

上面的代码，创建一个两个image水平并排的图形。如果你有两种不同分辨率的图片，代码会使用当前设备
正确的分辨率图片，并设置图片正确的scale属性。调用`UIGraphicsBeginImageContextWithOptions`时，
第3个参数为0，所以这儿绘制的image context也有正确的scale，最后`UIGraphicsGetImageFromCurrentImageContext`
也会得到正确的分辨率图片。因此，相同的代码在不同分辨率的设备上也能正确的工作。



{% highlight objc %}

UIImage* mars = [UIImage imageNamed:@"Mars"];
CGSize sz = mars.size;
UIGraphicsBeginImageContextWithOptions(
        CGSizeMake(sz.width*2, sz.height*2), NO, 0);
[mars drawInRect:CGRectMake(0,0,sz.width*2,sz.height*2)];
[mars drawInRect:CGRectMake(sz.width/2.0, sz.height/2.0, sz.width, sz.height)
           blendMode:kCGBlendModeMultiply alpha:1.0];
UIImage* im = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
    
{% endhighlight %}

上面的代码，创建一个image包含另一个image的图形。UIImage方法在矩形中绘制时会自动缩放，
你也可以指定image的合成模式（blend）。

{% highlight objc %}
UIImage* mars = [UIImage imageNamed:@"Mars"];
CGSize sz = mars.size;
UIGraphicsBeginImageContextWithOptions(
        CGSizeMake(sz.width/2.0, sz.height), NO, 0);
[mars drawAtPoint:CGPointMake(-sz.width/2.0, 0)];
UIImage* im = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();

{% endhighlight %}

上面的代码，会创建image的右半边图形。UIImage的绘制方法中，没有指定源文件的范围的方法，你可以设置一个
更小的context，然后绘制你需要显示的部分。

## CGImage Drawing

UIImage的Core Graphics版本是CGImage（实际上是CGImageRef）。

CIImage比UIImage更加强大，现在我们把一个图形分成两半显示，注意我们现在处于CFTypeRef世界中，
因此我们必须手动管理内存。

{% highlight objc %}
UIImage* mars = [UIImage imageNamed:@"Mars"];
// extract each half as a CGImage
CGSize sz = mars.size;
CGImageRef marsLeft = CGImageCreateWithImageInRect([mars CGImage],
                           CGRectMake(0,0,sz.width/2.0,sz.height));
CGImageRef marsRight = CGImageCreateWithImageInRect([mars CGImage],
                            CGRectMake(sz.width/2.0,0,sz.width/2.0,sz.height));
// draw each CGImage into an image context
UIGraphicsBeginImageContextWithOptions(
        CGSizeMake(sz.width*1.5, sz.height), NO, 0);
CGContextRef con = UIGraphicsGetCurrentContext();
CGContextDrawImage(con,
                    CGRectMake(0,0,sz.width/2.0,sz.height), marsLeft);
CGContextDrawImage(con,
CGRectMake(sz.width,0,sz.width/2.0,sz.height), marsRight);
UIImage* im = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
CGImageRelease(marsLeft); CGImageRelease(marsRight);

{% endhighlight %}

但是，上面的代码有一个问题，绘制的图形是上下颠倒的，它是上下对称颠倒的，专业术语叫做`fipped`。
这个问题，在你创建一个CGImage，然后使用它在`CGContextDrawImage`中绘制时会出现，这是由于
源contex和目标context的坐标系不匹配导致的。

这儿有很多方法解决这种坐标系不匹配的问题，比如在一个中间UIImage中绘制CGImage，然后从中导出，
下面的代码演示了这种方法。

{% highlight objc %}

CGImageRef flip (CGImageRef im) {
CGSize sz = CGSizeMake(CGImageGetWidth(im), CGImageGetHeight(im));
UIGraphicsBeginImageContextWithOptions(sz, NO, 0);
CGContextDrawImage(UIGraphicsGetCurrentContext(),
                       CGRectMake(0, 0, sz.width, sz.height), im);
CGImageRef result = [UIGraphicsGetImageFromCurrentImageContext() CGImage];
UIGraphicsEndImageContext();
return result;
}

{% endhighlight %}

在上个例子中，我们使用这种方法修正

{% highlight objc %}
CGContextDrawImage(con, CGRectMake(0,0,sz.width/2.0,sz.height),
                       flip(marsLeft));
CGContextDrawImage(con, CGRectMake(sz.width,0,sz.width/2.0,sz.height),
                       flip(marsRight));
{% endhighlight %}

但是，当你使用不同分辨率的图片时任然有问题，在使用双倍分辨率的图片时，所有的绘制输出都是错误的。原因是，
我们通过`imageNamed:`方法获得图片，它在双倍分辨率的设备中，会返回一个通过设置scale属性来补偿
尺寸的图片。但是，CGImage没有scale属性，它不知道image是双倍分辨率。所以，在双倍分辨率的设备中，
我们通过调用`image.CGImage`从UIImage中得到是两倍`image.size`的CGImage，导致以后所有的计算都是
错误的。

所有，为了得到正确的CGImage，我们必须指定正确的scale或明确指定CGImage的分辨率。下面的代码，可以正确的
工作在不同分辨率的设备中。

{% highlight objc %}

UIImage* mars = [UIImage imageNamed:@"Mars"];
CGSize sz = mars.size;
// Derive CGImage and use its dimensions to extract its halves
CGImageRef marsCG = [mars CGImage];
CGSize szCG = CGSizeMake(CGImageGetWidth(marsCG), CGImageGetHeight(marsCG));
CGImageRef marsLeft =
	CGImageCreateWithImageInRect(
            marsCG, CGRectMake(0,0,szCG.width/2.0,szCG.height));
CGImageRef marsRight =
        CGImageCreateWithImageInRect(
            marsCG, CGRectMake(szCG.width/2.0,0,szCG.width/2.0,szCG.height));
UIGraphicsBeginImageContextWithOptions(
        CGSizeMake(sz.width*1.5, sz.height), NO, 0);
// The rest is as before, calling flip() to compensate for flipping
CGContextRef con = UIGraphicsGetCurrentContext();
CGContextDrawImage(con, CGRectMake(0,0,sz.width/2.0,sz.height),
                       flip(marsLeft));
CGContextDrawImage(con, CGRectMake(sz.width,0,sz.width/2.0,sz.height),
                       flip(marsRight));
UIImage* im = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
CGImageRelease(marsLeft); CGImageRelease(marsRight);


{% endhighlight %}

另外，我们也可以在使用UIImage包围CGImage，并使用UIImage绘制而不是CGImage。UIImage可以调用
`imageWithCGImage:scale:orientation:`以指定正确的scale，而且没有flipping问题。

{% highlight objc %}

UIImage* mars = [UIImage imageNamed:@"Mars"];
CGSize sz = mars.size;
// Derive CGImage and use its dimensions to extract its halves
CGImageRef marsCG = [mars CGImage];
CGSize szCG = CGSizeMake(CGImageGetWidth(marsCG), CGImageGetHeight(marsCG));
CGImageRef marsLeft =
	CGImageCreateWithImageInRect(
            marsCG, CGRectMake(0,0,szCG.width/2.0,szCG.height));
CGImageRef marsRight =
    CGImageCreateWithImageInRect(
         marsCG, CGRectMake(szCG.width/2.0,0,szCG.width/2.0,szCG.height));
UIGraphicsBeginImageContextWithOptions(
        CGSizeMake(sz.width*1.5, sz.height), NO, 0);
[[UIImage imageWithCGImage:marsLeft
                         scale:mars.scale
                   orientation:UIImageOrientationUp]
     drawAtPoint:CGPointMake(0,0)];
[[UIImage imageWithCGImage:marsRight
                         scale:mars.scale
                   orientation:UIImageOrientationUp]
     drawAtPoint:CGPointMake(sz.width,0)];
UIImage* im = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
CGImageRelease(marsLeft); CGImageRelease(marsRight);

{% endhighlight %}

还有一种解决方法就是在CGImage绘制之前，给graphic context应用transform，这可以flipping context
的内部坐标系。这种方法简单，但是在应用其他transform时会有冲突。

* 为什么会发生flipping

因为Core Graphic来自于OS X，而OS X的坐标系原点在左下角，y坐标从下到上，而iOS的坐标原点在右上角，y
坐标从上到下。在大多数绘制场景中，graphic context坐标系会自动补偿，因此在graphic context中绘制时
坐标原点在你期望的左上角。但是在CGImage中绘制时，揭露了两个世界的不同。

## 镜像(Snapshots)

一个完整的view--从一个按钮到它的整个界面，以及它包含的完整view树--都可以在当前graphics context中使用
UIView的`drawViewHierarchyInRect:afterScreenUpdates:`方法绘制。这是iOS7中的新方法，比以前的
CALayer的`renderInContext:`方法更快。它返回一个类型为bitmap的view镜像。

我们可以使用镜像创建毛玻璃效果：

{% highlight objc %}
UIGraphicsBeginImageContextWithOptions(vc1.view.frame.size, YES, 0);
[vc1.view drawViewHierarchyInRect: vc1.view.frame afterScreenUpdates:NO];
 UIImage* im = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();

{% endhighlight %}

上面的代码得到view的镜像，然后将得到的图形模糊化，可以使用CIFilter完成，也可以使用更快的苹果提供的
Blurring and Tinting an Image的方法。

你也可以使用更快的UIView或UIScreen的方法`snapshotViewAfterScreenUpdates:`得到一个类型为UIView
的镜像，如果你想得到一个就像resizable image的镜像，使用
`resizableSnapshotViewFromRect:afterScreenSnapshots
Updates:withCapInsets:`方法。

## CIFilter 和 CIImage

CIFilter包含如下几个类别：

* 摹刻和渐变(Patterns and gradients)
* 影像合成(Compositing)
* 变色(Color)
* 几何失真(Geometric)
* 变换(Transformation)
* 过渡(Transition)
* 特殊功能(Special purpose)

使用`filterWithName:`或`filterNamesInCategories:`创建CIFilter，使用`setValue:forKey:`设置值，
或者使用`filterWithName:keyAndValues:`创建filter时指定值。CIFilter操作输入的CIImage，然后
输出一个CIImage，你可以通过CIImage的`initWithCGImage:`获得一个CIImage，也可以通过UIImage的
CIImage属性来获得一个CIImage。

当你在使用任何filter的时候，实际上没有进行任何操作，计算发生在你将最后的CIImage使用bitmap绘制时，你可以
使用下面的方式绘制：

* 通过`contextWithOptions:`创建CIContext，然后调用`createCGImage:fromRect:`处理最后得到的CIImage。
需要注意的是，这儿的CIImage没有frame或bound，它只有extent，你经常将其作为`createCGImage:fromRect:`的
第二个参数。最后输出的CGImage，你可以用于任何目的，比如显示在app上，转换为UIImage，或者更深层次的绘制。

* 通过下面的方法直接从CIImage创建UIImage：
— imageWithCIImage:
— initWithCIImage:
— imageWithCIImage:scale:orientation:
— initWithCIImage:scale:orientation:
然后你必须在graphic context中绘制，否则得到的CIImage并没有转换为bitmap。因此，`imageWithCIImage:`
得到的UIImage不能直接用于在UIImageView的中显示。这种方法通常用于绘制，而不是显示。

下面的代码使用第一种方法：

{% highlight objc %}
UIImage* moi = [UIImage imageNamed:@"Moi"];
CIImage* moi2 = [[CIImage alloc] initWithCGImage:moi.CGImage];
CGRect moiextent = moi2.extent;
// first filter
CIFilter* grad = [CIFilter filterWithName:@"CIRadialGradient"];
CIVector* center = [CIVector vectorWithX:moiextent.size.width/2.0
                                           Y:moiextent.size.height/2.0];
[grad setValue:center forKey:@"inputCenter"];
[grad setValue:@85 forKey:@"inputRadius0"];
[grad setValue:@100 forKey:@"inputRadius1"];
CIImage *gradimage = [grad valueForKey: @"outputImage"];
// second filter
CIFilter* blend = [CIFilter filterWithName:@"CIBlendWithMask"];
[blend setValue:moi2 forKey:@"inputImage"];
[blend setValue:gradimage forKey:@"inputMaskImage"];
// extract a bitmap
CGImageRef moi3 =
[[CIContext contextWithOptions:nil]
createCGImage:blend.outputImage
fromRect:moiextent];
UIImage* moi4 = [UIImage imageWithCGImage:moi3];
CGImageRelease(moi3);

{% endhighlight %}

如果采用第二种方法，代码如下：

{% highlight objc %}

UIGraphicsBeginImageContextWithOptions(moiextent.size, NO, 0);
[[UIImage imageWithCIImage:blend.outputImage] drawInRect:moiextent];
UIImage* moi4 = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();

{% endhighlight %}

##绘制UIView

重写UIView的`drawRect:`可以绘制UIView，所有在这个函数里面绘制的都会显示在这个View
中。当子类化一个UIView时，你会发现背景是黑色的，这是因为当下面的两个条件都满足：

* backgroundColor为nil
* opaque为YES

如果你不需要黑色的背景，改变其中一个。

## graphic context 设置

当在graphic context中绘制时，会使用整个context的设置，当你想改变设置时，你可以先使用`CGContextSaveGState`
保存当前的设置，以后再使用`CGContextRestoreGState`恢复整个设置。设置选项如下：

* Line thickness and dash style
CGContextSetLineWidth, CGContextSetLineDash (and UIBezierPath lineWidth, setLineDash:count:phase:)

* Line end-cap style and join style
CGContextSetLineCap, CGContextSetLineJoin, CGContextSetMiterLimit (and UIBezierPath 
lineCapStyle, lineJoinStyle, miterLimit)

* Line color or pattern
CGContextSetRGBStrokeColor, CGContextSetGrayStrokeColor, CGContextSet- StrokeColorWithColor, 
CGContextSetStrokePattern (and UIColor setStroke)

* Fill color or pattern
CGContextSetRGBFillColor, CGContextSetGrayFillColor, CGContextSetFill- ColorWithColor, 
CGContextSetFillPattern (and UIColor setFill)

* Shadow
CGContextSetShadow, CGContextSetShadowWithColor 

* Overall transparency and compositing
CGContextSetAlpha, CGContextSetBlendMode 

* Anti-aliasing
CGContextSetShouldAntialias


* Clipping area
Drawing outside the clipping area is not physically drawn.

* Transform (or “CTM,” for “current transform matrix”)
Changes how points that you specify in subsequent drawing commands are mapped onto the 
physical space of the canvas.

## 曲线和图像

使用下面方法绘制曲线：

* Position the current point:
CGContextMoveToPoint

* Trace a line:
CGContextAddLineToPoint, CGContextAddLines 

* Trace a rectangle:
CGContextAddRect, CGContextAddRects 

* Trace an ellipse or circle:
CGContextAddEllipseInRect

* Trace an arc:
CGContextAddArcToPoint, CGContextAddArc 

* Trace a Bezier curve with one or two control points:
CGContextAddQuadCurveToPoint, CGContextAddCurveToPoint

* Close the current path:
CGContextClosePath. This appends a line from the last point of the path to the first point. 
There’s no need to do this if you’re about to fill the path, since it’s done for you.

* Stroke or fill the current path:
CGContextStrokePath, CGContextFillPath, CGContextEOFillPath, CGContextDrawPath.
 Stroking or filling the current path clears the path. Use CGContextDraw- Path if you want 
both to fill and to stroke the path in a single command, because if you merely stroke it 
first with CGContextStrokePath, the path is cleared and you can no longer fill it. There 
are also a lot of convenience functions that create a path and stroke or fill it all in 
a single move:
1. CGContextStrokeLineSegments 
2. CGContextStrokeRect
3. CGContextStrokeRectWithWidth 
4. CGContextFillRect
5. CGContextFillRects
6. CGContextStrokeEllipseInRect 
7. CGContextFillEllipseInRect

如果你想开始新的绘制，你可以使用CGContextBeginPath，下面的代码演示如何创建一个向上的箭头。

{% highlight objc %}
// obtain the current graphics context
CGContextRef con = UIGraphicsGetCurrentContext();
// draw a black (by default) vertical line, the shaft of the arrow
CGContextMoveToPoint(con, 100, 100);
CGContextAddLineToPoint(con, 100, 19);
CGContextSetLineWidth(con, 20);
CGContextStrokePath(con);
// draw a red triangle, the point of the arrow
CGContextSetFillColorWithColor(con, [[UIColor redColor] CGColor]);
CGContextMoveToPoint(con, 80, 25);
CGContextAddLineToPoint(con, 100, 0);
CGContextAddLineToPoint(con, 120, 25);
CGContextFillPath(con);
// snip a triangle out of the shaft by drawing in Clear blend mode
CGContextMoveToPoint(con, 90, 101);
CGContextAddLineToPoint(con, 100, 90);
CGContextAddLineToPoint(con, 110, 101);
CGContextSetBlendMode(con, kCGBlendModeClear);
CGContextFillPath(con);
{% endhighlight %}

如果你想重用或分享一个曲线，你可以将它封装为一个CGPath，实际上是一个CGPathRef。你可以使用
CGMutablePathRef然后将一系列的CGPath加入其中，你也可以使用`CGContextCopyPath`复制当前
的path。下面的方法用于创建path，基于已经存在的path：

* CGPathCreateWithRect
* CGPathCreateWithEllipseInRect
* CGPathCreateCopyByStrokingPath
* CGPathCreateCopyByDashingPath
* CGPathCreateCopyByTransformingPath

UIKit的UIBezierPath封装了CGPath，提供了方法创建Path：

* moveToPoint:
* addLineToPoint:
* bezierPathWithRect:
* bezierPathWithOvalInRect:
* addArcWithCenter:radius:startAngle:endAngle:clockwise: 
* addQuadCurveToPoint:controlPoint:
* addCurveToPoint:controlPoint1:controlPoint2:
* closePath

UIBezierPath也提供了一个非常有用的方法`bezierPathWithRoundedRect:cornerRadius:`创建圆角矩形。
使用UIBezierPath创建同样的向上箭头：
{% highlight objc %}
// shaft of the arrow
UIBezierPath* p = [UIBezierPath bezierPath];
[p moveToPoint:CGPointMake(100,100)];
[p addLineToPoint:CGPointMake(100, 19)];
[p setLineWidth:20];
[p stroke];
// point of the arrow
[[UIColor redColor] set];
[p removeAllPoints];
[p moveToPoint:CGPointMake(80,25)];
[p addLineToPoint:CGPointMake(100, 0)];
[p addLineToPoint:CGPointMake(120, 25)];
[p fill];
// snip out triangle in the tail
[p removeAllPoints];
[p moveToPoint:CGPointMake(90,101)];
[p addLineToPoint:CGPointMake(100, 90)];
[p addLineToPoint:CGPointMake(110, 101)];
[p fillWithBlendMode:kCGBlendModeClear alpha:1.0];
{% endhighlight %}

## Clipping

将一部分区域保护起来，使这部分区域将来不会被绘制，这被称为Clipping。Clipping是整个Graphic context的
特性，任何新的Clipping都会和以前的区域交叉应用。如果你想应用自己单独的Clipping，然后应用完成后删除它，
可以包含在`CGContextSaveGState`和`CGContextRestoreGState`之间。

填充组合Path或用它指示一个Clipping区域时，系统遵循下面两种规则之一：

* Winding rule

填充或Clipping区域被顺时针或逆时针方向分隔

* Even-odd rule (EO)

填充或Clipping区域被简单的计数分隔

你可以在改变Clipping区域之前使用`CGContextGetClipBoundingBox`获得Graphic Context的bounds。下面的
代码演示如何使用Clipping绘制相同的箭头:

{% highlight objc %}

// obtain the current graphics context
CGContextRef con = UIGraphicsGetCurrentContext();
// punch triangular hole in context clipping region
CGContextMoveToPoint(con, 90, 100);
CGContextAddLineToPoint(con, 100, 90);
CGContextAddLineToPoint(con, 110, 100);
CGContextClosePath(con);
CGContextAddRect(con, CGContextGetClipBoundingBox(con));
CGContextEOClip(con);
// draw the vertical line
CGContextMoveToPoint(con, 100, 100);
CGContextAddLineToPoint(con, 100, 19);
CGContextSetLineWidth(con, 20);
CGContextStrokePath(con);
// draw the red triangle, the point of the arrow
CGContextSetFillColorWithColor(con, [[UIColor redColor] CGColor]);
CGContextMoveToPoint(con, 80, 25);
CGContextAddLineToPoint(con, 100, 0);
CGContextAddLineToPoint(con, 120, 25);
CGContextFillPath(con);

{% endhighlight %}

UIBezierPath使用`usesEvenOddFillRule`和`addClip`方法来使用Clipping。

## 渐变

## Colors and Patterns

{% highlight objc %}

// obtain the current graphics context
CGContextRef con = UIGraphicsGetCurrentContext();
CGContextSaveGState(con);
// punch triangular hole in context clipping region
CGContextMoveToPoint(con, 90, 100);
CGContextAddLineToPoint(con, 100, 90);
CGContextAddLineToPoint(con, 110, 100);
CGContextClosePath(con);
CGContextAddRect(con, CGContextGetClipBoundingBox(con));
CGContextEOClip(con);
// draw the vertical line, add its shape to the clipping region
CGContextMoveToPoint(con, 100, 100);
CGContextAddLineToPoint(con, 100, 19);
CGContextSetLineWidth(con, 20);
CGContextReplacePathWithStrokedPath(con);
CGContextClip(con);
// draw the gradient
CGFloat locs[3] = { 0.0, 0.5, 1.0 };
CGFloat colors[12] = {
        0.3,0.3,0.3,0.8, // starting color, transparent gray
        0.0,0.0,0.0,1.0, // intermediate color, black
        0.3,0.3,0.3,0.8 // ending color, transparent gray
    };
CGColorSpaceRef sp = CGColorSpaceCreateDeviceGray();
CGGradientRef grad =
        CGGradientCreateWithColorComponents (sp, colors, locs, 3);
CGContextDrawLinearGradient (
        con, grad, CGPointMake(89,0), CGPointMake(111,0), 0);
CGColorSpaceRelease(sp);
CGGradientRelease(grad);
CGContextRestoreGState(con); // done clipping
// draw the red triangle, the point of the arrow
CGContextSetFillColorWithColor(con, [[UIColor redColor] CGColor]);
CGContextMoveToPoint(con, 80, 25);
CGContextAddLineToPoint(con, 100, 0);
CGContextAddLineToPoint(con, 120, 25);
CGContextFillPath(con);

{% endhighlight %}

## Graphics Context Transforms

## Shadows

## Erasing

## Points and Pixels

## Content Mode

{% highlight objc %}

{% endhighlight %}