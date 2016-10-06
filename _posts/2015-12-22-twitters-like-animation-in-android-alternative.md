---
layout: post
title: Twitter's like animation in Android - alternative
tags: [animations, Twitter heart, ObjectAnimator]
---

Some time ago Twitter presented Hearts - replacement for star icons, with modern delightful animation of their state change.

<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr">You can say a lot with a heart. Introducing a new way to show how you feel on Twitter: <a href="https://t.co/WKBEmORXNW">https://t.co/WKBEmORXNW</a> <a href="https://t.co/G4ZGe0rDTP">pic.twitter.com/G4ZGe0rDTP</a></p>&mdash; Twitter (@twitter) <a href="https://twitter.com/twitter/status/661558661131558915">November 3, 2015</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

While heart symbol is more universal and expressive, today we'll try to reproduce new animation for alternative reality, using old, star icon. Effects of our work will look like this (and it's bit faster than animated gif):

![Button animation](/images/22/button_anim.gif "Button animation")

<iframe width="420" height="315" src="https://www.youtube.com/embed/EdZjYTbRNuA" frameborder="0" allowfullscreen></iframe>

While the easiest way for implementing this (and original heart) animation would be to use [Frame Animations], we'll try to implement it with more flexible solution - by drawing it by hand and animating with ObjectAnimator. This post will be just a quick overview, without deep technical details.

# Implementation

We'll create new View named `LikeButtonView` which will be built on top of FrameLayout which hosts three child views - CircleView showing circle below star icon, ImageView (with our star) and DotsView presenting dots floating around our button.

## CircleView

![Circle animation](/images/22/circle_anim.gif "Circle animation")

This view is responsible for drawing big circle below star icon. It could be implemented easier (via xml and `<shape android:shape="oval">`) but in this case we should care about background color below our button.

Our implementation draws circle(s) on canvas:

{% gist frogermcs/f900415dbbee650ef811 ondraw.java %}

Each frame starts from clearing whole canvas by drawing color with CLEAR mode. Then view draws inner and outer circle based on given progress (separately for both of them).

Inner circle uses mask paint defined in this way:

{% gist frogermcs/f900415dbbee650ef811 maskPaint.java %}

It means that our circle will create transparent hole inside our outer circle. 

Our View uses `tempCanvas` with `tempBitmap` defined here:

{% gist frogermcs/f900415dbbee650ef811 onSizeChanged.java %}

We need to do this for true transparency. Otherwise our inner circle would show window color instead.

For skilled eye there is one more thing - our outer circle changes its color based on current progress. It's done by [ArgbEvaluator] class which transforms two colors based on given fraction:

{% gist frogermcs/f900415dbbee650ef811 updateCircleColor.java %}

The rest of `CircleView` code is just an implementation. Full source code can be found here: [CircleView].

## DotsView

![Dots animation](/images/22/dots_anim.gif "Dots animation")

This view will draw dots floating around our star icon. Same as `CircleView` it will use `onDraw()` to do this:

{% gist frogermcs/f900415dbbee650ef811 onDraw2.java %}

Dots are drawn based on `currentProgress` backed by math and honestly, it's hard to point something interesting here (from Android SDK perspective). Instead, here are a couple math-related things:

- Dots are arranged on invisible circle - their position is determined by:  

{% gist frogermcs/f900415dbbee650ef811 math.java %}

what means: set dot on every `OUTER_DOTS_POSITION_ANGLE` (51 degrees).

- Each dot has its own color animation:

{% gist frogermcs/f900415dbbee650ef811 updateDotsPaints.java %}

It means that dot color is animated between 3 values in ranges: [0, 0.5) and [0.5, 1]. One more time we use `ArgbEvaluator` to make it smooth.

The rest is pretty straightforward. Full source code of this class is available here: [DotsView]

## LikeButtonView

Our final views group is composed from `CircleView`, `ImageView` and `DotsView`. 

{% gist frogermcs/f900415dbbee650ef811 likebutton.xml %}

We used [Merge] tag which helps to eliminate redundant view groups. `LikeButtonView` itself is `FrameLayout` so there is no need to have it twice.

Our final view animation is composed from a few smaller ones, played together by `AnimatorSet`:

{% gist frogermcs/f900415dbbee650ef811 onClick.java %}

And it's all about proper timings and interpolators. 

---

![Touch animation](/images/22/touch_anim.gif "Touch animation")

Our `LikeButtonView` also reacts on touch event (with scale animation):

{% gist frogermcs/f900415dbbee650ef811 onTouchEvent.java %}

And... That's all. 😃 As you can see, there is no magic here, but the final effect can be very nice. So what now? Let's make our apps even more delighful. 

---

## Source code

Ful source code of described project is available on Github [repository].

### Author 

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com
[repository]:https://github.com/frogermcs/LikeAnimation/
[Frame Animations]:https://www.bignerdranch.com/blog/frame-animations-in-android/
[ArgbEvaluator]:http://developer.android.com/reference/android/animation/ArgbEvaluator.html
[CircleView]:https://github.com/frogermcs/LikeAnimation/blob/master/app/src/main/java/frogermcs/io/likeanimation/CircleView.java
[DotsView]:https://github.com/frogermcs/LikeAnimation/blob/master/app/src/main/java/frogermcs/io/likeanimation/DotsView.java
[Merge]:http://developer.android.com/training/improving-layouts/reusing-layouts.html#Merge
