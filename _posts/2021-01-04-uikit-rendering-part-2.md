---
layout: post
title: "UIKit rendering - CATransaction"
author:
  name: "Gaétan Zanella"
  link: "https://twitter.com/gaetanzanella"
tags: iOS
---

On the [previous post](/2021-01-04/uikit-rendering-part-1), we discovered a mysterious `CATransaction.commit` at the bottom of our interrupted layout phase stack. Let's find out its role in the layout process.

### Experimentation


The `CATransaction` is little used explicitly. We find few usages of it in our projects at Fabernovel:

{% highlight swift %}

CATransaction.begin()
 collectionView.reloadData()
CATransaction.setCompletionBlock { ... }
 CATransaction.commit()

{% endhighlight %}

Apple's [documentation](https://developer.apple.com/documentation/quartzcore/catransaction) tells us that the `CATransaction` class has two main methods: `begin` and `commit`. To understand how they work, let's define a method that can block the main thread for a given time:

{% highlight swift %}

func blockThreadDuringThreeSeconds() {
     let date = Date()
     while -date.timeIntervalSinceNow < 3 {} 
}

{% endhighlight %}

#### Explicit transactions

Usually changes made to a view tree are not effective immediately. They are simply programmed. Indeed, if we block the execution of our code, just after having applied a modification to a tree, it is not immediately visible:

{% highlight swift %}

sampleView.backgroundColor = .red
 blockThreadDuringThreeSeconds()

{% endhighlight %}

![Explicit transactions](/assets/uikit-rendering-part-2/experiment_1.gif)

On the other hand, if we surround our modification by the begin and commit methods of CATransaction, the screen refreshes immediately:

{% highlight swift %}

CATransaction.begin()
 sampleView.backgroundColor = .red
 CATransaction.commit()
 blockThreadDuringThreeSeconds()

{% endhighlight %}

![Explicit transactions](/assets/uikit-rendering-part-2/experiment_2.gif)

Therefore a transaction transfers a view hierarchy to the render server that updates the screen.

#### Implicit transactions

Apple states it in its documentation:
> Any modification made to the view tree must be part of a transaction

In our first example when we changed the color of our sample view, where was the transaction?

Well, `CoreAnimation` started one for us. It watches each change that requires a screen refresh (probably with well-placed KVO) and starts a transaction if none is present. We define this transaction as "implicit".

> Implicit transactions are created automatically when the layer tree is modified by a thread without an active transaction

To reveal it, we can use a feature of the transactions: they are nestable. When one transaction is nested into another, the commit of the "child" transaction is effective only at the commit of the parent.

{% highlight swift %}

CATransaction.begin() 
sampleView.backgroundColor = .red
 CATransaction.begin() 
sampleView.backgroundColor = .green 
CATransaction.commit() // The screen does not refresh
 blockThreadDuringThreeSeconds() 
CATransaction.commit()// The screen refreshs

{% endhighlight %}

![Implicit transactions](/assets/uikit-rendering-part-2/experiment_3.gif)

Therefore, if we make two modifications: one, isolated, and the other, encapsulated in an explicit transaction, we find ourselves in the same case as our first example: the screen displays our modification only after three seconds.

{% highlight swift %}

sampleView.backgroundColor = .red
CATransaction.begin()
sampleView.backgroundColor = .green
CATransaction.commit() // The screen does not refresh
blockThreadDuringThreeSeconds() // The screen refreshs

{% endhighlight %}

![Implicit transactions](/assets/uikit-rendering-part-2/experiment_4.gif)

The culprit? The implicit transaction. Our transaction is encapsulated by the implicit transaction created by `CoreAnimation` when `sampleView.backgroundColor = .red` is executed. The modifications will be sent to render server only once the implicit transaction commits them. Our view becomes green only after three seconds.

### CATransaction and layout

We can easily verify that a transaction triggers a layout phase at commit time by modifying our `buttonAction` method:

{% highlight swift %}

func buttonAction() {
    CATransaction.begin()
    sampleViewHeightConstraint.constant = 20
    CATransaction.commit()
    sampleView.frame.height == 20 // true
}

{% endhighlight %}

This action method lays out the current view hierarchy. It is, in a way, similar to a `layoutIfNeeded` :

{% highlight text %}

#0: 0x000000010c584758 Layout View.layoutSubviews(self=0x00007faf8b70e980) at ViewController.swift:15
#1: 0x000000010c584a84 Layout @objc View.layoutSubviews() at ViewController.swift:0
...
#6: 0x000000010d7be79c QuartzCore CA::Transaction::commit() + 568
#7: 0x000000010c585147 Layout ViewController.buttonAction(sender=Any @ 0x00007ffee36790b8, self=0x00007faf8b709d80) at ViewController.swift:3
#8: 0x000000010c5851ac Layout @objc ViewController.buttonAction(_:) at ViewController.swift:0
...
#18: 0x0000000110cccbb1 CoreFoundation __CFRUNLOOP_IS_CALLING_OUT_TO_A_SOURCE0_PERFORM_FUNCTION__ + 17

{% endhighlight %}

When you think about it, it's quite logical that the commit of a transaction is followed by a layout phase. The role of a transaction is to send the current state of the view tree to the render server. It seems that it lays out the view tree before sending it.

So we come to a first element of answer: looking for the next layout phase is looking for the commit of the current implicit transaction.

Indeed, in the stack of our initial code, we do find a call for a transaction commit:

{% highlight c++ %}

CA::Transaction::commit()

{% endhighlight %}

And it is an implicit transaction! The modification of the sample view constraint is the origin of it, we did nothing explicitly.

### CATransaction and completionBlock

It is possible to be notified once a transaction completes. This is particularly interesting when a transaction encapsulates animations. The block is only executed once all the programmed animations are completed.

Here is an example from one of our project at Fabernovel:

{% highlight swift %}

@IBAction func buttonAction(_ sender: Any) {
    CATransaction.begin()
 collectionView.reloadData()
    CATransaction.setCompletionBlock { ... }
 CATransaction.commit()
}

{% endhighlight %}

{% highlight text %}

#0: 0x0000000108f918e4 Layout closure #1 in ViewController.buttonAction(_:) at ViewController.swift:30
#1: 0x0000000108f919bd Layout thunk for @escaping @callee_guaranteed () -> () at ViewController.swift:0
...
#4: 0x000000010e8a78cf libdispatch.dylib _dispatch_main_queue_callback_4CF + 628
#5: 0x000000010d6fac99 CoreFoundation __CFRUNLOOP_IS_SERVICING_THE_MAIN_DISPATCH_QUEUE__ + 9

{% endhighlight %}

Here, without any animation, our block is executed on the main queue right after the commit method so once the table view has finished reloading.

---

In the [next post](/2021-01-04/uikit-rendering-part-3), we will tackle the run loops and what is its link to the transaction commit of our interrupted layout phase stack.
