---
layout: post
title: "UIKit rendering - The run loop"
author:
  name: "Gaétan Zanella"
  link: "https://twitter.com/gaetanzanella"
tags: iOS
---

On the [previous post](/2021-01-04/uikit-rendering-part-2), we discovered that the commit of the current implicit transaction is responsible of our initial interrupted layout phase. So we only have one question left: when does it take place?

The answer is related to an important concept on iOS: the run loop.

I will refer to those three publications in the following article:

- [Run Loop Programming Guide ](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/Multithreading/RunLoopManagement/RunLoopManagement.html)
- [Nicolas Bouilleaud - Run, RunLoop, Run!](https://bou.io/RunRunLoopRun.html)
- [Rob on StackOverflow - How do you schedule a block to run on the next run loop iteration? ](https://stackoverflow.com/questions/15161434/how-do-you-schedule-a-block-to-run-on-the-next-run-loop-iteration/15168471#15168471)

### Role of a run loop

The run loop is the mechanism that differentiates a command line program from an interactive application.

A run loop can be vizualised as an infinite loop attached to a thread waiting perpetually for an event. When an event comes, the run loop executes the block code associated to it (if any) on the thread and put the thread back to sleep once the work is done.

On iOS, a run loop can be attached to a `NSThread`. Its role is to ensure that its `NSThread` is busy when there is work to do and at rest when there is none.

The main thread automatically launches its run loop at the application launch.

### Implementing a run loop

{% highlight swift %}

func postMessage(runloop, message) {
    runloop.queue.pushBack(message)
    runloop.signal()
}

func run(runloop) {
    do {
        runloop.wait()
        message = runloop.queue.popFront()
        dispatch(message)
    } while(true)
}

{% endhighlight %}

This pseudo code comes from Nicolas' article. It highlights the two main actions of a run loop:

- A run loop waits for an event
- It dispatches it once received

### The run loop on iOS

On iOS, there are of two types of event: timers and sources.

The sources basically correspond to events that an application can handle: a touch, a network call that ends, etc. They are provided by the system.

Rob Mayoff, on his StackOverflow post describes each of the steps performed by a run loop on iOS:

{% highlight text %}

while (true) { 
    Call kCFRunLoopBeforeTimers observer callbacks; 
    Call kCFRunLoopBeforeSources observer callbacks;
     Perform blocks queued by CFRunLoopPerformBlock; 
    Call the callback of each version 0 CFRunLoopSource that has been signalled;
     if (any version 0 source callbacks were called) {
         Perform blocks newly queued by CFRunLoopPerformBlock; 
    } 
    if (I didn't drain the main queue on the last iteration
         AND the main queue has any blocks waiting) { 
        while (main queue has blocks) { 
            perform the next block on the main queue
         } 
    } else {
         Call kCFRunLoopBeforeWaiting observer callbacks;
         Wait for a CFRunLoopSource to be signalled
           OR for a timer to fire
           OR for a block to be added to the main queue;
         Call kCFRunLoopAfterWaiting observer callbacks;
         if (the event was a timer) {
             call CFRunLoopTimer callbacks for timers that should have fired by now
         } else if (event was a block arriving on the main queue) {
             while (main queue has blocks) {
                 perform the next block on the main queue
             } 
        } else {
             look up the version 1 CFRunLoopSource for the event
             if (I found a version 1 source) {
                 call the source's callback
             } 
        } 
    }
     Perform blocks queued by CFRunLoopPerformBlock; 
}

{% endhighlight %}

We find our initial representation of the run loop as a simple infinite loop alongisde all the different operations and checks that it performs.

Thanks to the `Core Foundation` developer teams, anytime a block is executed during the cycle of a run loop, it is always called by a function with very distinctive prototypes:

{% highlight swift %}

static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__();
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_TIMER_CALLBACK_FUNCTION__();
static void __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__();
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__();
static void __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE1_PERFORM_FUNCTION__();

{% endhighlight %}

It helps debugging. We know where we are in the run loop pass. To get the full detailed list, please refer to Nicolas' [article](https://bou.io/RunRunLoopRun.html).

### The run loop observers

In the [Rob's post](https://stackoverflow.com/questions/15161434/how-do-you-schedule-a-block-to-run-on-the-next-run-loop-iteration/15168471#15168471), you may have noticed those "observers callbacks". It is indeed possible to observe the run loop, alongside scheduling blocks, to be notified at a desired time.

The observer blocks are always performed by the debugging function:

{% highlight c %}

static void __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__();

{% endhighlight %}

## CoreAnimation and Run Loop

What is the link between a run loop and a `CATransaction`?

As stated by Rob Mayoff, `CoreAnimation` is an active observer of the run loop. It waits for the `kCFRunLoopBeforeWaiting` event. This latter informs `CoreAnimation` of the standby of the run loop. Thus at that moment, `CoreAnimation` is sure that all the modifications made during the current run loop pass have been scheduled. It can commit the implicit transaction it started, if any, when a first change was made during the loop to display the changes.

We can verify this statement by looking again at the bottom of our stacks of our breakpoints A and B of the first part of the article.

In A, a touch (a source) woke up the run loop. The touch has been dispatched and triggered the action of our button.

{% highlight text %}

#0: Layout ViewController.buttonAction(sender=Any @ 0x00007ffeee5e00b8, self=0x00007ff0a1f03e00) at ViewController.swift:30
#1: Layout @objc ViewController.buttonAction(_:) at ViewController.swift:0
...
#11: CoreFoundation __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__ + 17 …

{% endhighlight %}

At that moment, the modification of the sample view height constraint forced `CoreAnimation` to start an implicit transaction.

The loop continued, triggering its various observers along its way. `CoreAnimation`, as a skillful observer, committed the implicit transaction it started at the action call just before the run loop finished its loop.

{% highlight text %}

#0: Layout View.layoutSubviews(self=0x00007ff0a1d0ca20) at ViewController.swift:15
#1: Layout @objc View.layoutSubviews() at ViewController.swift:0
...
#6: QuartzCore CA::Transaction::commit() + 568
...
#8: CoreFoundation __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__ + 23 ...

{% endhighlight %}

We can generalize this behavior by observing the symbolic C++ breakpoint:

{% highlight swift %}
CA::Context::commit_transaction(CA::Transaction*)
{% endhighlight %}

![Commits](/assets/uikit-rendering-part-3/transaction_commit.gif)

## Summary

`CoreAnimation` renders our app view hierarchy at each run loop pass and not at each view hierarchy modification. To do so, it observes each modification made to each view and begins a `CATransaction` if one does not already exist. As an observer of the run loop, `CoreAnimation` finally commits this transaction just before the thread is put to sleep. The interest is to gather all the modifications made since the wake-up and not refreshing the screen excessively. Just before making the rendering, `CoreAnimation` makes sure each view is laid out by triggering a layout phase.

## The events of a UIViewController

An interest in discovering the run loop and its interaction with `CoreAnimation` is to have a new point of view on the events that trigger the code of our applications. Apple has taken care to expose us a high level and accessible API: `viewWillAppear`, `viewDidAppear`, `viewWillLayoutSubviews` etc. We can now try to give them a new meaning.

### viewWillAppear

{% highlight text %}

#0: Layout PresentedViewController.viewWillAppear(animated=true, self=0x00007fafb6602be0) at ViewController.swift:51
 #1: Layout @objc PresentedViewController.viewWillAppear(_:) at ViewController.swift:0 
... 
#6: UIKit _cleanUpAfterCAFlushAndRunDeferredBlocks + 388 
#7: UIKit _afterCACommitHandler + 137
 #8: CoreFoundation __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__ + 23

{% endhighlight %}

When presenting a `UIViewController`, `viewWillAppear` is the first method called by UIKit. According to this breakpoint, it is called in the same stack as the commit of the implicit transaction and therefore as a layout phase. The `UIViewController` view is about to be sent to the render server. However, the layout has not yet taken place: the `layoutSubviews` methods of our views have not yet been called.

### viewWillLayoutSubviews and viewDidLayoutSubviews

{% highlight text %}

#0: Layout PresentedViewController.viewWillLayoutSubviews(self=0x00007fafb6602be0) at ViewController.swift:59
 #1: Layout @objc PresentedViewController.viewWillLayoutSubviews() at ViewController.swift:0 
...
 #6: QuartzCore CA::Transaction::commit() + 568 
#7: UIKit _afterCACommitHandler + 272
 #8: CoreFoundation __CFRUNLOOP_IS_CALLING_OUT_TO_AN_OBSERVER_CALLBACK_FUNCTION__ + 23 ...

{% endhighlight %}

`viewWillLayoutSubviews` announces the start of layout of the view controller root view. It is called in the same `viewWillAppear` stack but this time the commit has taken place (see #6). Successive calls to the `layoutSubviews` methods of the `UIViewController` views are about to start. `viewDidLayoutSubviews` is called just after the execution of the `layoutSubviews` of the root view of the view controller.

### viewDidAppear

{% highlight text %}

#0: Layout PresentedViewController.viewDidAppear(animated=true, self=0x00007fafb6602be0) at ViewController.swift:55 
#1: Layout @objc PresentedViewController.viewDidAppear(_:) at ViewController.swift:0
 ...
 #17: CoreFoundation __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__ + 9 ...

{% endhighlight %}

`viewDidAppear` is called a few seconds later - the time for the presentation animation to finish. It appears in the stack of a block of code sent to the main thread. This block was probably defined using the `setCompletionBlock` method at the beginning of the `UIViewController` presentation, so `viewDidAppear` is called once all the presentation animations are finished.

## Conclusion

In this journey, we discovered the main components of the iOS rendering process: a transaction, a run loop and an observer pattern.

However, we cannot claim to have fully answered the question. We went a bit beyong the intelligible world of the Apple's documentation but the reality escapes us. From one version of iOS to another, the elements described above may change and many of the statements are actually simple observations.

We also made a few shortcuts. If you try to put the C++ breakpoint given above, you may encounter commits made by `CoreAnimation` in blocks other than those triggered by an observer. In particular, if you break too long on a breakpoint, `CoreAnimation` seems to trigger a layout phase as soon as the current stack ends.

We have nevertheless managed to break through the layer, represented by UIKit, which hides the heart of an iOS application. The events exposed by UIKit allow us to always execute our code in respect of the infinite loops of the run loop that drives the screen refreshs. We highlighted the application's life cycle and the importance of scrupulously respecting the UIKit events.
