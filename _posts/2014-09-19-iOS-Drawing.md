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

{% highlight objc %}

{% endhighlight %}