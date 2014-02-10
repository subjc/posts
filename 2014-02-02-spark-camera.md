---
layout: post
title:  "Spark Camera's recording meter"
date:   2014-02-02 21:02:31
description: "Just when you thought we'd seen every take on a camera app, Spark Camera comes along to prove us all wrong."
image: "/media/2014-02-02-spark-camera/images/title-image.png"
video: true
video_mp4: "/media/2014-02-02-spark-camera/video/title-video.m4v"
---

## Foreword

There's no denying that camera apps are in vogue. At the time of writing, a third of the apps in the Best New Apps section of the App Store were in the Photo & Video category and with good reason, camera apps are a fantastic way to express creativity for both the budding filmmaker and the developer behind them. We're going to be looking [Spark Camera](https://www.sparkcamera.com/), one of the standouts of great interface and awesome user experience. 

[Spark Camera](https://www.sparkcamera.com/) was built by design firm [IDEO](http://www.ideo.com/) (pronounced *“eye-dee-oh”*) who have a pretty [illustrious history](http://www.ideo.com/work/mouse-for-apple/). It has an elegant and simple look which combines form and function. In particular we're going to be dissecting the circular progress view which, in one simple circle of colour, displays everything you need to know about your recording.

## Dissecting

To get a feel for what's going on behind the scenes, we're going to use a [neat trick](http://petersteinberger.com/blog/2013/how-to-inspect-the-view-hierarchy-of-3rd-party-apps/) where we can inject the Reveal library[^1] into any third party app running on a device. 

![Spark Camera Revealed](/media/2014-02-02-spark-camera/images/spark-revealed.png)

##Thinking

We can see that the view hierarchy is as minimal as the interface. The view we're looking at building (<code>CaptureProgressView</code>) has one subview (<code>RecordingTimeIndicatorView</code>) which in turn has a <code>UILabel</code> and <code>UIView</code>. This tells us that the circular progress view is likely composed of one or more <code>CALayers</code> with their contents drawn rather than subviews. 

We can also see that there's no great big <code>UIButton</code> added to over the top so we're likely looking to use a <code>UIGestureRecognizer</code> or overriding <code>touchesBegan:WithEvent:</code> and <code>touchesEnded:WithEvent:</code> on the view to start and stop the recording. 

## Laying the foundations

To kick things off, we're going to create a subclass of <code>UIView</code> and call it something meaningful (in this case <code>RecordingCircleOverlayView</code>). 

We can see that we'll need a light grey circle to display as the "track" for the progress, so we'll need to create a <code>CAShapeLayer</code> and provide it a <code>CGPathRef</code> for the circle. To create the <code>CGPathRef</code>, we need to first create a <code>UIBezierPath</code> for the circle using the method <code>bezierPathWithArcCenter:radius:startAngle:endAngle:clockwise:</code>

To provide the coloured segments that represent the recording progress we'll likely be using additional <code>CAShapeLayers</code> with the same <code>CGPathRef</code> as the background layer, so we can go ahead and store this <code>UIBezierPath</code> as a property so we don't have to recreate it each time. 

{% highlight objc %}
	   CGPoint arcCenter = CGPointMake(CGRectGetMidY(self.bounds), CGRectGetMidX(self.bounds));
	CGFloat radius = CGRectGetMidX(self.bounds) - insets.top - insets.bottom;

	self.circlePath = [UIBezierPath bezierPathWithArcCenter:arcCenter
                                                 	 radius:radius
                                                 startAngle:M_PI
                                            	   endAngle:-M_PI
                                            	  clockwise:NO];
{% endhighlight %}

You may have noticed we're creating the <code>UIBezierPath</code> to be drawn anti-clockwise from the startAngle: of <code>M_PI</code> and <code>endAngle:</code> of <code>-M_PI</code>. This is to match the behaviour of [Spark Camera](https://www.sparkcamera.com/) where the recording progress starts at 270° and moves counter clockwise around the circle. 

Now that we have the <code>UIBezierPath</code> created, we can create the <code>CAShapeLayer</code> that'll provide the background circle and pass it the <code>CGPathRef</code> we get from the <code>UIBezierPath</code>.

{% highlight objc %}
    CAShapeLayer *backgroundLayer = [CAShapeLayerlayer];
    backgroundLayer.path = self.circlePath.CGPath;
    backgroundLayer.strokeColor = [[UIColor lightGrayColor] CGColor];
    backgroundLayer.fillColor = [[UIColorclearColor] CGColor];
    backgroundLayer.lineWidth = self.strokeWidth;
{% endhighlight %}

Then we just add it as a sublayer of our <code>RecordingCircleOverlayView</code>'s layer.

{% highlight objc %}
    [self.layer addSublayer:backgroundLayer];
{% endhighlight %}

If we build and run now we'll see that we've got our fancy light grey circle sitting in the middle of our view.

![Fancy circle](/media/2014-02-02-spark-camera/images/empty-circle.png)

Now we're going to need a way of starting and stopping the progress circle. If we look back at [Spark Camera](https://www.sparkcamera.com/), to start recording we need to touch down with our finger and top pause we just let go. This rules out <code>UITapGestureRecognizer</code> as it fires on <code>UIControlEventTouchUpInside</code>, meaning we'd have to touch down and touch up before we register an event. 

While we could use a <code>UIButton</code> and the control event <code>UIControlEventTouchDown</code>, we've seen in the dissection with Reveal that [Spark Camera](https://www.sparkcamera.com/) doesn't do this, so why should we. Instead we'll tap into the power of <code>UIResponder</code> (of which <code>UIView</code> is a subclass) and override <code>touchesBegan:WithEvent:</code> and <code>touchesEnded:WithEvent:</code> on our <code>UIView</code> subclass, <code>RecordingCircleOverlayView</code>.

Now that we've got a way of starting and stopping the progress circle, we need to start drawing the progress.

To display the progress segments we'll be using the same technique that we used to display the background circle layer, but with a slight twist. To give the appearance of the segment growing over time, we're going to animate the <code>strokeEnd</code> property on <code>CAShapeLayer</code>[^2]. We can use our circle <code>UIBezierPath</code> that we stored away earlier to create a full circle for the segment and use <code>strokeEnd</code> to draw only the portion of the segment that has elapsed.

{% highlight objc %}
    CAShapeLayer *progressLayer = [CAShapeLayerlayer];
    progressLayer.path = self.circlePath.CGPath;
    progressLayer.strokeColor = [[selfrandomColor] CGColor];
    progressLayer.fillColor = [[UIColorclearColor] CGColor];
    progressLayer.lineWidth = self.strokeWidth;
    progressLayer.strokeEnd = 0.f;
{% endhighlight %}

We set the <code>strokeEnd</code> to 0 initially so that we can animate it later on. 

Then we add it as a sublayer of our <code>RecordingCircleOverlayView</code>'s layer.

{% highlight objc %}
    [self.layeraddSublayer:progressLayer];
{% endhighlight %}

.. but we also want to keep a reference to it so we can animate the <code>strokeEnd</code> property. We also know that we could potentially be using multiple <code>CAShapeLayers</code> (one for each progress segment) so storing each segment in it's own property wouldn't be feasible. Instead we'll create an <code>NSMutableArray</code> property on our <code>RecordingCircleOverlayView</code> and add each of our progress segment layers to that. 

{% highlight objc %}
    [self.progressLayersaddObject:progressLayer];
{% endhighlight %}

.. but we can also see that we might need a reference to the current segment. If we look back to [Spark Camera](https://www.sparkcamera.com/), the current segment is the one that grows, the rest maintain their size and shift their offset based on the next segment. We could be clever and deduce that the last item in our <code>progressLayers</code> array is the current segment, but it's often nicer to be explicit. 

{% highlight objc %}
    self.currentProgressLayer = progressLayer;
{% endhighlight %}

So at the end of all that, we might end up with a method that looks a bit like this

{% highlight objc %}
	- (void)addNewLayer
	{
	    CAShapeLayer *progressLayer = [CAShapeLayer layer];
	    progressLayer.path = self.circlePath.CGPath;
	    progressLayer.strokeColor = [[self randomColor] CGColor];
	    progressLayer.fillColor = [[UIColor clearColor] CGColor];
	    progressLayer.lineWidth = self.strokeWidth;
	    progressLayer.strokeEnd = 0.f;
	    
	    [self.layer addSublayer:progressLayer];
	    [self.progressLayers addObject:progressLayer];
	    
	    self.currentProgressLayer = progressLayer;
	}
{% endhighlight %}

Now that we have a reference to our current progress segment, as well as all preceding segments, we need a way to animate the progress of the current segment and the position of the existing segments. We also need a way to pause the animation when we receive <code>touchesEnded:withEvent:</code>. 

This raises a few interesting challenges. 

The first challenge is how we're going to adjust the position of the existing segments. We could apply a rotation transform to the layer, but we could also take advantage of another <code>CAShapeLayer</code> property, <code>strokeStart</code>, an animatable property which when combined with <code>strokeEnd</code> can define the region of the path to stroke. 

The second challenge is how we might pause the animation. Usually when interacting with an animation, it's very much a set and forget scenario; we tell the animation what to do and how long to take and it will diligently go off and perform it. To effectively 'pause' the animation, we're going to have to do some trickery involving taking a snapshot of the current state of the layer and remove the animation altogether. 

To achieve this, we'll be using <code>CABasicAnimation</code> and the <code>presentationLayer</code> property on <code>CALayer</code>. 

## Animating

Let's take a look at the entirety of our animation method and step through the bits of interest.

{% highlight objc %}
	- (void)updateAnimations
	{    
	    CGFloat duration = self.duration * (1.f - [[self.progressLayers firstObject] strokeEnd]);
	    CGFloat strokeEndFinal = 1.f;
	    
	    for (CAShapeLayer *progressLayer in self.progressLayers)
	    {
	        CABasicAnimation *strokeEndAnimation = nil;
	        strokeEndAnimation = [CABasicAnimation animationWithKeyPath:@"strokeEnd"];
	        strokeEndAnimation.duration = duration;
	        strokeEndAnimation.fromValue = @(progressLayer.strokeEnd);
	        strokeEndAnimation.toValue = @(strokeEndFinal);
	        strokeEndAnimation.autoreverses = NO;
	        strokeEndAnimation.repeatCount = 0.f;
	        strokeEndAnimation.fillMode = kCAFillModeForwards;
	        strokeEndAnimation.removedOnCompletion = NO;
	        strokeEndAnimation.delegate = self;
	        [progressLayer addAnimation:strokeEndAnimation forKey:@"strokeEndAnimation"];
	        
	        strokeEndFinal -= (progressLayer.strokeEnd - progressLayer.strokeStart);
	        
	        if (progressLayer != self.currentProgressLayer)
	        {
	            CABasicAnimation *strokeStartAnimation = nil;
	            strokeStartAnimation = [CABasicAnimation animationWithKeyPath:@"strokeStart"];
	            strokeStartAnimation.duration = duration;
	            strokeStartAnimation.fromValue = @(progressLayer.strokeStart);
	            strokeStartAnimation.toValue = @(strokeEndFinal);
	            strokeStartAnimation.autoreverses = NO;
	            strokeStartAnimation.repeatCount = 0.f;
	            strokeStartAnimation.fillMode = kCAFillModeForwards;
	            strokeStartAnimation.removedOnCompletion = NO;
	            [progressLayer addAnimation:strokeStartAnimation forKey:@"strokeStartAnimation"];
	        }
	    }
	}
{% endhighlight %}

We can see we're going to be looping over our collection of progress segment layers and adding a <code>CABasicAnimation</code> to animate the <code>strokeEnd</code> of each. We'll also be adding a <code>CABasicAnimation</code> to animate the <code>strokeStart</code> of all layers that aren't the current layer. This translates into the current segment appearing to grow while maintaining it's starting position, while each of the previous segments will appear to maintain their size but move along the "track".

But what's with those <code>duration</code> and <code>strokeEndFinal</code> variables? 

Let's start with <code>duration</code>. If we imagine that we want the entire animation to take 45 seconds we can pass that in, but what about when we pause and start the animation again? We don't want that to take another 45 seconds, we want it to take however long the previous animation had left before we paused it. To maintain the a persistent duration across all animations, we need a way of keeping track of how far along we've progressed. We know that <code>strokeEnd</code> is a value between 0 and 1 so we can easily use the <code>strokeEnd</code> of the first segment that was added to determine an overall percentage of how far along we are. For example, if the <code>strokeEnd</code> of the first segment was 0.5, we'd know we were half way through. 

Now what about this <code>strokeEndFinal</code>. If we imagine we have multiple progress segments, then we wouldn't want them to all animate to the end, instead we would want them to take into account the region of the full circle that has already elapsed. To do this we initialize <code>strokeEndFinal</code> with 1.0 and deduct from it the percentage of the full circle that each preceding segment occupies. In other words if we have multiple segments, the first should finish at the end of the circle, the second should finish at the start of the first and so on.. 

## Pausing

Now that we have our animation logic in place, we need to figure out how we're going to pause/stop the animation when we receive <code>touchesEnded:withEvent:</code>

You may have noticed in the <code>updateAnimations</code> method we set both the <code>stokeEnd</code> and <code>strokeStart</code> animations to not be removed on completion. Since we're going to be doing some fancy footwork by copying the state of the layer when we stop the animations, we will remove the animations ourselves.

Removing the animations while maintaining the visual state of the layers really boils down to two steps. 

1. For each <code>CAShapeLayer</code> we have representing our progress segments, set the <code>strokeStart</code> and <code>strokeEnd</code> values to the values held by the layers <code>presentationLayer</code>. 

2. Remove all animations from the <code>CAShapeLayer</code>.

The <code>presentationLayer</code> of a <code>CALayer</code> represents the current visual state of the layer and, to quote the [iOS 7 docs](https://developer.apple.com/library/mac/documentation/GraphicsImaging/Reference/CALayer_class/Introduction/Introduction.html#//apple_ref/occ/instm/CALayer/presentationLayer): 

> While an animation is in progress, you can retrieve this object and use it to get the current values for those animations.

Sounds perfect!

If we put these two together, we might end up with something like this

{% highlight objc %}
	- (void)removeAnimations
	{
	    for (CAShapeLayer *progressLayer in self.progressLayers)
	    {
	        progressLayer.strokeStart = [progressLayer.presentationLayer strokeStart];
	        progressLayer.strokeEnd = [progressLayer.presentationLayer strokeEnd];
	        [progressLayer removeAllAnimations];
	    }
	}
{% endhighlight %}

## Drawing within the lines

So we can now start and stop our animations, awesome! But there's one more piece of the puzzle. We need to make sure once we have completed our animation we don't keep adding layers and updating animations when we receive <code>touchesBegan:withEvent:</code>.

To do this, we'll set our <code>RecordingCircleOverlayView</code> instance to become the delegate of all the <code>strokeEnd</code> animations we create, implement the delegate callback <code>animationDidStop:finished:</code> and check for any animations that have finished. If any animation has finished, we can assume that all have finished and the circle is complete! Then we store the completion state in a flag (<code>finishedAnimating</code>) and check it before we start or stop any further animations. 

{% highlight objc %}
	- (void)animationDidStop:(CAAnimation *)anim finished:(BOOL)flag
	{
	    if (self.hasFinishedAnimating == NO && flag)
	    {
	        [self removeAnimations];
	        self.finishedAnimating = flag;
	    }
	}
{% endhighlight %}

We'll also call our <code>removeAnimations</code> method to ensure that when the animations are finished, we maintain the final state of our layers.

## Wrapping up

<video src="/media/2014-02-02-spark-camera/video/spark-finished.m4v" autoplay="true" loop="true"></video>

As you can see there's not a lot of code behind a control like this, there are a few edge cases here and there but the real work is coming up with such an ingeniously simple and elegant way to solve a problem like this on a mobile device.

You can [checkout this project on Github](https://github.com/subjc/SparkRecordingCircle).

[^1]: Full disclosure: I work for Itty Bitty Apps, developers of Reveal. 
[^2]: Ole Begemann did a [great writeup](http://oleb.net/blog/2010/12/animating-drawing-of-cgpath-with-cashapelayer/) (3+ years ago!) on this technique 

