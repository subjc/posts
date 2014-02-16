---
layout: post
title:  "Facebook Paper's tilting panner"
date:   2014-02-12 18:20:00
description: "Deconstructing an expressive and immersive way to explore photos."
image: "/media/2014-02-12-facebook-paper-photo-panner/images/title-image.jpg"
video: true
video_mp4: "/media/2014-02-12-facebook-paper-photo-panner/video/title-video.m4v"
---

## Background

In 2011 [Facebook purchased Push Pop Press](http://pushpoppress.com/about/), a company founded by former Apple employees [Kimon Tsinteris](http://kimtsi.com) and [Mike Matas](http://mikematas.com) aimed at creating a platform on which engaging digital books and publications for iOS could be built. [Push Pop Press](http://pushpoppress.com/)' technology was initially used to create Al Gore's book [Our Choice](https://itunes.apple.com/au/app/al-gore-our-choice-plan-to/id432753658?mt=8), which would become the flagship example of the platform. At the time, it was unknown as to whether Facebook had plans for the platform [Push Pop Press](http://pushpoppress.com/) had developed or whether this was purely an "[acquihire](http://en.wikipedia.org/wiki/Acqui-hiring‎)". 

When Facebook announced [Paper](https://itunes.apple.com/us/app/paper-stories-from-facebook/id794163692?mt=8) alongside [Facebook Creative Labs](https://www.facebook.com/labs), a previously unknown skunkworks within Facebook, it now seems apparent that at the very least, the ethos behind [Push Pop Press](http://pushpoppress.com/)' innovative digital creation tool had not fallen by the wayside.

## Analysis

While [Paper](https://itunes.apple.com/us/app/paper-stories-from-facebook/id794163692?mt=8) is laden with noteworthy interactions, we're going to be looking at the panoramic photo panner. In particular, this control typifies an incredibly articulate use of device motion and touch free interaction. Unless you've used [Paper](https://itunes.apple.com/us/app/paper-stories-from-facebook/id794163692?mt=8) in person, it's hard to convey the level of immersion you get when viewing photos, especially panoramic content. It feels like what Apple were trying to do when they implemented their parallax effect for icons and dialogs in iOS 7. 

There is an inherent risk when including motion based controls in an app, often they serve no real purpose and at worst, they can hinder engagement rather than improving it. The [Facebook Creative Labs](https://www.facebook.com/labs) team have done a superb job at treading these boundaries to create a control that is there when you need it and gets out of your way when you don't. In short, it just "feels right".

## Moving parts

If think of the photo panner without the associated motion control, we can break it down into a view to display the image (<code>UIImageView</code>), a view for the image view to live in which handles the panning (<code>UIScrollView</code>) and a layer to display the scroll bar (<code>CAShapeLayer</code>).

Displaying the image at the full height of our device can be handled by <code>UIScrollView</code>'s zoom functionality, we just need to provide the appropriate zoom scale based on the aspect ratio of the device and image.

{% highlight objc %}
CGFloat zoomScale = [self maximumZoomScaleForImage:image];

self.panningScrollView.maximumZoomScale = (CGRectGetHeight(self.panningScrollView.bounds) / CGRectGetWidth(self.panningScrollView.bounds)) * (image.size.width / image.size.height);
self.panningScrollView.zoomScale = zoomScale;
{% endhighlight %}

From these components, we can build much of the photo panner functionality. To react to device motion, we'll need to make use of the many device sensors that we have at our disposal.

## Device Motion

The last few generations of iOS devices have shipped with a bunch of different sensors for measuring device orientation and acceleration, the one in particular that we'll be looking at is the gyroscope. To quote the [tome of all human knowledge](http://en.wikipedia.org/wiki/Gyroscope):

> A gyroscope is a device for measuring or maintaining orientation, based on the principles of angular momentum.

For us, this means we can accurately predict, based on a reference point, what direction our device is facing.

To be notified when the gyroscope orientation has changed, we'll be using an instance of <code>CMMotionManager</code>, a class that encapsulates interaction with the device motion sensors. <code>CMMotionManager</code> allows you effectively subscribe (using a block) to changes in accelerometer, gyroscope or magnetometer data. It also provides something called device motion updates which encompass the attitude, rotation rate and acceleration of a device. We'll be using this as it provides the same data as the raw gyroscope callbacks, but whose bias has been removed by Core Motion algorithms.

To start receiving callbacks for device motion, we call <code>startDeviceMotionUpdatesToQueue:withHandler:</code> and pass in an <code>NSOperationQueue</code> and a block to perform on the change.

{% highlight objc %}
[self.motionManager startDeviceMotionUpdatesToQueue:[NSOperationQueue mainQueue] 
withHandler:^(CMDeviceMotion *motion, NSError *error) {
    // Do our processing
}];
{% endhighlight %}

Now that we're receiving callbacks from our <code>CMMotionManager</code> when the device orientation changes, we need to translate the gyroscope data into something that can adjust our <code>UIScrollView</code>'s <code>contentOffset</code>.

The data we care about the most is the <code>rotationRate</code> which will give us the <code>x</code>, <code>y</code> and <code>z</code> rotation of our device. 

{% highlight objc %}
CGFloat xRotationRate = motion.rotationRate.x;
CGFloat yRotationRate = motion.rotationRate.y;
CGFloat zRotationRate = motion.rotationRate.z;
{% endhighlight %}

In particular, we'll be using the <code>yRotationRate</code> to determine our device tilt, however we'll also be using the <code>xRotationRate</code> and <code>zRotationRate</code>.

If we take a look at [Paper](https://itunes.apple.com/us/app/paper-stories-from-facebook/id794163692?mt=8), much of what makes it "feel right" is that there's little to no accidental movement triggered when you rotate the device along an axis other than the <code>y</code>. 

To accomplish (or at the very least approximate) this, we'll use the <code>xRotationRate</code> and <code>zRotationRate</code> as a threshold for responding to our <code>yRotationRate</code>. 

{% highlight objc %}
if (fabs(yRotationRate) > (fabs(xRotationRate) + fabs(zRotationRate)))
{
  // Do our movement
}
{% endhighlight %}

If we have a <code>yRotationRate</code> that is greater than the sum of the <code>xRotationRate</code> and <code>zRotationRate</code>, we'll assume that the movement was intentional and adjust our <code>UIScrollView</code> accordingly. 

Translating the device movement into scroll position is an instance where the dreaded magic number becomes a necessity. There's no direct analog we can use to translate between device motion and scroll position, it's a matter of playing with a multiplier until you find something that, again, "feels right". 

We also have to factor in the zoom scale of the image we're displaying so regardless of image dimensions, device motions translates into the same relative change in scroll position. 

{% highlight objc %}
CGFloat invertedYRotationRate = yRotationRate * -1;

CGFloat zoomScale = (CGRectGetHeight(self.panningScrollView.bounds) / CGRectGetWidth(self.panningScrollView.bounds)) * (image.size.width / image.size.height);
CGFloat interpretedXOffset = self.panningScrollView.contentOffset.x + (invertedYRotationRate * zoomScale * kRotationMultiplier);

CGPoint contentOffset = [self clampedContentOffsetForHorizontalOffset:interpretedXOffset];
{% endhighlight %}

The <code>clampedContentOffsetForHorizontalOffset</code> is a simple method takes a horizontal offset and returns a <code>CGPoint</code> representing an offset for the <code>UIScrollView</code> that centers the content vertically and restricts it from exceeding the horizontal bounds.

{% highlight objc %}
- (CGPoint)clampedContentOffsetForHorizontalOffset:(CGFloat)horizontalOffset;
{
  CGFloat maximumXOffset = self.panningScrollView.contentSize.width - CGRectGetWidth(self.panningScrollView.bounds);
  CGFloat minimumXOffset = 0.f;
  
  CGFloat clampedXOffset = fmaxf(minimumXOffset, fmin(horizontalOffset, maximumXOffset));
  CGFloat centeredY = (self.panningScrollView.contentSize.height / 2.f) - (CGRectGetHeight(self.panningScrollView.bounds)) / 2.f;
  
  return CGPointMake(clampedXOffset, centeredY);
}
{% endhighlight %}

We've also inverted the rotation rate to mimic [Paper](https://itunes.apple.com/us/app/paper-stories-from-facebook/id794163692?mt=8)'s scroll direction when rotating the device (a detail I overlooked for far too long). 

At this stage, we'd have something that on [Paper](https://itunes.apple.com/us/app/paper-stories-from-facebook/id794163692?mt=8) does the job, but if we build and ran our code now, we'd quickly notice a disconcerting jitter on movement. This is because we've done nothing to smooth or filter the changes. 

It's a bit _too_ accurate. 

## Smoothing

If we take another look at [Paper](https://itunes.apple.com/us/app/paper-stories-from-facebook/id794163692?mt=8), rotating the device causes the image to glide across the screen, coming to a slow rest when it reaches it's apex. In our current state, we glide with the finesse of a giraffe on ice. We not only need to smooth out the general movement but also incorporate an ease out function so we don't arrive at a complete stop. 

Choosing the highest level of abstraction and only dropping down a level when the original doesn't meet the requirements or is not performant is an mindset that's prevalant throughout Cocoa and Core Foundation. We're provided with a wealth of options for common problems. Grand Central Dispatch and <code>NSOperationQueue</code>'s for asynchronous tasks, <code>UIView</code> and <code>CALayer</code> for displaying content on the screen and UIView block based animation and <code>CAAnimation</code> for animating that content. 

Each of these overlap in functionality to a large extent, but exist to tackle problems in a different way, depending on the requirement. 

By using UIView block based animation, we can tap into the power of Core Animation, provide an ease out function, not block user interaction and have our changes be relative to our current state, all with one call. 

{% highlight objc %}
CGFloat kRotationMultiplier = 5.f;
[UIView animateWithDuration:kMovementSmoothing
                      delay:0.0f
                    options:UIViewAnimationOptionBeginFromCurrentState|
                            UIViewAnimationOptionAllowUserInteraction|
                            UIViewAnimationOptionCurveEaseOut
                 animations:^{
                     [self.panningScrollView setContentOffset:contentOffset animated:NO];
                 } completion:NULL];
{% endhighlight %}

Now if we build and run our code, we'd notice the jitter gone and the whole interacting feeling much more natural.

## Scroll bar

To implement the scroll bar, we're going to be using <code>CAShapeLayer</code> and the <code>strokeStart</code> and <code>strokeEnd</code> properties to adjust the apparent length and location of the scroll bar. We won't go into too much detail on this technique as we've covered it [previously](/spark-camera/), instead we'll be delving into a way to keep it in lock-step with the <code>contentOffset</code> of the <code>UIScrollView</code> using <code>CADisplayLink</code>.

> A <code>CADisplayLink</code> object is a timer object that allows your application to synchronize its drawing to the refresh rate of the display.

We'll be using our <code>CADisplayLink</code> object to provide us with a display synchronized callback in which to poll the current position of our <code>UIScrollView</code> content and adjust our scroll bar accordingly. The benefit of using a CADisplayLink over an NSTimer for this kind of operation is that we can be align our scroll bar changes to the potentially varying frame rate of the display.

Setting up a <code>CADisplayLink</code> object is similar to that of an NSTimer; we create the <code>CADisplayLink</code>, set the target callback to fire at every screen refresh and add it to a run loop. 

{% highlight objc %}
self.displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(displayLinkUpdate:)];
[self.displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
{% endhighlight %}

You may notice we're adding our <code>CADisplayLink</code> object to the run loop for the <code>NSRunLoopCommonModes</code> mode. While it's not covered in this post, we'll also want to support touch tracking which will place our <code>UIScrollView</code> in the <code>UITrackingRunLoopMode</code> tracking mode. If we were to use the <code>NSDefaultRunLoopMode</code> in this instance, we'd lose visual feedback of the scroll bar while tracking touches.

As we're using block based animation to update our <code>UIScrollView</code>, we can't rely on the <code>contentOffset</code> property to be 1:1 with what is being displayed onscreen. Instead we'll be polling the <code>presentationLayer</code> which will provide us with a close approximation of what is being displayed.

Unfortunately the properties we need aren't as easily identified on the <code>presentationLayer</code> as they are on <code>UIScrollView</code>, but it's not difficult to translate what we have to what we understand. 

To retrieve the current <code>contentOffset</code> and <code>contentSize</code>, we'll be using the <code>presentationLayer</code>'s from our <code>UIScrollView</code> and <code>UIImageView</code> respectively.

{% highlight objc %}
CALayer *panningImageViewPresentationLayer = self.panningImageView.layer.presentationLayer;
CALayer *panningScrollViewPresentationLayer = self.panningScrollView.layer.presentationLayer;

CGFloat horizontalContentOffset = CGRectGetMinX(panningScrollViewPresentationLayer.bounds);
CGFloat contentWidth = CGRectGetWidth(panningImageViewPresentationLayer.frame);
{% endhighlight %}

Once we have these, we need to calculate based on our <code>UIScrollView</code> width, what percentage of the content is visible and the position of the content relative to the size of the content.

{% highlight objc %}
CGFloat visibleWidth = CGRectGetWidth(self.panningScrollView.bounds);  
CGFloat clampedXOffsetAsPercentage = fmax(0.f, fmin(1.f, horizontalContentOffset / (contentWidth - visibleWidth)));

CGFloat scrollBarWidthPercentage = visibleWidth / contentWidth;
CGFloat scrollableAreaPercentage = 1.0 - scrollBarWidthPercentage;
{% endhighlight %}

All that is left now is to pass these values along to our <code>CAShapeLayer</code>, which in this case is a <code>sublayer</code> of another view's <code>layer</code>, every time we receive the callback from our <code>CADisplayLink</code> object.

{% highlight objc %}
- (void)updateWithScrollAmount:(CGFloat)scrollAmount forScrollableWidth:(CGFloat)scrollableWidth inScrollableArea:(CGFloat)scrollableArea
{
  self.scrollBarLayer.strokeStart = scrollAmount * scrollableArea;
  self.scrollBarLayer.strokeEnd = (scrollAmount * scrollableArea) + scrollableWidth;
}
{% endhighlight %}

At this point, there's one more thing to take into account. It wasn't mentioned in [previous post](/spark-camera/), but many <code>CALayer</code> properties provide an [implicit animation](https://developer.apple.com/library/ios/documentation/Cocoa/Conceptual/CoreAnimation_guide/AnimatableProperties/AnimatableProperties.html) that we conveniently overrode by providing our own <code>CABasicAnimation</code>'s. 

In our case, this would cause our scroll bar updates to lag behind the actual <code>UIScrollView</code> updates while their animations completed. 

To get around this implicit animation, we need to remove the actions for the <code>strokeStart</code> and <code>strokeEnd</code> from our <code>CAShapeLayer</code>.

{% highlight objc %}
self.scrollBarLayer.actions = @{@"strokeStart": [NSNull null], @"strokeEnd": [NSNull null]};
{% endhighlight %}

## Closing

<video src="/media/2014-02-12-facebook-paper-photo-panner/video/paper-finished.m4v" autoplay="true" loop="true"></video>

Hopefully this post has provided some appreciation for the thought and finesse that have gone into this and other iOS controls, and we've only just scratched the surface. 

You can [checkout this project on Github](https://github.com/subjc/SubjectiveCPhotoPanner).