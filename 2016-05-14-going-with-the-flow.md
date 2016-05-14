---
layout: post
title:  "Going with the flow"
date:   2016-04-14 15:15:00
description: "A look at UICollectionViewLayout"
author:
  name: Matt Delves
  url: https://twitter.com/mattdelves
---

For a long time, UICollectionViewLayout was a class that caused me to shy away from picking up the particular task in creating the UI. Yet, they are a common part in any innovative iOS interface. If you want to present a collection of items that vary in size or width, then a custom layout is the way to go. There are many different ways of creating this style of layout, though I'm going to focus on the approach of not using auto layout to determine the size of the cell.

If you search google, you'll find a lot of articles presenting what is known as the 'Pinterest' layout. That is, a layout with two columns and the height of the cells are different. We'll be creating a form of this layout and then taking it a step further and applying some UIKit dynamics so that the flow of the cells when you scroll is fluid. There'll be a small spring as the cell rests in place after scrolling has stopped.

A great way of getting to understand how this all fits together is to use a Playground. These are perfect for changing values and seeing the result straight away. I've included a playground with this post that is available on GitHub.

## FlowLayout or Layout?

This is a question that many people ask. When you drag a UICollectionViewController onto the storyboard in Interface Builder, or not if you don't like storyboards, then you'll end up with a UICollectionViewFlowLayout being the default layout. For 99% of your needs, this will suffice. You can get away with a lot by just using this layout.

We are though wanting to be innovative and for that, we are going to create a subclass of UICollectionViewLayout. I would highly recommend reading Apple's documentation about [UICollectionViewLayout](https://developer.apple.com/library/ios/documentation/UIKit/Reference/UICollectionViewLayout_class/) as it explains what we need to do to create our own layout.

## Cells, Supplementary Views, Decorator Views

For a UICollectionView there are three types of view that can be displayed. These are Cells, Supplementary Views and Decorator Views.

### UICollectionViewCell

This is what displays the different items in the collection. They display your data on screen.

When working with cells in your layout, there are two functions you need to implement. These are:

* `layoutAttributesForItemAtIndexPath:`
* `layoutAttributesForElementsInRect:`

These are the two main functions that provide the collection view with the information it needs to lay itself out.

### Supplementary Views

These are used for the header and footer of a section in a collection view. They are simple UIViews that get dequed as required by the collection view data source. When creating the view, make sure you implement prepareForReuse so that the views can be reused. UIKit does an amazing job of keeping track of when to reuse a view, there is often no reason to go about implementing your own cell reuse.

If you want to use Supplementary Views as part of your layout, then you need to implement the function `layoutAttributesForSupplementaryViewOfKind:atIndexPath:` in your layout subclass. We aren't going to be considering these views in this post.

### Decorator Views

These views exist to provide decoration around items in your collection view. As an example, if you wanted to place an image after every 5th item in the collection view then a decorator view would be what you require.

With regard to your layout, you will need to implement the function `layoutAttributesForDecorationViewOfKind:atIndexPath:`. We aren't going to be considering these views in this post.

## Layout Attributes

It can be observed that I've made some mention of this thing called Layout Attributes in the function calls mentioned above. Layout Attributes are what provides the collection view with information about the origin and size of the item.

You will be dealing with the class `UICollectionViewLayoutAttributes` when creating the layout. For the layout being created, you should specify its `frame` attribute. This will tell the collection view where the origin of the cell is and also how large it is.

When creating the instance of UICollectionViewLayoutAttributes you need to initialize it with the index path of the item it represents. For a cell you use the initializer `init(forCellWithIndexPath indexPath: NSIndexPath)`. There are corresponding initializers for supplementary and decoreator views as well.

Within the playground you'll see that the layout attributes are created by the following code:

{% highlight swift %}
let staticAttributes: [UICollectionViewLayoutAttributes] = newlyVisible.map { path in
  let attributes = UICollectionViewLayoutAttributes(forCellWithIndexPath: path)
  let size = dataSource.cellSizes[path.item]
  let origin = dataSource.cellOrigins[path.item]
  attributes.frame = CGRect(origin: origin, size: size)

  return attributes
}
{% endhighlight %}

To provide some context to this piece of code, we have an array of NSIndexPath's that are now visible on the screen and creating an array of UICollectionViewLayoutAttributes. We then assign this to a variable `staticAttributes` that gets used later on.

We are fetching the details of the size and origin of the cell from the data source.

## Calculating the cell positions

We just did some hand wavy magic to set the layout attributes. Now we need to unpack it and show how we achieve the desired outcome.

A key part to designing software, is to seperate the conceerns of our classes. In this case, it is the responsibility of the type conforming to `UICollectionViewDataSource` to provide the details about the data. The size and origin of a cell are a good fit for this type.

In order to avoid repeatedly calculating everything, we store everything when we create the data source. If the data source changes, then we will need to do these calculations again.

The first bit of calculation we're going to do is create some items that have unique hieght. We'll also store the size and the origin of each cell as we calculate it.

{% highlight swift %}
let items = (0..<100).map {_ in
    return Item(
        height: randomHeight()
    )
}

let cellOrigins: [CGPoint]
let cellSizes: [CGSize]
{% endhighlight %}

When the data source is initialised, we go and calculate everything. This looks like:

{% highlight swift %}
override init() {
    var tempOrigins = [CGPoint]()
    var tempSizes = [CGSize]()
    var leftHeight: CGFloat = 16.0
    var rightHeight: CGFloat = 16.0
    let padding: CGFloat = 32.0
    let leftOrigin: CGFloat = 16.0
    let rightOrigin: CGFloat = 200.0

    items.enumerate().forEach { index, event in
        var x: CGFloat = leftOrigin
        var y: CGFloat = 0.0
        let width: CGFloat = 150.0
        let height: CGFloat = event.height

        if rightHeight > leftHeight {
            y = leftHeight
            leftHeight += event.height + padding
        } else {
            x = rightOrigin
            y = rightHeight
            rightHeight += event.height + padding
        }

        tempOrigins.append(CGPoint(x: x, y: y))
        tempSizes.append(CGSize(width: width, height: height))
    }

    cellOrigins = tempOrigins
    cellSizes = tempSizes

    super.init()
}
{% endhighlight %}

## Content Size

For a collection view to scroll correctly, it needs to know how large it is. This is known as the content size and it is the responsibility of the UICollectionViewLayout subclass to inform the collection view about it. The layout has the function `collectionViewContentSize` that must be implemented. As we have done the hard work of calculating the size and origin of each cell when we create the data source, we can iterate over those values to get the total size of the collection view.

We are also saving this value so that we don't need to constantly recalculate it as we modify the layout.

{% highlight swift %}
override func collectionViewContentSize() -> CGSize {
    if staticContentSize != CGSizeZero {
        return staticContentSize
    }

    guard let collectionView = collectionView,
          let dataSource: TestDataSource = collectionView.dataSource as? TestDataSource else { return CGSizeZero }
    var maxY: CGFloat = 0.0
    (0..<dataSource.items.count).forEach { index in
        let originY = dataSource.cellOrigins[index].y
        let height = dataSource.cellSizes[index].height
        let newMax = originY + height
        if newMax > maxY {
            maxY = newMax
        }
    }

    staticContentSize = CGSize(width: 320, height: maxY + 10)

    return staticContentSize
}
{% endhighlight %}

## UIKit Dynamics

We now have some idea of how this all fits together, though it is just static. There is now fluid nature to the cells. We achieve this by using UIKit Dynamics. As part of UIKit there is a class called UIDynamicAnimator that allows us to change our static UICollectionViewLayoutAttributes into ones that have been effected by the dynamic animator.

After we have created our static attributes, we then loop over each of them, create a spring behaviour (an instance of UIAttachmentBehavior) and add them to the UIDynamicAnimator instance.

{% highlight swift %}
let touchLocation = collectionView.panGestureRecognizer.locationInView(collectionView)

staticAttributes.forEach { attributes in
  let center = attributes.center
  let spring = UIAttachmentBehavior(item: attributes, attachedToAnchor: center)
  spring.length = 0.5
  spring.damping = 0.1
  spring.frequency = 1.5

  if (!CGPointEqualToPoint(CGPointZero, touchLocation)) {
    let yDistanceFromTouch = touchLocation.y - spring.anchorPoint.y
    let xDistanceFromTouch = touchLocation.x - spring.anchorPoint.x
    let scrollResistance = (yDistanceFromTouch + xDistanceFromTouch) / 1500.0
    var center = attributes.center
    if (latestDelta < 0) {
      center.y += max(latestDelta, latestDelta * scrollResistance);
    } else {
      center.y += min(latestDelta, latestDelta * scrollResistance);
    }
    attributes.center = center
  }

  dynamicAnimator.addBehavior(spring)
}
{% endhighlight %}

We're now at a point where we can tell the layout what the attributes are which exist within a rect or a series of index paths. As we've added in a dynamic animator we will be using the values from it to tell the collection view where things are on screen.

This is done by implementing `layoutAttributesForElementsInRect` and `layoutAttributesForItemAtIndexPath` as follows.

{% highlight swift %}
override func layoutAttributesForElementsInRect(rect: CGRect) -> [UICollectionViewLayoutAttributes]? {
    return dynamicAnimator?.itemsInRect(rect).map {
        ($0 as? UICollectionViewLayoutAttributes)!
    }
}

override func layoutAttributesForItemAtIndexPath(indexPath: NSIndexPath) -> UICollectionViewLayoutAttributes? {
    return dynamicAnimator?.layoutAttributesForCellAtIndexPath(indexPath)
}
{% endhighlight %}

If we were to leave things as such, we would find that the dynamics didn't work. This is because we need to change things as they scroll. The method `` is what we will use. It allows us to see when a change happens from a scroll event and then modify our dynamic behaviors accordingly.

{% highlight swift %}
override func shouldInvalidateLayoutForBoundsChange(newBounds: CGRect) -> Bool {
    guard let collectionView = collectionView,
          let dynamicAnimator = dynamicAnimator else { return false }

    let delta = newBounds.origin.y - collectionView.bounds.origin.y
    latestDelta = delta

    let touchLocation = collectionView.panGestureRecognizer.locationInView(collectionView)
    dynamicAnimator.behaviors.forEach { behavior in
    if let springBehaviour = behavior as? UIAttachmentBehavior, let item = springBehaviour.items.first {
      let yDistanceFromTouch = touchLocation.y - springBehaviour.anchorPoint.y
      let xDistanceFromTouch = touchLocation.x - springBehaviour.anchorPoint.x
      let scrollResistance = (yDistanceFromTouch + xDistanceFromTouch) / 1500.0
      var center = item.center
      if (delta < 0) {
        center.y += max(delta, delta*scrollResistance);
      } else {
        center.y += min(delta, delta*scrollResistance);
      }
      item.center = center
      dynamicAnimator.updateItemUsingCurrentState(item)
    }
  }
  return false
}
{% endhighlight %}


### There can be only one

It needs stating that for each item in the collection view, there can ever only be one behavior in the dynamic animator. When we prepare the layout for use, we need to go through and make sure we aren't adding the same behaviour to the dynamic animator. The code to achieve this looks like:

{% highlight swift %}
let visibleRect = CGRectInset(CGRect(origin: collectionView.bounds.origin, size: collectionView.frame.size), 0, -100)
let visiblePaths = indexPaths(visibleRect)
var currentlyVisible: [NSIndexPath] = []

dynamicAnimator.behaviors.forEach { behavior in
  if let behavior = behavior as? UIAttachmentBehavior,
     let item = behavior.items.first as? UICollectionViewLayoutAttributes {
    if !visiblePaths.contains(item.indexPath) {
      dynamicAnimator.removeBehavior(behavior)
    } else {
      currentlyVisible.append(item.indexPath)
    }
  }
}

let newlyVisible = visiblePaths.filter { path in
  return !currentlyVisible.contains(path)
}
{% endhighlight %}

Here we loop over the behaviours that we have created and then depending on whether they should or shouldn't be there either remove them or add the index path to the list that is currently visible. We then find those who are newly visible and as such require behaviours created for them.

## Bringing it all together.

So we've seen much of what exists in the UICollectionViewLayout and also how we can calculate things in beforehand. As you get your hands dirty with collection view layouts, you'll begin to see how you can improve upon what I've provided here and make an inovative layout as a result.

There's a lot of content in this post and much which I've waved my hand over. Fear not though as the playground for this post gives you every freedom to see just how it all ties together.

Till next time.
