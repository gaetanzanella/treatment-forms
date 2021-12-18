---
layout: post
title: "UIKit rendering - Tracking a layout phase"
author:
  name: "Gaétan Zanella"
  link: "https://twitter.com/gaetanzanella"
tags: iOS
---

On iOS, the layout of a view hierarchy is expressed dynamically using the Auto Layout constraint system. Thus a view hierarchy can adapt itself to all the possible device screens sold by Apple. Auto Layout takes care to translate all the constraints into equations where the positions and the sizes of the views are the variables. As a consequence the rendering of a view hierarchy requires a calculation before each screen refresh.

Let's find out when the equations solving takes place.

## The layout phase

In the life cycle of an application, Apple calls these equations solving moments "layout passes" without ever formally describing them. They were mentioned during a [WWDC 2018 talk](https://developer.apple.com/videos/play/wwdc2018/220/) which specified the order and objectives of the different calculations.

To avoid repetitive and useless calculations, the layout phases always appear after the execution of our code. Whether it is following a touch, a network call or the appearance of a view controller, the modification of a constraint affects the position or the size of its associated view only later.

{% highlight swift %}

myView.frame.size.height = 30
myView.translatesAutoresizingMaskIntoConstraints = false
myView.frame.size.height == 30 // true
myView.heightAnchor.constraint(equalToConstant: 50).isActive = true
myView.frame.size.height == 30 // true, changing the constraint has not changed the view height
{% endhighlight %}

Therefore we need to have a better understanding of the iOS application code execution path to find out the timing of the layout computation. How does Auto Layout manage to insert the layout computation at the right time and in such an efficient way?

Let's start by tracking a layout phase!

## Tracking a layout phase

### The layoutSubviews method

Apple discourages us to call the method `layoutSubviews` directly. The UIKit library calls it for us during a layout phase.

Its role is to give to each view its right size and position. It is in this method that the equations defined by our constraints are solved.

During a layout phase, UIKit calls successively (not recursively), and from top to bottom, `layoutSubviews` on each of the views that compose the current view tree. At the end of the chain execution, each view has its final position and size.

Subclassing `layoutSubviews` allows us to take part of the view layout calculations and modifing the default implementation. This is also the right time to modify the elements that depend on the size of the current view.

{% highlight swift %}

override func layoutSubviews() {
    super.layoutSubviews()
    layer.cornerRadius = bounds.height / 2
}
{% endhighlight %}

![Corner radius](/assets/uikit-rendering-part-1/corner_radius.jpeg)

### Triggering a layout phase

UIKit is not the only one to be able to trigger layout calculations. The `layoutIfNeeded` method allows to explicitly trigger a layout phase at any time. We thus get the position of the views that we manipulate during the execution of our code:

{% highlight swift %}

myView.heightAnchor.constraint(equalToConstant: 50).isActive = true
 myView.window?.layoutIfNeeded() // explicit layout phase
 myView.frame.size.height == 30 // false

{% endhighlight %}

We usually use it to animate changes made by a constraint:

{% highlight swift %}

sampleViewHeightConstraint.constant = 50 
UIView.animate(withDuration: 1) { 
    self.view.layoutIfNeeded() 
}

{% endhighlight %}

Indeed, as a *layout phase* triggers successive calls to the `layoutSubviews` of the views on screen, the size and position of those views are modified in the animation block. The resulting animation mechanisms are the same as those that would have been triggered if the view had been explicitly modified:

{% highlight swift %}

UIView.animate(withDuration: 1) {
    self.view.frame.size.height = 50
}

{% endhighlight %}

![Layout animation](/assets/uikit-rendering-part-1/layout_animation.gif)

### Catching an implicit layout phase

Our first goal will be to intercept a call to a `layoutSubviews` method implicitly triggered by UIKit. Indeed, we don't need to call `layoutIfNeeded` every time we modify a constraint, someone or something is calling it for us. If we manage to do so, all we have to do is to determine when this call is performed to answer our initial question: when does a layout phase take place?

Let's start with the following sample code and two breakpoints A and B:

{% highlight swift %}

class SampleView: UIView {  
    override func layoutSubviews() { 
        super.layoutSubviews() // breakpoint B
         layer.cornerRadius = bounds.height / 2
     } 
}

  class ViewController: UIViewController {
      @IBOutlet var sampleView: SampleView!
     @IBOutlet var sampleViewHeightConstraint: NSLayoutConstraint!  

    @IBAction func buttonAction(_ sender: Any) {
         sampleViewHeightConstraint.constant = 20 // breakpoint A
     } 
}

{% endhighlight %}

We added a button and a sample view to a `UIViewController` view in a xib and we connected them using an `IBAction` and an `IBOutlet`. We also added a reference to a height constraint attached the sample view.

![Implicit transactions](/assets/uikit-rendering-part-1/implicit_transaction.gif)

When the button is pressed, the height constraint of the sample view is modified. The stack of our first breakpoint A looks like:

{% highlight text %}

#0: Layout ViewController.buttonAction(sender=Any @ 0x00007ffeee5e00b8, self=0x00007ff0a1f03e00) at ViewController.swift:30
 #1: Layout @objc ViewController.buttonAction(_:) at ViewController.swift:0
 #2: UIKit -[UIApplication sendAction:to:from:forEvent:] + 83
 #3: UIKit -[UIControl sendAction:to:forEvent:] + 67
 #4: UIKit -[UIControl _sendActionsForEvents:withEvent:] + 450
 #5: UIKit -[UIControl touchesEnded:withEvent:] + 580
 #6: UIKit -[UIWindow _sendTouchesForEvent:] + 2729 
#7: UIKit -[UIWindow sendEvent:] + 4086
 #8: UIKit -[UIApplication sendEvent:] + 352 
#9: UIKit __dispatchPreprocessedEventFromEventQueue + 2796
 #10: UIKit __handleEventQueueInternal + 5949
 #11: CoreFoundation __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__ + 17
 #12: CoreFoundation __CFRunLoopDoSources0 + 271
 #13: CoreFoundation __CFRunLoopRun + 1263 
#14: CoreFoundation CFRunLoopRunSpecific + 635
 #15: GraphicsServices GSEventRunModal + 62
 #16: UIKit UIApplicationMain + 159 
#17: Layout main at AppDelegate.swift:12
 #18: libdyld.dylib start + 1
 #19: libdyld.dylib start + 1

{% endhighlight %}

By continuing the execution, the breakpoint B placed in the `layoutSubviews` of the sample view is triggered. By modifying the constraint, UIKit has been forced to trigger a layout phase to satisfy our new layout. We are in the presence of an implicit layout phase!

{% highlight text %}

#0: Layout View.layoutSubviews(self=0x00007ff0a1d0ca20) at ViewController.swift:15
#1: Layout @objc View.layoutSubviews() at ViewController.swift:0
#2: UIKit -[UIView(CALayerDelegate) layoutSublayersOfLayer:] + 1515
#3: QuartzCore -[CALayer layoutSublayers] + 177
#4: QuartzCore CA::Layer::layout_if_needed(CA::Transaction*) + 395
#5: QuartzCore CA::Context::commit_transaction(CA::Transaction*) + 343
#6: QuartzCore CA::Transaction::commit() + 568
#7: QuartzCore CA::Transaction::observer_callback(__CFRunLoopObserver*, unsigned long, void*) + 76
#8: CoreFoundation __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__ + 23
#9: CoreFoundation __CFRunLoopDoObservers + 430
#10: CoreFoundation __CFRunLoopRun + 1537
#11: CoreFoundation CFRunLoopRunSpecific + 635
#12: GraphicsServices GSEventRunModal + 62
#13: UIKit UIApplicationMain + 159
#14: Layout main at AppDelegate.swift:12
#15: libdyld.dylib start + 1
#16: libdyld.dylib start + 1

{% endhighlight %}

Except the `layoutSubviews` method, there is no trace of our code. This stack is totally distinct from the one executed when we pressed the button.

The goal is clear: avoiding unnecessary calculations. By running the layout phase on a later and separate stack, UIKit ensures that the layout calculations take place after all modifications that could affect the position of the views have been made.

But where does this stack come from? We could imagine a more complicated scenario: we could modify a constraint in a block in the `dispatchAsync` method. How does UIKit manage to always insert the layout phase stack so well?

### The causes of a layout phase

Let's analyse the elements of the stack.

The top of the stack is familiar. `CALayer.layoutSublayers`, for example, is a public method available since iOS 2. It is obviously equivalent to `layoutSubviews` : `CALayerDelegate` is the link between an `UIView` and a `CALayer`. We guess that the layout of a view involves the layout of its layer.

The lower part of the stack is the one we are really interested in. It contains the first moments of the layout phase.

{% highlight text %}

#6: QuartzCore CA::Transaction::commit() + 568
#7: QuartzCore CA::Transaction::observer_callback(__CFRunLoopObserver*, unsigned long, void*) + 76
#8: CoreFoundation __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__ + 23
#9: CoreFoundation __CFRunLoopDoObservers + 430
#10: CoreFoundation __CFRunLoopRun + 1537
#11: CoreFoundation CFRunLoopRunSpecific + 635
#12: GraphicsServices GSEventRunModal + 62
#13: UIKit UIApplicationMain + 159
#14: Layout main at AppDelegate.swift:12
#15: libdyld.dylib start + 1
#16: libdyld.dylib start + 1

{% endhighlight %}

At the origin of the layout calculation of our view, we find a `commit` method of a mysterious `CATransaction` class. Below, we see methods related to an enigmatic run loop: `UIApplicationMain`, `CFRunLoopRun` and `CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_PERFORM_FUNCTION`.

The run loop is an essential component of all iOS applications. It is a safe bet that the success of our exploration depends on its understanding and its connection with the `CATransaction` class.

---

The steps of our exploration are defined:
- What is a `CATransaction`?
- What is a run loop?
- How are they linked?

On the next post, we will tackle the first topic: [CATransaction](/2021-01-04/uikit-rendering-part-2).
