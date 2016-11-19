---
layout: post
title: FrameMetrics — realtime app smoothness tracking
tags: [android, FrameMetrics, performance, metrics]
---

A couple months ago, when Android Nougat was announced, among new features like Multi-window, enhanced notifications, VR support, 72 new emojis 👍 and others, there was a new addition to monitoring tools: [Frame Metrics API](https://developer.android.com/about/versions/nougat/android-7.0.html#framemetrics_api).

On older Android versions when we would like to see graphic performance metrics we had to get them via adb shell, by calling:

**$ adb shell dumpsys gfxinfo com.frogermcs.framemetrics**

As a result we get:

{% gist frogermcs/56b49ee97b50d26a5f4cfd5ad6d6461c gistfile1.txt %}

These statistics are generated for last 120 frames for given application package (*hint: this is not always last 2 seconds - frames are drawn only when Android is requested to do this e.g. when layout is changed or animated*).

While this data can give us a lot of information about UI performance, accessing it from console isn’t the most convenient way, especially when we would like to use it during automatic app testing. Of course it isn’t impossible, and great example how it can be done is described in [Automated Performance Testing Codelab](https://codelabs.developers.google.com/codelabs/android-perf-testing/index.html?index=..%2F..%2Findex#0) (it’s implemented as a MonkeyRunner script).

# FrameMetrics

Starting from Android SDK 24, we can have access to these informations directly from application code. There is no limit to the past 120 frames, because frame timing info for current app window is captured in realtime. Having direct access to this data means also that we are able to measure production app to know how it works on users’ devices.

FrameMetrics API [documentation](https://developer.android.com/reference/android/view/FrameMetrics.html) is pretty clear about what can be measured (chronologic order):

- [UNKNOWN_DELAY_DURATION](https://developer.android.com/reference/android/view/FrameMetrics.html#UNKNOWN_DELAY_DURATION)

- [INPUT_HANDLING_DURATION](https://developer.android.com/reference/android/view/FrameMetrics.html#INPUT_HANDLING_DURATION)

- [ANIMATION_DURATION](https://developer.android.com/reference/android/view/FrameMetrics.html#ANIMATION_DURATION)

- [LAYOUT_MEASURE_DURATION](https://developer.android.com/reference/android/view/FrameMetrics.html#LAYOUT_MEASURE_DURATION)

- [DRAW_DURATION](https://developer.android.com/reference/android/view/FrameMetrics.html#DRAW_DURATION)

- [SYNC_DURATION](https://developer.android.com/reference/android/view/FrameMetrics.html#SYNC_DURATION)

- [COMMAND_ISSUE_DURATION](https://developer.android.com/reference/android/view/FrameMetrics.html#COMMAND_ISSUE_DURATION)

- [SWAP_BUFFERS_DURATION](https://developer.android.com/reference/android/view/FrameMetrics.html#SWAP_BUFFERS_DURATION)

- [TOTAL_DURATION](https://developer.android.com/reference/android/view/FrameMetrics.html#TOTAL_DURATION)

Metrics names are mostly self explaining but if this list looks not complete to you, take a look at [FrameMetrics sources](https://github.com/android/platform_frameworks_base/blob/4b1a8f46d6ec55796bf77fd8921a5a242a219278/core/java/android/view/FrameMetrics.java) to see how each metric is calculated:

{% gist frogermcs/b88e189ea01d83c3c91646f76897b5c7 FrameMetrics.java %}

There are also great videos where **Colt McAnlis** explains among the others what VSYNC is or how layout is transformed to GPU commands (`FrameMetrics.COMMAND_ISSUE_DURATION`). So they also can shed some light on FrameMetrics fields:

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Some FrameMetrics fields can be demystified by <a href="https://twitter.com/duhroach">@duhroach</a> videos: Rendering Performance on <a href="https://twitter.com/hashtag/AndroidDev?src=hash">#AndroidDev</a> <a href="https://t.co/LG3BUDK4Fs">https://t.co/LG3BUDK4Fs</a> <a href="https://twitter.com/hashtag/PerfMatters?src=hash">#PerfMatters</a></p>&mdash; Miroslaw Stanek (@froger_mcs) <a href="https://twitter.com/froger_mcs/status/797912470828548096">November 13, 2016</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

# ActivityFrameMetrics

To make FrameMetrics even easier to use I created simple library called **ActivityFrameMetrics**. It is small 1-class library which prints janky frames warnings to Logcat:

![Logcat](/images/29/logcat.png "Logcat")

Installation is pretty simple — you can copy [ActivityFrameMetrics.java](https://github.com/frogermcs/ActivityFrameMetrics/blob/master/activityframemetrics/src/main/java/com/frogermcs/activityframemetrics/ActivityFrameMetrics.java) class to your project or use gradle dependency:


{% gist frogermcs/41f69bffff535b33eaca9a88e0c831e2 ActivityFrameMetrics.gradle %}

To use it, just register `ActivityFrameMetrics` as an ActivityLifecycleCallback in your `Application` class:

{% gist frogermcs/0429206107879fdb23237cdcb7397dae MyApplication.java %}

By default library prints Logcat warning for every frame which rendering took more than 17ms, and error for more than 34ms.

Default values can be changed with `ActivityFrameMetrics.Builder` params:

{% gist frogermcs/db346227c17676b5e0ee09b11bc2b4ee ActivityFrameMetricsBuilder.java %}

Repo for ActivityFrameMetrics library can be found on [Github](https://github.com/frogermcs/ActivityFrameMetrics).

# Digging deeper into FrameMetrics

If it’s still not enough and you are still curious how FrameMetrics works under the hood here are a couple hints where you could start searching.

First of all FrameMetrics is just a simple data container written in java but nothing really is measured in it. Instead this data comes as a notification from C++ code which is responsible for view rendering. Good starting point could be [ThreadedRenderer](https://github.com/android/platform_frameworks_base/blob/4b1a8f46d6ec55796bf77fd8921a5a242a219278/core/java/android/view/ThreadedRenderer.java) class which proxies rendering to render thread.
If you explore C++ code a bit more, at the end probably you’ll find [JankTracker](https://github.com/android/platform_frameworks_base/blob/4b1a8f46d6ec55796bf77fd8921a5a242a219278/libs/hwui/JankTracker.cpp) class which is responsible for filling up FrameMetrics data (and you will realise that this is exactly the same data which is presented in **dumpsys gfxinfo**) and [FrameInfo](https://github.com/android/platform_frameworks_base/blob/4b1a8f46d6ec55796bf77fd8921a5a242a219278/libs/hwui/FrameInfo.h) class which is C++ data container which is in sync with FrameMetrics.java.

What else you can find there is only up to you. **Just never stop searching and always be curious**. 

Thanks for reading!

### Author 

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com
