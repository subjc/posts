---
layout: post
title:  "Unread's pull-for-menu"
date:   2014-02-19 17:00:00
description: "Deconstructing the evolution of revolution."
image: "/media/2014-03-22-unread-overlay-menu/images/title.jpg"
video: true
video_mp4: "/media/2014-03-22-unread-overlay-menu/video/title-video.m4v"
author:
  name: Same Page
  url: https://twitter.com/sampage
---

## Background

In mid-2013 the RSS world was dealt a substantial blow. Google announced that their (*the*) RSS subscription service, [Google Reader](http://www.google.com/reader/about/), was to be shuttered. With it, millions of voices suddenly cried out in terror, and were suddenly silenced.

Declining use was cited as the key reason for the shutdown, though the immense reaction from [Google Reader](http://www.google.com/reader/about/) users indicated that this was a service which still maintained a large audience. The web was abuzz with concern for the future of RSS and the open web in general, though there was also a sense of optimism, a chance for those without the resources of a giant like Google to gain a foothold in the vacuum of what was once a tightly dominated market. To build a worthy replacement for the [Google Reader](http://www.google.com/reader/about/) exiles.

Despite what could’ve been its death knell, RSS is alive and well today with services like [Feedly](http://feedly.com/), [Feedwrangler](https://feedwrangler.net) and [Feedbin](https://feedbin.com/) filling the void left by [Google Reader](http://www.google.com/reader/about/)'s demise. Along with it came a new breed of modern iOS RSS readers. Among them is [Unread](http://jaredsinclair.com/unread/), a delightfully clean and easy to use client for the aforementioned services, built by [Jared Sinclair](http://jaredsinclair.com/). In the short time it has spent on the app store, it has garnered a considerable following, so considerable that there’s a good chance you’re reading these very words from [Unread](http://jaredsinclair.com/unread/).

This article is about [Unread](http://jaredsinclair.com/unread/)'s pull-for-menu interaction, but it is also about history, how far we’ve come and how we got here.

## Landscape

If we were to plot the landscape of news and content aggregation apps on iOS, we might plot apps like [Flipboard](http://flipboard.com/) and [Pulse](https://www.pulse.me/) (now LinkedIn Pulse) at one end of the scale, where the experience drives not only content consumption but content discovery. These are the apps you imagine yourself using when you sit down on a Sunday morning with a coffee (tea for those in the antipodes) and get lost in the magazine experience.

On the opposite end of that scale we have apps like [Reeder](http://reederapp.com/ios/), apps that are about consuming content in the most efficient way, the apps you use to escape the monotony of a daily commute or rid yourself of [FOMO](http://en.wikipedia.org/wiki/Fear_of_missing_out). This is where one might plot [Unread](http://jaredsinclair.com/unread/).

[Unread](http://jaredsinclair.com/unread/) continues a theme of restraint we've discussed [previously](http://subjc.com/castro-playback-scrubber/). The way it bills itself is simple: you sign in to your RSS aggregation account of choice and you read. That's it. Within this spartan ethos [Unread](http://jaredsinclair.com/unread/) provides an experience designed and engineered for single handed use.

To really appreciate where [Unread](http://jaredsinclair.com/unread/)'s menu interaction is coming from, let's get Darwinistic.

## Evolution

If we look back to [Tweetie](http://en.wikipedia.org/wiki/Tweetie), an app regarded as an icon of iOS development, it introduced us to the the now commonplace pull-to-refresh pattern. Pull-to-refresh became so accepted, even expected, it was validated by Apple and adopted as the default mechanism for refreshing your Mail.app inbox.

Then there was the [Facebook iOS app](https://itunes.apple.com/au/app/facebook/id284882215?mt=8‎) which popularized the navigation drawer (aka "God Burger", "Burger Basement" and many other epithets). While they've since removed this for the navigation (it still remains for contacts), the extent at which it spread throughout the iOS design landscape made it a conventional, accepted pattern.

Fast forward to today and we have [Unread](http://jaredsinclair.com/unread/)'s menu, an amalgam of these two accepted, conventional patterns. It's the evolution of two revolutionary interactions which set a precedent for how we interact with our devices.

[Unread](http://jaredsinclair.com/unread/) provides a tutorial on first launch which explains how to present the menu, though one could argue that it isn't needed. It is a product of its lineage and as such, can rely on a certain level of expectation and understanding that that lineage has established.

## Deconstructing

This year's WWDC yielded a bumper crop of *new shiny* for developers to play with: UIKit Dynamics, Text Kit, Sprite Kit and UIViewController transitions to name but a few. We'll be using two of these to recreate [Unread](http://jaredsinclair.com/unread/)'s menu, UIViewController transitions and UIKit Dynamics, although the latter we won’t be dealing with directly.

<video src="/media/2014-03-22-unread-overlay-menu/video/slomo.m4v" autoplay="true" loop="true"></video>

The first thing we notice when we pull the content to expose the menu is the spring in the pull indicator. Contrasted against the focussed, understated reading interface of [Unread](http://jaredsinclair.com/unread/) it's hard to miss. It's reminiscent of the (sadly) short lived iOS 6 pull-to-refresh animation[^1], delightfully playful and descriptive of the interaction progress.

We’ve covered similar dynamic behaviour [in the past](http://subjc.com/castro-playback-scrubber/) and implemented it using UIKit Dynamics, this time around we’ll step up an abstraction layer.

## That 7 parameter method

One of the great features of Objective-C is named parameters. Coupled with the verbosity of the language, it provides a natural way to describe and document the intent of a method, though the length of some methods can scare some new developers. One such method is the newly added UIView block based animation method, <code>animateWithDuration:delay:usingSpringWithDamping:initialSpringVelocity:options:animations:completion:</code> which, while not the longest method in Cocoa Touch, is certainly on the [scoreboard](https://github.com/Quotation/LongestCocoa).

Despite its imposing presence, it's an incredibly simple to use yet powerful method for adding dynamic animated behaviour to your interface without the overhead of pulling in a full UIKit Dynamics stack. It was noted by some [observant](http://iosdevweekly.com/issues/136) [readers](https://twitter.com/followben/status/442224305657483264) that the dynamic behaviour in a [previous post](http://subjc.com/castro-playback-scrubber/) may have been implemented using this method, so it seems like a great opportunity to put it to work on the pull-for-menu spring behaviour.

## Stretttch

If you imagine stretching a out a rubber band, the further it is stretched, the thinner the band will become. This physical behaviour is mirrored in [Unread](http://jaredsinclair.com/unread/)'s pull interaction and while it's a small detail, one that you mightn’t notice unless you were looking for it, it strengthens the perception that when we’re dragging the scroll view beyond its <code>contentSize</code>, we’re met by resistance.

To mimic this behaviour in our implementation, we’ll be providing a view (<code>SCSpringExpandingView</code>) that will animate between two different frames. The view's frame for our collapsed, un-expanded state will take full width of its superview with a height that matches, giving us a small square view.

{% highlight objc %}
- (CGRect)frameForCollapsedState
{
    return CGRectMake(0.f, CGRectGetMidY(self.bounds) - (CGRectGetWidth(self.bounds) / 2.f),
                      CGRectGetWidth(self.bounds), CGRectGetWidth(self.bounds));
}
{% endhighlight %}

When we stretch our view into its expanded state, we’ll use a frame that is the height of the superview but only half the width. We’ll also shift the horizontal origin across so our view stays within the centre of the superview.

{% highlight objc %}
- (CGRect)frameForExpandedState
{
    return CGRectMake(CGRectGetWidth(self.bounds) / 4.f, 0.f,
                      CGRectGetWidth(self.bounds) / 2.f, CGRectGetHeight(self.bounds));
}
{% endhighlight %}

To round the corners of our view, we’ll set the <code>cornerRadius</code> of our stretching view’s layer to be half the width of the view, giving it a circular appearance when collapsed and a rounded edge when expanded. We’ll need to update this value when we transition between our collapsed and expanded states as we modify the width of the frame, otherwise one of the cases would have a rounded edge that would be contrary to the width of the view.

{% highlight objc %}
- (void)layoutSubviews
{
    [super layoutSubviews];
    self.stretchingView.layer.cornerRadius = CGRectGetMidX(self.stretchingView.bounds);
}
{% endhighlight %}

Now all that is left to do is to animate between the two states using our new friend with the long name, <code>animateWithDuration:delay:usingSpringWithDamping:initialSpringVelocity:options:animations:completion:</code>.

We’ve seen most of the parameters used in this method before, but let’s take a quick look at the two that matter to us, <code>usingSpringWithDamping</code> and <code>initialSpringVelocity</code>.

<code>usingSpringWithDamping</code> takes a <code>CGFloat</code> value between 0.0 and 1.0 and determines the oscillation of the spring, in a physical sense, the strength of the spring. A value closer to 1.0 will increase the strength of the spring and result in low oscillation. A value closer to 0.0 will weaken the strength of the spring and result in high oscillation.

<code>initialSpringVelocity</code> also takes a <code>CGFloat</code> however the value passed will be relative to the distance travelled during the animation. A value of 1.0 translates to the total animation distance traversed in 1 second while a value of 0.5 translates to half the animation distance traversed in 1 second.

While these parameters correspond to physical properties, for the most part it’s a case of *if it feels good, do it*.

{% highlight objc %}
[UIView animateWithDuration:0.5f
                      delay:0.0f
     usingSpringWithDamping:0.4f
      initialSpringVelocity:0.5f
                    options:UIViewAnimationOptionBeginFromCurrentState
                 animations:^{
                     self.stretchingView.frame = [self frameForExpandedState];
                 } completion:NULL];
{% endhighlight %}

And that’s it. With just one method call and some waving of magic numbers, we can take advantage of the dynamic underpinnings of UIKit in iOS 7.

## Three's a crowd

Now that we’ve created our <code>SCSpringExpandingView</code>, we'll need to create a view that houses the three <code>SCSpringExpandingView</code>s. Let's call it <code>SCDragAffordanceView</code>.

The basic job of the the <code>SCDragAffordanceView</code> will be to layout the three <code>SCSpringExpandingView</code>'s as well as providing an interface with which we can pass in the progress of pull-for-menu interaction.

To layout our <code>SCSpringExpandingView</code>s, we'll override <code>layoutSubviews</code> and align each of the views frames equally spaced across our bounds.

{% highlight objc %}
- (void)layoutSubviews
{
    [super layoutSubviews];

    CGFloat interItemSpace = CGRectGetWidth(self.bounds) / self.springExpandViews.count;

    NSInteger index = 0;
    for (SCSpringExpandView *springExpandView in self.springExpandViews)
    {
        springExpandView.frame = CGRectMake(interItemSpace * index, 0.f, 4.f,
                                 CGRectGetHeight(self.bounds));
        index++;
    }
}
{% endhighlight %}

Now that we have the views laid out, we'll need to update them when someone calls the <code>setProgress:</code> method. If we look back to [Unread](http://jaredsinclair.com/unread/), we can see three distinct states for each of the spring views: Collapsed, expanded and completed. The first two we've mentioned, but the final is what indicates that the pull-for-menu interaction has reached a point where releasing will trigger the menu to be shown.

To implement this, we'll iterate over our three <code>SCSpringExpandingView</code>s and update the colour of each based primarily on whether the <code>progress</code> passed in is greater or equal to 1.0, followed by whether the <code>progress</code> is great enough that the view should expand.

{% highlight objc %}
- (void)setProgress:(CGFloat)progress
{
    _progress = progress;

    CGFloat progressInterval = 1.0f / self.springExpandViews.count;

    NSInteger index = 0;
    for (SCSpringExpandView *springExpandView in self.springExpandViews)
    {
        BOOL expanded = ((index * progressInterval) + progressInterval < progress);

        if (progress >= 1.f)
        {
            [springExpandView setColor:[UIColor redColor]];
        }
        else if (expanded)
        {
            [springExpandView setColor:[UIColor blackColor]];
        }
        else
        {
            [springExpandView setColor:[UIColor grayColor]];
        }

        [springExpandView setExpanded:expanded animated:YES];
        index++;
    }
}
{% endhighlight %}

Now that we’ve covered some of the new hotness, let’s take a detour down a well travelled road.

## Nested UIScrollView

Ask any iOS developer and they’ll tell you, nested scroll views are *the* user interface element, so much so that Apple have [dedicated a chapter](https://developer.apple.com/library/ios/documentation/windowsviews/conceptual/UIScrollView_pg/NestedScrollViews/NestedScrollViews.html) of their <code>UIScrollView</code> programming guide to the topic. It's criminal that we’ve studied this many innovative iOS interfaces together without mentioning them.

For our example content, we’ll be displaying some riveting Lorem Ipsum with <code>UITextView</code>, a class which received some TextKit love in the great facelift of iOS 7. While we won’t be covering any of the new APIs in this entry, anyone interested should check out the [fantastic writeup on objc.io](http://www.objc.io/issue-5/getting-to-know-textkit.html). Instead, all we need to remember is that <code>UITextView</code> is a subclass of the mighty <code>UIScrollView</code>.

We want our <code>SCDragAffordanceView</code> to always be at hand, ready to present our menu. One option to consider would be to add it as a subview of our <code>UITextView</code> and modify its vertical origin based on the <code>contentOffset</code> of our <code>UITextView</code>, but this overloads the responsibility of our <code>UITextView</code> beyond just displaying text and just *feels* a bit wrong.

Instead lets create a separate instance of <code>UIScrollView</code> which our <code>UITextView</code> and <code>SCDragAffordanceView</code> will be added as subviews of.

{% highlight objc %}
self.enclosingScrollView = [[UIScrollView alloc] initWithFrame:self.view.bounds];
self.enclosingScrollView.alwaysBounceHorizontal = YES;
self.enclosingScrollView.delegate = self;
[self.view addSubview:self.enclosingScrollView];
{% endhighlight %}

The key line here is setting <code>alwaysBounceHorizontal</code> to <code>YES</code>. Now regardless of the <code>contentSize</code> of the scroll view, dragging horizontally will always continue beyond the bounds with the expected resistance.

If our nested <code>UITextView</code>’s horizontal content size doesn’t exceed its bounds, then we’ll achieve the effect of having just one <code>UIScrollView</code>, while separating concerns in our code.

We’ll also want to become the delegate of the scroll view so that we detect the scroll view is being dragged and update our <code>SCDragAffordanceView</code>’s progress accordingly.

{% highlight objc %}
- (void)scrollViewDidScroll:(UIScrollView *)scrollView
{
    if (scrollView.isDragging)
    {
        self.menuDragAffordanceView.progress = scrollView.contentOffset.x /
                                             CGRectGetWidth(self.menuDragAffordanceView.bounds);
    }
}
{% endhighlight %}

Finally when we receive the <code>scrollViewDidEndDragging:willDecelerate:</code> delegate callback, we’ll use that same progress we calculated in the <code>scrollViewDidScroll:</code> callback to determine whether to present our menu view controller. If not, we’ll set our progress back to 0.0.

{% highlight objc %}
- (void)scrollViewDidEndDragging:(UIScrollView *)scrollView willDecelerate:(BOOL)decelerate
{
    if (self.menuDragAffordanceView.progress >= 1.f)
    {
        [self presentViewController:self.menuViewController
                           animated:YES
                         completion:NULL];
    }
    else
    {
        self.menuDragAffordanceView.progress = 0.f;
    }
}
{% endhighlight %}

With that dusty road behind us, let’s get stuck into the next piece of iOS 7 hotness.

## UIViewControllerTransitioningDelegate

What a difference a version makes. Had this post been written before iOS 7, it would have been a much longer and caveat filled affair. Previously if you wanted behaviour like [Unread](http://jaredsinclair.com/unread/)’s pull-for-menu, you'd have to insert your view atop the current view controller, window, or some other equally smelly behaviour. While this would give you the desired effect, it always felt as though you were going against the grain of the framework.

Thankfully in iOS 7, Apple noticed this pattern emerging and took another cue from the developer community, providing a clean, sanctioned way to achieve this using a set of minimal protocols. You can now define custom animations and interactive transitions between view controllers by implementing the <code>UIViewControllerTransitioningDelegate</code> protocol.

This <code>UIViewControllerTransitioningDelegate</code> protocol declares a handful of methods which allow you to return animator objects that define one of the three phases of view transition: presenting, dismissing, and interacting. Our custom transition will be defining the presenting and dismissing phases.

In our view controller, we’ll declare that we conform to the <code>UIViewControllerTransitioningDelegate</code> protocol and implement the two methods we care about, <code>animationControllerForPresentedController:presentingController:sourceController:</code> and <code>animationControllerForDismissedController:</code>.

Now that we provide the callbacks for a custom view controller transition, we need a view controller to present. [Unread](http://jaredsinclair.com/unread/)'s neat menu item animations are out of the scope of this article, so for our case we just need to create a view controller (<code>SCMenuViewController</code>) to be presented when the menu interaction is triggered.

{% highlight objc %}
self.menuViewController = [[SCMenuViewController alloc] initWithNibName:nil bundle:nil];
{% endhighlight %}

Once we’ve created an instance of this class, we need to set its <code>transitionDelegate</code> to be our view controller and set its <code>modalPresentationStyle</code> to <code>UIModalPresentationCustom</code> so that it calls back to its <code>transitioningDelegate</code> when presented.

{% highlight objc %}
self.menuViewController.modalPresentationStyle = UIModalPresentationCustom;
self.menuViewController.transitioningDelegate = self;
{% endhighlight %}

Now when we present our menu view controller, it will callback to its <code>transitioningDelegate</code> (our view controller) to ask for the presenting <code>UIViewControllerAnimatedTransitioning</code> animator object.

## UIViewControllerAnimatedTransitioning

To provide our animator objects to our menu view controller, we’ll start by creating a plain old NSObject subclass called <code>SCOverlayPresentTransition</code>, and declare that it conforms to the <code>UIViewControllerAnimatedTransitioning</code> protocol. In our <code>animationControllerForPresentedController:presentingController:sourceController:</code> delegate callback, we’ll create an instance of our <code>SCOverlayPresentTransition</code> object and return it.

{% highlight objc %}
- (id<UIViewControllerAnimatedTransitioning>)animationControllerForPresentedController:(UIViewController *)presented presentingController:(UIViewController *)presenting sourceController:(UIViewController *)source
{
    return [[SCOverlayPresentTransition alloc] init];
}
{% endhighlight %}

For the dismissal animation, we’ll create another NSObject subclass called <code>SCOverlayDismissTransition</code> and provide an instance of it when we receive the <code>animationControllerForDismissedController:</code> delegate callback.

{% highlight objc %}
- (id<UIViewControllerAnimatedTransitioning>)animationControllerForDismissedController:(UIViewController *)dismissed
{
    return [[SCOverlayDismissTransition alloc] init];
}
{% endhighlight %}

The implementation of our present and dismiss transition objects consist of two methods, <code>transitionDuration:</code> and <code>animateTransition:</code>. The <code>transitionDuration:</code> method as you may have guessed simply requests an <code>NSTimeInterval</code> to dictate the duration of the animation. The <code>animateTransition:</code> is where the real work of the transition is done.

The sole parameter of the <code>animateTransition:</code> method is an object conforming to the <code>UIViewControllerContextTransitioning</code> protocol. From this object we can pluck out the objects and information we need to drive our animation, including the view controllers involved in the transition. It also provides methods that we’ll use to notify the framework that we’ve completed our transition.

{% highlight objc %}
UIViewController *presentingViewController = [transitionContext viewControllerForKey:UITransitionContextFromViewControllerKey];
UIViewController *overlayViewController = [transitionContext viewControllerForKey:UITransitionContextToViewControllerKey];
{% endhighlight %}

Once we have the presenting and presented view controllers, we need to add their views as subviews of the transition’s container view so that they both appear during the animation.

{% highlight objc %}
UIView *containerView = [transitionContext containerView];
[containerView addSubview:presentingViewController.view];
[containerView addSubview:overlayViewController.view];
{% endhighlight %}

The final piece of the presenting transition is to simply animate the views however we fancy, then notify the <code>transitionContext</code> object whether we’ve completed our transition successfully.

{% highlight objc %}
overlayViewController.view.alpha = 0.f;
NSTimeInterval transitionDuration = [self transitionDuration:transitionContext];
[UIView animateWithDuration:transitionDuration
                  animations:^{
                     overlayViewController.view.alpha = 0.9f;
                 } completion:^(BOOL finished) {
                     BOOL transitionWasCancelled = [transitionContext transitionWasCancelled];
                     [transitionContext completeTransition:transitionWasCancelled == NO];
                 }];
{% endhighlight %}

The <code>SCOverlayDismissTransition</code> will follow largely the same process, albeit in the opposite direction.

Now when our menu view controller is presented it will use our custom transition, keeping the presenting view controller’s view in the view hierarchy.

## Closing

<video src="/media/2014-03-22-unread-overlay-menu/video/finished.m4v" autoplay="true" loop="true"></video>

As we approach the 6th anniversary of the iOS App Store its amazing how far the app landscape has come. The idea that we can already consider apps as classics is an indication of just how fast it's moving. Every year developers are given a new collection of toys to build great apps with, yet there is still room for the venerable <code>UIScrollView</code>.

You can [checkout this project on GitHub](https://github.com/subjc/SubjectiveCUnreadMenu).

[^1]: If you're feeling nostalgic for the heady iOS 6 days, there's a great clone of the iOS 6 pull-to-refresh control [on GitHub](https://github.com/Sephiroth87/ODRefreshControl)
