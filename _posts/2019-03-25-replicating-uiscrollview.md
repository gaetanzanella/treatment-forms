---
layout: post
title: "Replicating UIScrollView"
author:
  name: "Gaétan Zanella"
  link: "https://twitter.com/gaetanzanella"
tags: iOS
---

[UIScrollView](https://developer.apple.com/documentation/uikit/uiscrollview) might be one of the most sentitive part of the iOS ecosystem. It provides the overall solution to display large content on the small screens of the iOS devices. In fact, `UIScrollView` is rarely used as-it. It is the superclass of several `UIKit` classes including `UITableView` and `UITextView`. You could find thousands of tutorials on the web on how to easily take advantage of it.

In this article, I would like to go a little bit deeper and expound on how an `UIScrollView` actually works. To do so, we will try to create a humble copy of it. Our starting point is a basic `UIView` subclass:

```swift
class MyScrollView: UIView {

}
```

Of course, `UIScrollView` has a lot of features: paging, content inset, layout guides, content touches delay etc. We will not be able to reproduce all of them here. We will focus our efforts on the main `UIScrollView` abilities:

- A scroll view has a content 
- Its content is defined by its size
- An user can scroll the content by _dragging_
- A scroll view adds animations to make the interactions look and feel natural

Let's get started!

## Scroll View Content

If you already play a bit with a scroll view, you already know that its content is simply its subviews. Let's add some subviews!


```swift
class MyScrollView: UIView {

    override init(frame: CGRect) {
        super.init(frame: frame)
        setUpView()
    }

    private func setUpView() {
        let colors: [UIColor] = [.purple, .red]
        let height: CGFloat = 200
        for i in 0..<100 {
            let view = UIView(
                frame: CGRect(
                    x: 0, 
                    y: CGFloat(i) * height, 
                    width: bounds.width, 
                    height: height
                )
            )
            view.backgroundColor = colors[i%colors.count]
            addSubview(view)
        }

    }
}
```

## IMAGE ##

## Scrolling

How to slide subviews ?

We first need to detect the user interactions. The answer is the `UIScrollView` API. It has a public [pan gesture recognizer](https://developer.apple.com/documentation/uikit/uiscrollview/1619425-pangesturerecognizer) that looks for dragging gestures. Let's add it in our own subclass:

```swift
class MyScrollView: UIView {

    let panGestureRecognizer = UIPanGestureRecognizer()

    override init(frame: CGRect) {
        super.init(frame: frame)
        setUpView()
    }

    @objc private func panGestureUpdate(_ sender: UIPanGestureRecognizer) {
        // ...
    }

    private func setUpView() {
        // ...
        panGestureRecognizer.addTarget(self, action: #selector(panGestureUpdate(_:)))
        addGestureRecognizer(panGestureRecognizer)
    }
}
```

There are mulitple ways to slide subviews. A first naive `panGestureUpdate` implementation may look like this:

```swift
class MyScrollView: UIView {

    @objc private func panGestureUpdate(_ sender: UIPanGestureRecognizer) {
        let translation = sender.translation(in: self).y
        switch sender.state {
        case .changed:
            subviews.forEach { $0.frame.origin.y += translation }
        case ...:
            break
        }
        sender.setTranslation(.zero, in: self)
    }
}
```

Each time `panGestureUpdate` is triggered, we update the frame for each of the subviews. It works!

## IMAGE ##

But, this solution has an obvious drawback. This `forEach` might have a huge performance cost. We can do way better knowing some ordinary `UIView` features.

A view is placed relatively to its _superview’s coordinate system_. This latter has the origin at its top left by default but we can easily change it:

```swift
class MyScrollView: UIView {

    @objc private func panGestureUpdate(_ sender: UIPanGestureRecognizer) {
        let translation = sender.translation(in: self).y
        switch sender.state {
        case .changed:
            bounds.origin.y += translation
        case ...:
            break
        }
        sender.setTranslation(.zero, in: self)
    }
}
```

Indeed `bounds.origin` describes the view’s location in its own coordinate system. Changing it modifies the scroll view coordinate system's origin and slides all its subviews accordingly.

In order to stick a bit more to the original API, we can now define the `contentOffset` property:

```swift
class MyScrollView: UIView {

    var contentOffset: CGPoint {
        set {
            bounds.origin = newValue
        }
        get {
            return bounds.origin
        }
    }

    @objc private func panGestureUpdate(_ sender: UIPanGestureRecognizer) {
        let translation = sender.translation(in: self).y
        switch sender.state {
        case .changed:
            contentOffset.y += translation
        case ...:
            break
        }
        sender.setTranslation(.zero, in: self)
    }
}
```

It is a simple computed property which directly accesses the bounds's origin.

## IMAGE ##

## Content Size

For now the content has no limit. We should fix that.

`UIScrollView` has a `contentSize` property which definese how far it can slide its subviews. Let's add it:

```swift
class MyScrollView: UIView {

    var contentSize: CGSize = .zero

    @objc func panGestureAction(_ sender: UIPanGestureRecognizer) {
        // ...
        var offset = contentOffset
        offset.y -= translation
        contentOffset = _rubberBandContentOffset(forOffset: offset)
        // ...
    }

    func _rubberBandContentOffset(forOffset offset: CGPoint) -> CGPoint {
        return max(min(contentSize.height - bounds.height, contentOffset.y), 0)
    }
}
```

The limit is reached when the scroll view’s bounds origin is `CGPoint.zero` or when the entire content has scrolled.

## Animations

Interacting with a native scroll view is great. All the motions feel natural. When we reach the end of content or when we left our finger, the content doesn't scroll lineary. 

To known how far it should be allowed to slide its subviews upward and leftward. That is the scroll view’s content size — its contentSize property.
