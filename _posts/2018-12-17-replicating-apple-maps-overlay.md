---
layout: post
title: "Replicating Apple Maps overlay"
author:
  name: "Gaétan Zanella"
  link: "https://twitter.com/gaetanzanella"
tags: iOS
---

We find overlays in many Apple native apps like Shortcuts, Maps andStocks. How to replicate it ?

<p align="center">
    <img src="/assets/replicating-apple-maps-overlay/apple-maps.gif" width="222">
</p>

The overlays presented in the different Apple applications do not look exactly the same. In Stocks, the overlay looks a bit viscous for instance. I focuses my research on the overlay displayed in the Shortcuts app. It helped me to reproduce some edge cases.

Let’s analyse it :

- There is a smooth transition between the scroll and the translation. This is the most important point. The user can continuously scroll or drag the overlay in a single gesture.

<p align="center">
    <img src="/assets/replicating-apple-maps-overlay/scroll.gif" width="222">
</p>

- The user can drag the overlay without scrolling using other UI elements like a search bar.

- The overlay can skip some notches based on the velocity.

- The overlay can be displayed on compact horizontal but transformed into a simple detail view controller in a UISplitViewController on regular environment.

- The overlay has a rubber band effect only when the user drags the overlay using an UI element that is not the scroll view.

<p align="center">
    <img src="/assets/replicating-apple-maps-overlay/rubberBand.gif" width="222">
</p>

- The overlay goes down only if the scroll view can not scroll down. This is the thickest one. The user can alternatively scroll down & drag the overlay up in a single gesture until it reaches the highest notch.

<p align="center">
    <img src="/assets/replicating-apple-maps-overlay/scrollToTranslation.gif" width="222">
</p>

- If the user drags the overlay without using the scroll view, the content offset of the scroll view remains the same.

<p align="center">
    <img src="/assets/replicating-apple-maps-overlay/keedOffsetOnTranslation.gif" width="222">
</p>

There are currently several libraries available to mimic it :
- [Pulley](https://github.com/52inc/Pulley)
- [FloatingPanel](https://github.com/SCENEE/FloatingPanel)

I tried all of them and they all miss something on the motion. So I decided to replicate it myself using a different approach. Some parts of the motion are customizable :

- The number of notches. Apple always uses 3 notches but why not providing a way to change it ?
- The height of the notches.
- The animation when the overlay finishes reaching a notch.
- The translation function. We could imagine scenarios where when the user moves its finger, the overlay goes twice faster. This could be used to notify the user that the overlay has reached a boundary for instance. The user’s finger keeps moving but the overlay moves decreasingly.

Give it a try ! And please let me known if you see anything wrong. For the details of the implementation, see the related Stack Overflow question.
