---
layout: post
title:  "Castro's playback scrubber"
date:   2014-02-19 17:00:00
description: "Deconstructing a modern take on a classic control."
image: "/media/2014-02-19-castro-playback-scrubber/images/castro-title.jpg"
video: true
video_mp4: "/media/2014-02-19-castro-playback-scrubber/video/castro-title.m4v"
author:
  name: Same Page
  url: https://twitter.com/sampage
---

## Background

The wealth and calibre of podcast apps on the App Store is impressive (read: intimidating). Offerings like [Pocket Casts](http://www.shiftyjelly.com/pocketcasts) by [Shifty Jelly](http://www.shiftyjelly.com/) and [Downcast](http://www.downcastapp.com/) by Jamawkinaw Enterprises have done an amazing job at raising the bar, and with Apple as a [fellow player](https://itunes.apple.com/au/app/podcasts/id525463029?mt=8) in the category, standing out is no mean feat.

Enter [Castro](http://castro.fm/), a relatively recent addition to the App Store by [Supertop](http://supertop.co/), a duo consisting of [Padraig Kennedy](http://twitter.com/Padraig) and [Oisin Prendiville](http://twitter.com/prendio2).

As well as developing [Castro](http://castro.fm/), [Supertop](http://supertop.co/) are behind [Tokens](http://usetokens.com/), an OS X app for developers to manage App Store promo codes.

[Castro](http://castro.fm/) is an exercise in bold restraint. It eschews buttons wherever possible in favour of gestural interactions allowing the screen to defer to the content, rather than the controls.

While Apple provided the interactive pop gesture in iOS 7, it has largely been used as an auxiliary way of popping view controllers off the navigation stack. [Castro](http://castro.fm/) takes opposite tact and does away with the back button altogether, incorporating gestures to naturally switch between contexts.

When listening to a podcast, the playback controls follow you around the app, morphing between states to give you only what you need. When you need to do more than just pause and play, the playback controls give you a fluid way to scrub through your podcast.

That's what we'll be looking at today.

## Analysis

[Castro](http://castro.fm/)'s playback controls are built atop the traditional model of device playback controls: play/pause, fast forward and rewind. Scrubbing through content is where these controls really shine.

The standard pattern in iOS (and many platforms) is to utilize a playhead to convey elapsed time, as well as be the control with which to scrub through the playback. [Castro](http://castro.fm/) takes a modern, touch optimized approach where the entire playback bar becomes one big slider to move the playhead throughout the content. This creates a responsive and natural way to adjust the playback position.

Having a control that manipulates playback reside in the primary touch area of the device is fraught with potential issues. Having a control that can skip through content with a swipe increases these issues exponentially. How often have you accidentally swiped the screen of a device when you're holding it in one hand?

[Castro](http://castro.fm/) overcomes these issues by not committing the change to the playback until the control has finished animating to its editing state, a change that takes ~0.2 seconds. This provides just enough time to cancel any touches before you accidentally skip halfway through that hour long podcast.

The attention to detail in this control is phenomenal. Just when you think you've got it figured out there's another little subtlety ready to impress.

## Reasoning

If you'll indulge me for a moment, it may be interesting to dig a bit deeper into why we might need such fine control over the playback of podcast content.

It's reasonable to say that controls for audio playback on mobile devices are generally geared toward music playback. While podcasts are delivered in the same format, consumption typically differs from music.

Podcasts are primarily delivering content through verbal language. When the meaning of the content is reliant on the preceding context, the ability to quickly move backward and forward becomes more important (this is where we might go off on a tangent about music and its construction also being reliant on the preceding context, but that may be out of the scope of this post).

At the time of writing, the top 10 songs on iTunes had an average length of 3:34, whereas the top 10 podcasts had an average length of 43:56. If we couple the nature of the content with the increased average duration, it becomes apparent that the resolution of the controls to manipulate playback should be adjusted accordingly.

## Deconstructing

At first glance, it might seem like our friend <code>UIScrollView</code> would be a good candidate to provide the foundation of the scrubbing mechanism, but if we look closely, there's more than immediately meets the eye.

<video src="/media/2014-02-19-castro-playback-scrubber/video/castro-scrubber-bounce.m4v" autoplay="true" loop="true"></video>

Notice that slight bounce when the controls bar returns to its original position after the dragging interaction? That's not something we normally see with <code>UIScrollView</code>, at least not out of the box. Normally we'd expect to see <code>UIScrollView</code> have an ease out curve when returning to its original position. This control overshoots its mark and snaps back.

If we do some sleuthing, iTunes tells us that the minimum iOS version for [Castro](http://castro.fm/) is iOS 7. To draw a long bow, we could interpret this evidence of the use of UIKit Dynamics.

To add a bit more weight to this hypothesis, we need only look to the hint we get when we tap on the edges of the control.

<video src="/media/2014-02-19-castro-playback-scrubber/video/castro-tap-bounce.m4v" autoplay="true" loop="true"></video>

Notice that the view again exceeds its original position slightly before coming to a rest?

This is a subtle but significant detail. It's part of what makes the control really great. All animations share the same physical properties.

Looks like a good reason to get our feet wet with UIKit Dynamics.

## Movement

Before we get started with UIKit Dynamics (AKA the fun stuff), we're going to need a way of moving our controls horizontally to initiate the scrubbing.

To do this, we'll add a <code>UIPanGestureRecognizer</code> to our control's view to track the location of our finger on the screen and translate that to the horizontal center of our control's view.

{% highlight objc %}
- (void)panGestureRecognized:(UIPanGestureRecognizer *)panGesture
{
    CGPoint translationInView = [panGesture translationInView:self.view];

    if (panGesture.state == UIGestureRecognizerStateBegan)
    {
        self.touchesBeganPoint = translationInView;
    }
    else if (panGesture.state == UIGestureRecognizerStateChanged)
    {
        CGFloat translatedCenterX = self.view.center.x +
                                    (translationInView.x - self.touchesBeganPoint.x);
        self.controlsView.center = CGPointMake(translatedCenterX, self.controlsView.center.y);
    }
}
{% endhighlight %}

Now that we're able to move our controls around, on to the fun stuff.

## UIKit Dynamics

At a basic level, UIKit Dynamics is a collection of high level APIs that provide developers a way to imbue user interface elements with properties that mimic the physical world. What this means for those using the device is a more cohesive and realistic feel across all apps that incorporate dynamics.

We'll skip over the basics, but for an introduction to UIKit Dynamics, Ash Furrow has a great [writeup](http://www.teehanlax.com/blog/introduction-to-uikit-dynamics/) on [the teehan+lax blog](http://www.teehanlax.com/blog).

To mimic the force applied to the control's view to snap it back to its original position, we'll be using the <code>UISnapBehavior</code> class.

{% highlight objc %}
self.dynamicAnimator = [[UIDynamicAnimator alloc] initWithReferenceView:self.view];

UISnapBehavior *snapBehavior = [[UISnapBehavior alloc] initWithItem:self.controlsView
                                                        snapToPoint:point];
snapBehavior.damping = 0.35f;
{% endhighlight %}

We create a <code>UISnapBehavior</code> instance with the <code>initWithItem:snapToPoint:</code> initializer where <code>self.controlsView</code> is the view that we want to drag and <code>point</code> is the original center point that we want to snap back to.

We also provide a <code>damping</code> value which will control how much bounce/oscillation we see once the view returns to its origin.

This will ensure that if our view moves (for instance by a user gesture), UIKit Dynamics will kick in and snap it back to its original position with a slight bounce at the end, however we're not going to add our <code>UISnapBehavior</code> to our <code>UIDynamicAnimator</code> just yet.

Instead we're going to use our <code>panGestureRecognized:</code> method we defined earlier and the state of our <code>UIPanGestureRecognizer</code> to determine when we add/remove our <code>UISnapBehavior</code>.

{% highlight objc %}
- (void)panGestureRecognized:(UIPanGestureRecognizer *)panGesture
{
    CGPoint translationInView = [panGesture translationInView:self.view];

    if (panGesture.state == UIGestureRecognizerStateBegan)
    {
        [self.dynamicAnimator removeBehavior:self.snapBehavior];
        self.touchesBeganPoint = translationInView;
    }
    else if (panGesture.state == UIGestureRecognizerStateChanged)
    {
        CGFloat translatedCenterX = self.view.center.x +
                                    (translationInView.x - self.touchesBeganPoint.x);
        self.controlsView.center = CGPointMake(translatedCenterX, self.controlsView.center.y);
    }
    else if (panGesture.state == UIGestureRecognizerStateEnded)
    {
        [self.dynamicAnimator addBehavior:self.snapBehavior];
    }
}
{% endhighlight %}

By removing our <code>UISnapBehavior</code> when we detect the start of a pan gesture and only re-adding it once the gesture has ended, we avoid having UIKit Dynamics interfere with our control's view while we're tracking touches.

But there's a problem that reveals itself once we build and run. We haven't told UIKit Dynamics anything about how we want our view to react to the force of the <code>UISnapBehavior</code>.

By default, a dynamic item (<code>UIView</code> is one these) will allow rotation, so when the force of our <code>UISnapBehavior</code> is applied, it will rotate based on the force.

To disable rotation, we need to create an instance of <code>UIDynamicItemBehavior</code>, associate it with our control's view and use it to override the default rotation.

{% highlight objc %}
UIDynamicItemBehavior *dynamicItemBehavior = [[UIDynamicItemBehavior alloc]
                                             initWithItems:@[self.controlsView]];
dynamicItemBehavior.allowsRotation = NO;

[self.dynamicAnimator addBehavior:dynamicItemBehavior];
{% endhighlight %}

Now that we have multiple behaviors added to our <code>UIDynamicAnimator</code>, it's a good time to talk about composite behaviors.

## Composite Behaviors

UIKit Dynamics ships with a few concrete subclasses of <code>UIDynamicBehavior</code> that we can use out of the box: <code>UIAttachmentBehavior</code>, <code>UICollisionBehavior</code>, <code>UIDynamicItemBehavior</code>, <code>UIGravityBehavior</code>, <code>UIPushBehavior</code> and <code>UISnapBehavior</code>. But we can also subclass <code>UIDynamicBehavior</code> and add child behaviors to create our own composite behaviors.

We can turn the two behaviors we use to recreate the scrubbing interaction into a composite behavior which will clean up our code, as well as make it easy for us to add/remove the behaviors with one call.

{% highlight objc %}
@implementation SCScrubbingBehavior

- (id)initWithItem:(id<UIDynamicItem>)item snapToPoint:(CGPoint)point;
{
    if (self = [super init])
    {
        UIDynamicItemBehavior *dynamicItemBehavior = [[UIDynamicItemBehavior alloc]
                                                     initWithItems:@[item]];
        dynamicItemBehavior.allowsRotation = NO;
        [self addChildBehavior:dynamicItemBehavior];

        UISnapBehavior *snapBehavior = [[UISnapBehavior alloc] initWithItem:item
                                                                snapToPoint:point];
        snapBehavior.damping = 0.35f;
        [self addChildBehavior:snapBehavior];
    }
    return self;
}

@end
{% endhighlight %}

## Scrubbing

Now that we've got our control tracking our touches and returning to its original position on release, we need a view that can represent the elapsed time relative to the total duration.

In [previous](http://subjc.com/spark-camera) [articles](http://subjc.com/facebook-paper-photo-panner), we've handled displaying progress by using <code>CAShapeLayer</code>. For our case, we don't need the low level flexibility of <code>CAShapeLayer</code> so we'll be implementing our progress view with old reliable <code>UIView</code>.

We'll start by making a subclass of <code>UIView</code> which will be responsible for displaying our progress as both the white timeline and the text label showing the elapsed time.

{% highlight objc %}
@implementation SCTimelineView

- (id)initWithFrame:(CGRect)frame
{
    if (self = [super initWithFrame:frame])
    {
        self.clipsToBounds = YES;
        self.backgroundColor = [[UIColor blackColor] colorWithAlphaComponent:0.5f];

        self.progressView = [[UIView alloc] initWithFrame:CGRectZero];
        self.progressView.backgroundColor = [UIColor whiteColor];
        [self addSubview:self.progressView];

        self.elapsedTimeLabel = [[UILabel alloc] initWithFrame:CGRectZero];
        self.elapsedTimeLabel.textAlignment = NSTextAlignmentCenter;
        self.elapsedTimeLabel.font = [UIFont boldSystemFontOfSize:15.f];
        [self addSubview:self.elapsedTimeLabel];
    }
    return self;
}
{% endhighlight %}

Now we'll need a way to update the views based on what time has elapsed and how that relates to the overall duration of the podcast.

This is the point where we'll need an object that models these two properties. To do this, we'll create an <code>NSObject</code> subclass with an <code>elapsedTime</code> and <code>totalTime</code> property represented as <code>NSTimeIntervals</code>.

{% highlight objc %}
@interface SCPlaybackItem : NSObject

@property (nonatomic, assign) NSTimeInterval elapsedTime;
@property (nonatomic, assign) NSTimeInterval totalTime;

@end
{% endhighlight %}

All the pieces are in place, so now we need a way of notifying our timeline view that it needs to update its subviews based on the state of our <code>SCPlaybackItem</code> object. For this, we'll provide a public interface with which we can pass in our <code>SCPlaybackItem</code> object to our <code>SCTimelineView</code> and allow it to update its subviews accordingly.

{% highlight objc %}
#pragma mark - Public

- (void)updateForPlaybackItem:(SCPlaybackItem *)playbackItem
{
    [self updateProgressViewForPlaybackItem:playbackItem];
    [self updateElapsedTimeLabelForPlaybackItem:playbackItem];
}
{% endhighlight %}

Updating our timeline progress is as simple as modifying the width of the <code>progressView</code> based on the elapsed time vs the overall duration.

{% highlight objc %}
- (void)updateProgressViewForPlaybackItem:(SCPlaybackItem *)playbackItem
{
    if (playbackItem.totalTime > 0.f)
    {
        CGFloat progress = playbackItem.elapsedTime / playbackItem.totalTime;
        self.progressView.frame = CGRectMake(0.f, 0.f, CGRectGetWidth(self.bounds) * progress,
                                             CGRectGetHeight(self.bounds));
    }
}
{% endhighlight %}

Updating our elapsed time label has a little bit more to it.

If we look at [Castro](http://castro.fm/)'s timeline view, there's a lovely little touch where the elapsed time text switches from the right to the left of the playhead based on whether there is enough space to do so. This allows the eye to follow the playhead and see both the relative elapsed time as well as the actual elapsed time in hours/minutes/seconds.

When we update our elapsed time label, we'll first size our label to the appropriate width, then position it depending on whether it is narrow enough to fit in the width of the <code>progressView</code>.

{% highlight objc %}
- (void)updateElapsedTimeLabelForPlaybackItem:(SCPlaybackItem *)playbackItem
{
    self.elapsedTimeLabel.text = [playbackItem stringForElapsedTime];

    CGSize labelSize = [self.elapsedTimeLabel.text sizeWithAttributes:
                        @{NSFontAttributeName:self.elapsedTimeLabel.font}];

    self.elapsedTimeLabel.frame = CGRectMake(CGRectGetMinX(self.elapsedTimeLabel.frame),
                                             CGRectGetMinY(self.elapsedTimeLabel.frame),
                                             labelSize.width + kLabelWidthPadding,
                                             CGRectGetHeight(self.bounds));

    if (CGRectGetWidth(self.elapsedTimeLabel.bounds) > CGRectGetWidth(self.progressView.bounds))
    {
        [self configureElapsedLabelOriginForPendingSegment];
    }
    else
    {
        [self configureElapsedLabelOriginForElapsedSegment];
    }
}
{% endhighlight %}

## Transforming

[Castro](http://castro.fm/)'s timeline view follows the same restrained ethos found throughout the app. When you're scrubbing through the content it expands vertically, borrowing the elapsed time label from the controls to give you just the information you need at that time. When you've finished scrubbing, it collapses down into a slim ~2 point view.

<video src="/media/2014-02-19-castro-playback-scrubber/video/castro-transform.m4v" autoplay="true" loop="true"></video>

When we slow down this process, we can see that the elapsed time label appears to grow as the view expands.

To mimic this, we'll use a <code>CGAffineTransform</code> on our <code>SCTimelineView</code> to adjust both the scale and positioning of the view.

While it might seem initially counter intuitive, we'll have our default (<code>CGAffineTransformIdentity</code>) be the expanded state and our collapsed state be the transformed state. This will allow us to effectively restore our elapsed time label to its default transform when we expand out and "shrink" it down when we collapse our timeline view.

{% highlight objc %}
    CGAffineTransform timelineViewScaleTransform = CGAffineTransformMakeScale(1.f,
                                                   kTimelineCollapsedHeight /
                                                   kTimelineExpandedHeight);
    CGAffineTransform timelineViewTranslationTransform = CGAffineTransformMakeTranslation(0.f,
                                                   kTimelineExpandedHeight /
                                                   kTimelineCollapsedHeight);

    self.timelineView.transform = CGAffineTransformConcat(timelineViewScaleTransform,
                                                          timelineViewTranslationTransform);
{% endhighlight %}

In addition to the scale transformation, we're also transforming the position of the timeline view. This gives the appearance of the timeline view collapsing to become part of the controls view and expanding out to become its own, distinct control.

Expanding the view is as simple as setting our <code>SCTimelineView</code>'s transform back to <code>CGAffineTransformIdentity</code>.

{% highlight objc %}
self.timelineView.transform = CGAffineTransformIdentity;
{% endhighlight %}

## Committing Changes

We spoke briefly about the potential pitfalls of using gestures to control playback of long content, namely accidental swiping, and how [Castro](http://castro.fm/) gets around this is by not committing playback scrubbing until the timeline has completed its expansion animation.

The great thing about this approach is it keeps the control feeling responsive while still providing a sense of security that you won't accidentally skip through the podcast.

To implement this behavior, we'll need to keep a reference to the elapsed time before it has been modified, set a flag (<code>commitTimelineScrubbing</code>) in our expansion animation completion block to indicate that we assume the scrubbing was intentional, and restore the elapsed time if the flag is set to <code>NO</code> when we collapse our timeline view.

In the callback from our <code>UIPanGestureRecognizer</code>, we'll store our reference to the elapsed time when the <code>UIPanGestureRecognizer</code> is in the <code>UIGestureRecognizerStateBegan</code> state, as well as expanding the timeline view.

{% highlight objc %}
if (panGesture.state == UIGestureRecognizerStateBegan)
{
    [self.dynamicAnimator removeBehavior:self.snapBehavior];

    self.elapsedTimeAtTouchesBegan = self.playbackItem.elapsedTime;
    [self expandTimelineViewAnimated:YES];
}
{% endhighlight %}

When we expand the timeline view, we set the <code>commitTimelineScrubbing</code> flag to <code>YES</code> in our animation completion block.

{% highlight objc %}
void (^completionBlock)(BOOL finished) = ^void(BOOL finished) {
    self.commitTimelineScrubbing = YES;
};
{% endhighlight %}

Finally, when we collapse our timeline view, we rollback the changes if the <code>commitTimelineScrubbing</code> flag is set to <code>NO</code>.

{% highlight objc %}
if (self.shouldCommitTimelineScrubbing == NO)
{
    self.playbackItem.elapsedTime = self.elapsedTimeAtTouchesBegan;
    [self.timelineView updateForPlaybackItem:self.playbackItem];
    [self.controlsView updateForPlaybackItem:self.playbackItem];
}
{% endhighlight %}

## Behaviors, Behaviors, Behaviors

Another detail mentioned briefly was the hint the control gives about the scrubbing interaction when you tap on an empty part of it, much like that of the iOS 7 lockscreen camera.

An immediate reaction is to implement this using a combination of <code>UIPushBehavior</code> to propel the controls outward, <code>UIGravityBehavior</code> to bring the control back to its original position and <code>UICollisionBehavior</code> to keep the control within the superview bounds.

This is roughly how we're going to implement this behavior, but with a bit of a twist.

If we were to use <code>UICollisionBehavior</code> exclusively to bring the view to a rest, it would collide with the superview bounds before coming to a stop at its original position. This would be in contrast to the physical properties we've already defined in our scrubbing behavior, breaking the illusion that it exists in the physical world.

To maintain this illusion, we're still going to use the combination of <code>UIPushBehavior</code>, <code>UIGravity</code> and <code>UICollisionBehavior</code>, however we'll supplement it with our existing <code>SCScrubbingBehavior</code> composite behavior.

To implement the basis of the tap hint behavior, we'll create another <code>UIDynamicItem</code> subclass to serve as a composite behavior.

{% highlight objc %}
@implementation SCTapHintBehavior

- (id)initWithItem:(NSArray *)items
{
    if (self = [super init])
    {
        self.pushBehavior = [[UIPushBehavior alloc] initWithItems:items
                                                    mode:UIPushBehaviorModeInstantaneous];
        [self addChildBehavior:self.pushBehavior];

        self.gravityBehavior = [[UIGravityBehavior alloc] initWithItems:items];
        [self addChildBehavior:self.gravityBehavior];

        self.collisionBehavior = [[UICollisionBehavior alloc] initWithItems:items];
        [self addChildBehavior:self.collisionBehavior];
    }
    return self;
}

@end
{% endhighlight %}

When we create an instance of our <code>SCTapHintBehavior</code>, we'll become the <code>collisionDelegate</code> of the <code>UICollisionBehavior</code> so that we can respond to collision events, as well as disabling the <code>translatesReferenceBoundsIntoBoundary</code> flag as we'll be managing the boundaries ourselves.

{% highlight objc %}
self.tapHintBehavior = [[SCTapHintBehavior alloc] initWithItem:@[self.controlsView]];
self.tapHintBehavior.collisionBehavior.collisionDelegate = self;
self.tapHintBehavior.collisionBehavior.translatesReferenceBoundsIntoBoundary = NO;
{% endhighlight %}

To respond to touch events, we'll add a <code>UITapGestureRecognizer</code> to our controls view and handle touches in the selector we provide.

Now we're going to be doing some pretty funky looking behavior juggling, but the core logic is fairly straightforward.

When we receive a tap, remove our <code>SCScrubbingBehavior</code> and add our <code>SCTapHintBehavior</code>.

{% highlight objc %}
[self.dynamicAnimator removeBehavior:self.scrubbingBehavior];
[self.dynamicAnimator addBehavior:self.tapHintBehavior];
{% endhighlight %}

Check the location of the tap to decide the direction of the bounce.

{% highlight objc %}
CGPoint locationInView = [tapGestureRecognized locationInView:self.view];
if (locationInView.x < CGRectGetMidX(self.view.bounds))
{
    // Push right
}
else
{
    // Push left
}
{% endhighlight %}

Trigger an instantaneous push on our <code>UIPushBehavior</code> in that direction.

{% highlight objc %}
[self.tapHintBehavior.pushBehavior setAngle:0.f magnitude:kPushMagnitude];
self.tapHintBehavior.pushBehavior.active = YES;
{% endhighlight %}

Adjust the angle of our <code>UIGravityBehavior</code> to bring the view back to its original position.

{% highlight objc %}
[self.tapHintBehavior.gravityBehavior setAngle:M_PI magnitude:kGravityMagnitude];
{% endhighlight %}

Create a boundary for our <code>UICollisionBehavior</code> that is slightly outside its origin.

{% highlight objc %}
CGPoint leftCollisionPointTop = CGPointMake(kCollisionBoundaryOffset * -1, 0.f);
CGPoint leftCollisionPointBottom = CGPointMake(kCollisionBoundaryOffset * -1,
                                               CGRectGetHeight(self.view.bounds));

[self.tapHintBehavior.collisionBehavior addBoundaryWithIdentifier:@"leftCollisionPoint" fromPoint:leftCollisionPointTop toPoint:leftCollisionPointBottom];
{% endhighlight %}

Then when we receive our delegate callback from the <code>UICollisionBehavior</code> that it has collided with its boundary, we remove our <code>SCTapHintBehavior</code> and add our <code>SCScrubbingBehavior</code>, triggering the <code>UISnapBehavior</code> to snap the controls back to their original position.

{% highlight objc %}
- (void)collisionBehavior:(UICollisionBehavior *)behavior beganContactForItem:(id<UIDynamicItem>)item withBoundaryIdentifier:(id<NSCopying>)identifier atPoint:(CGPoint)p
{
    [self.dynamicAnimator addBehavior:self.scrubbingBehavior];
    [self.dynamicAnimator removeBehavior:self.tapHintBehavior];
}
{% endhighlight %}

## Closing

<video src="/media/2014-02-19-castro-playback-scrubber/video/castro-finished.m4v" autoplay="true" loop="true"></video>

Admittedly this turned into a much longer article than was initially planned, thanks for sticking with it!

[Castro](http://castro.fm/)'s playback controls are built in such a way that explaining nuances piecemeal would do it a disservice. To quote one of [Dieter Rams](http://en.wikipedia.org/wiki/Dieter_Rams)' [10 principals of good design](http://en.wikipedia.org/wiki/Dieter_Rams#Dieter_Rams:_ten_principles_for_good_design):

>> Good design is thorough down to the last detail.

You can [checkout this project on Github](https://github.com/subjc/SubjectiveCCastroControls).
