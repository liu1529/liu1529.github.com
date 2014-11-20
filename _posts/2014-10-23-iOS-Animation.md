---
title: iOS Animation
layout: post
tags: [iOS,Animation]
comments: true
---

动画是iOS里面重要的组成部分，当你设置某些属性时可以启动动画，比如你设置view的背景从黑变为红。如果你想为一个
游戏应用添加不间断的动画，使用iOS7加入的`Sprite Kit`，这儿不讨论它。

# 绘制，动画，线程
当你改变view的可视属性（如`backgroundColor`,`hidden`,`alpha`等），这些变化不会现在发生。系统会记下这些
发生的变化，标记这个view需要重绘。当你代码完成，系统再等一段时间后，会重绘所有需要重绘的view。我们把这个称为
"重绘时刻(redraw moment)"。

你在同一段代码改变view的可见属性，然后再改回来，在屏幕上不会发生任何变化。系统在你代码完成后，过一段时间才会发生
"重绘时刻"，因此这期间的代码不管运行多长时间，屏幕也不会发生变化。

{% highlight objc %}

// view starts out green
view.backgroundColor = [UIColor redColor];
// ... time-consuming code goes here ...
view.backgroundColor = [UIColor greenColor];
// code ends, redraw moment arrives

{% endhighlight %}

这是你不需要关心view重绘顺序的原因，即使你调用`setNeedsDisplay`，告诉下一个"重绘时刻"这个view需要重绘。

同样，动画在下一个"重绘时刻"才会被执行(你可以强制现在执行，但是这不常见)，当你将一个view从点1移动到点2时，
你可以想象发生了如下事情：

1. view被设置到点2，但是"重绘时刻"没有到达，view还在点1位置。
2. 你的代码完成
3. "重绘时刻"到达，如果没有动画，view立即移动到点2绘制。这儿有动画，动画效果开始于点1，所以view还在点1。
4. 动画执行，view在点1和点2之间绘制，我们把它称为动画`infight`。
5. 动画结束，view现在在点2位置绘制。
6. 动画效果消失，view现在出现在点2位置。

动画在另一个线程中执行，你不需要关心这些细节，但是你需要准备进行线程之间的通信。

* Presentation Layer：

其实没有真实的动画效果在屏幕前面，虽然看起来如此。实际上，动画执行时不是layer本身在屏幕上绘制，而是其presentation layer
在屏幕上面绘制。因此，当你动画改变layer的位置从点1到点2时，名义上的位置已经立即发生了改变；与此同时，presentation layer
并没有发生改变，直到重绘时刻到达，然后开始动画改变，用户实际上看见的是presentation layer绘制。

layer的presentation layer可以通过`presentationLayer`属性访问（layer本身通过`modelLayer`属性访问），它的类型是id，
因此你可能需要强制转换成CALayer来使用它。访问`presentationLayer`是不常见的，但是在动画过程中获得当前的状态时，可能会用到。

当动画在执行时，界面默认接受用户的触控，这时可能会导致程序出错。你需要在动画执行时关闭界面触控，在动画结束后再打开。
因为你的代码可以在动画开始后被执行，或在动画执行过程中开始一个新的动画，所以你需要小心处理防止动画冲突。多个动画可以
被同时开始和执行，但是当开始一个正在执行的动画或改变一个正在执行的动画的属性，会导致动画的不连贯，然后这个动画会停止。
有时你会使用这种方法停止一个动画，但是通常你必须小心注意不要这么做。

外部事件会打断你的动画，比如动画执行时用户按下home键或有电话打进来，系统会简单的取消你的所有正在执行的动画。由于你已经
设置好动画完成后正确的状态，所以你的程序状态还是正确的。如果你的app需要在动画中间出现，需要比较高级的代码来完成。

# UIImageView和UIImage动画

UIImageView提供了一个简单的动画效果，通过属性`animationImages`或`highlightedAnimationImages`指定动画的图片，
通过`animationDuration`指定间隔时间，通过`animationRepeatCount`指定重复次数。通过`startAnimating`方法开始动画，
通过`stopAnimating`方法停止动画。动画开始和结束后显示`image`或`highlightedImage`属性图片。

{% highlight objc %}

UIImage* mars = [UIImage imageNamed: @"Mars"];
UIGraphicsBeginImageContextWithOptions(mars.size, NO, 0);
UIImage* empty = UIGraphicsGetImageFromCurrentImageContext();
UIGraphicsEndImageContext();
NSArray* arr = @[mars, empty, mars, empty, mars];
UIImageView* iv = [[UIImageView alloc] initWithImage:empty];
CGRect r = iv.frame;
r.origin = CGPointMake(100,100);
iv.frame = r;
[self.view addSubview: iv];
iv.animationImages = arr;
iv.animationDuration = 2;
iv.animationRepeatCount = 1;
[iv startAnimating];

{% endhighlight %}

你可以通过UIImage的方法创建变为Animated UIImage：

* animatedImageWithImages:duration:
* animatedImageNamed:duration:
* animatedResizableImageNamed:capInsets:resizingMode:duration:

你不能控制Animated UIImage何时开始和结束，你也不能控制它的重复次数。Animated UIImage一旦出现在界面中，
就会一直执行动画效果，要想控制它只有在界面中删除或替换它，Animated UIImage可以出现在任何UIImage可以出现
的地方。

{% highlight objc %}

 NSMutableArray* arr = [NSMutableArray array];
float w = 18;
for (int i = 0; i < 6; i++) {
    UIGraphicsBeginImageContextWithOptions(CGSizeMake(w,w), NO, 0);
    CGContextRef con = UIGraphicsGetCurrentContext();
    CGContextSetFillColorWithColor(con, [UIColor redColor].CGColor);
    CGContextAddEllipseInRect(con, CGRectMake(0+i,0+i,w-i*2,w-i*2));
    CGContextFillPath(con);
    UIImage* im = UIGraphicsGetImageFromCurrentImageContext();
    UIGraphicsEndImageContext();
    [arr addObject:im];
}
UIImage* im = [UIImage animatedImageWithImages:arr duration:0.5];
// assume self.b is a button in the interface
[self.b setImage:im forState:UIControlStateNormal];

{% endhighlight %}


# View动画

动画实际上一个layer动画，view的`alpha`,`backgroundColor`,`bounds`,`center`,`frame`,`transform`
都可以直接有动画效果，改变view的内容也可以有动画效果。


## 基于Block的view动画

可以通过UIView的方法，在block中添加动画。比如，如果self.v的背景色是黄色，现在想要动画变成红色：

{% highlight objc %}

[UIView animateWithDuration:0.4 animations:^{
        self.v.backgroundColor = [UIColor redColor];
}];

{% endhighlight %}

> 也可使用老的方法`beginAnimations:context:`和`commitAnimations`，但是Apple不鼓励使用它们。

所有在block中的可动画改变都会发生动画，所有在同一个block中，你可以同时改变view的颜色和位置，也可以包含
多个view的改变。

{% highlight objc %}

UIView* v2 = // ... create and configure new view
v2.alpha = 0;
[self.v.superview addSubview:v2];
[UIView animateWithDuration:0.4 animations:^{
     self.v.alpha = 0;
    v2.alpha = 1;
} completion:^(BOOL finished) {
     [self.v removeFromSuperview];
}];

{% endhighlight %}

我们可以使用`performWithoutAnimation:`的block改变可动画的属性，使其没有动画效果。

{% highlight objc %}

[UIView animateWithDuration:0.4 animations:^{
    self.v.backgroundColor = [UIColor redColor];
    [UIView performWithoutAnimation:^{
        CGPoint p = self.v.center;
        p.y -= 100;
        self.v.center = p;
	}]; 
}];

{% endhighlight %}

在`animations:`block中的动画是有序的，但是在`performWithoutAnimation:`block中的没有。
因此在block中你改变一个可动画的属性后，随后你改变它，否则会导致冲突：

{% highlight objc %}
[UIView animateWithDuration:2 animations:^{
     CGPoint p = self.v.center;
     p.y = 100;
    self.v.center = p;
     CGPoint p2 = self.v.center;
     p2.y = 300;
     self.v.center = p2;
}];

{% endhighlight %}

甚至，在block后面改变可动画属性也会导致冲突：

{% highlight objc %}

[UIView animateWithDuration:2 animations:^{
     CGPoint p = self.v.center;
     p.y = 100;
    self.v.center = p;
}];
CGPoint p2 = self.v.center;
p2.y = 300;
self.v.center = p2;

{% endhighlight %}

因此，当你在`animations:`block中改变可动画属性后，不要再次改变它，除非这个动画完成。

## view动画选项

完整的动画方法是`animateWithDuration:delay:options:animations:completion:`，
指定动画的次数，并不容易实现，这儿有一个方法通过递归实现：

{% highlight objc %}

- (void) animate: (int) count {
    CGPoint pOrig = self.v.center;
    NSUInteger opts = UIViewAnimationOptionAutoreverse;
    [UIView animateWithDuration:1 delay:0 options:opts animations:^{
          CGPoint p = self.v.center;
        	p.x += 100;
           self.v.center = p;
    } completion:^(BOOL finished) {
        self.v.center = pOrig;
        if (count)
            [self animate:count-1];
	}]; 
}

{% endhighlight %}

使用递归方法是危险的，但是在次数比较小的情况下是安全的。

某些选项可以指定当动画已经添加或发生时，动画怎么进行：

* UIViewAnimationOptionBeginFromCurrentState
如果动画已经添加或正在进行，通过使用presentation layer决定动画从何开始，如果可能，和前面的动画合并。

* UIViewAnimationOptionOverrideInheritedDuration
阻止从周围或正在进行的动画中继承持续时间

* UIViewAnimationOptionOverrideInheritedCurve
阻止从周围或正在进行的动画中继承动画曲线

{% highlight objc %}

[UIView animateWithDuration:1 animations:^{
    CGPoint p = self.v.center;
    p.x += 100;
    self.v.center = p;
}];
NSUInteger opts = 0;
[UIView animateWithDuration:1 delay:0 options:opts animations:^{
     CGPoint p = self.v.center;
    p.x = 0;
    self.v.center = p;
} completion:nil];

{% endhighlight %}

上面代码中，如果opts为0，系统将会取消第一个动画，view直接先向右跳100，再动画向左移动到0的位置。
而如果opts为UIViewAnimationOptionBeginFromCurrentState，那么view将组合两个动画，先动画
向右移动，再动画向左移动。

UIViewAnimationOptionOverrideInheritedDuration使用在动画的嵌套过程中。

{% highlight objc %}

[UIView animateWithDuration:2 animations:^{
    CGPoint p = self.v.center;
    p.x += 100;
    self.v.center = p;
    NSUInteger opts = 0;
    [UIView animateWithDuration:0.5 delay:0 options:opts animations:^{
        self.v.backgroundColor = [UIColor blackColor];
    } completion:nil];
}];

{% endhighlight %}

上面代码中，如果opts为0，那么颜色变化的动画的持续时间为2s，而如果opts为UIViewAnimationOptionOverrideInheritedDuration，
那么颜色变化动画将有自己的持续时间0.5s。

在UIView层取消一个正在进行的动画是困难的（但是在CALayer层确很简单），我们可以再添加一个动画，
使新添加的动画和原来的动画冲突，这样就可以取消第一个动画，但是得保证第二个动画对第一个对话的
最终状态没有影响。

{% highlight objc %}

-(void) animate {
     CGPoint p = self.v.center;
     p.x += 100;
     self.pFinal = p;
      [UIView animateWithDuration:4 animations:^{
          self.v.center = p;
     }];
}
-(void) cancel {
    [UIView animateWithDuration:0 animations:^{
        CGPoint p = self.pFinal;
         p.x += 1;
        self.v.center = p;
     } completion:^(BOOL finished) {
          CGPoint p = self.pFinal;
         self.v.center = p;
	}]; 
}

{% endhighlight %}

我们使用同样的技巧取消重复进行的动画

{% highlight objc %}

-(void) animate {
     CGPoint p = self.v.center;
    self.pOrig = p;
    p.x += 100;
    NSUInteger opts = UIViewAnimationOptionAutoreverse |
    UIViewAnimationOptionRepeat;
    [UIView animateWithDuration:1 delay:0 options:opts animations:^{
           self.v.center = p;
    } completion: nil];
}
-(void) cancel {
    [UIView animateWithDuration:0 animations:^{
        self.v.center = self.pOrig;
}]; }

{% endhighlight %}

## Springing view 动画

在iOS7中，可以使用spring使动画在最终点摆动

{% highlight objc %}

[UIView animateWithDuration:0.8 delay:0
    usingSpringWithDamping:0.7 initialSpringVelocity:0
                     options:0 animations:^{
    CGPoint p = self.v.center;
	p.y += 100;
    self.v.center = p;
} completion:nil];

{% endhighlight %}

## Keyframe view 动画

在iOS7中，你可以通过keyframe制定一系列动画执行顺序（在以前只能在CALayer层做这些事情），
你可以调用`animateKeyframesWithDuration:...`，然后在其block中多次使用`addKeyframe...`
添加动画的不同阶段。每个阶段的开始时间和持续时间都小于1，相对于整个动画时间。

{% highlight objc %}

__block CGPoint p = self.v.center;
[UIView animateKeyframesWithDuration:4 delay:0 options:0 animations:^{
	self.v.alpha = 0;
    [UIView addKeyframeWithRelativeStartTime:0 relativeDuration:.25 animations:^{
        p.x += 100;
        p.y += 50;
        self.v.center = p;
    }];
    [UIView addKeyframeWithRelativeStartTime:.25 relativeDuration:.25 animations:^{
        p.x -= 100;
        p.y += 50;
        self.v.center = p;
    }];
    [UIView addKeyframeWithRelativeStartTime:.5 relativeDuration:.25 animations:^{
         p.x += 100;
         p.y += 50;
        self.v.center = p;
    }];
    [UIView addKeyframeWithRelativeStartTime:.75 relativeDuration:.25 animations:^{
        p.x -= 100;
        p.y += 50;
        self.v.center = p;
    }];
    } completion:nil];

{% endhighlight %}

## Transitions

Transition用于强调view的内容变化，可以使用下面的两种方法创建：

* transitionWithView:duration:options:animations:completion:
* transitionFromView:toView:duration:options:completion:

下面的例子中，将UIImageView的内容变化为一个笑脸：

{% highlight objc %}

[UIView transitionWithView:self.iv duration:0.8
        options:UIViewAnimationOptionTransitionFlipFromLeft animations:^{
    self.iv.image = [UIImage imageNamed:@"Smiley"];
} completion:nil];

{% endhighlight %}

在例子中，我将内容的变化放在有了`animations`的block中，这是可行的但是会给人错误的提示，
实际上如果是内容的变化，不需要放在block中，它可以放在任何地方，在整个代码的前面或后面都是
可行的。你可能在这儿使用	`animations`的block放置其他动画，比如改变view的center。

在transition中，默认情况下，会提前准备一个最终情况下的快照，如果你想在transition中
启动绘制，你可以使用`UIViewAnimationOptionAllowAnimatedContent`选项。

`transitionFromView:toView:duration:options:completion: `使用第二个view替代第一个
view，并使他们的superview发生transition动画。取决于你设置的选项，有两种不同的配置：

* 删除subview，添加另一个subview

当你未使用UIViewAnimationOptionShowHideTransitionViews选项时，且第二个subview和第一个
subview不在同一个view层次结构中，那么第一个subview将它的superview中删除，添加到第二个subview
的superview中。

* 隐藏subview，显示另一个subview

当使用UIViewAnimationOptionShowHideTransitionViews选项，且第一个和第二个subview在同一个
view层次结构中，那么设置第一个view的hidden为YES，第二个view的hidden为NO。


# Implicit Layer动画

## 动画处理

## Media Timing 功能

# 核心动画(Core Animation)

## CABasicAnimation和它的层次结构

## 使用CABasicAnimation

## Keyframe 动画

## 使属性产生动画效果

## 动画分组

## Transitions

## 动画列表

# 动作(Actions)

## Emitter Layers

## CIFilter Transitions

## UIKit Dynamics

## Motion Effects

## 动画和自动布局



{% highlight objc %}

{% endhighlight %}