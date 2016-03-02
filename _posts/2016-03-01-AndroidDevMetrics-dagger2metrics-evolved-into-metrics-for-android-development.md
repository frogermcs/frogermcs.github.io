---
layout: post
title: AndroidDevMetrics - dagger2metrics evolved into metrics for Android development
tags: [AndroidDevMetrics, performance, metrics, activity lifecycle]
---

[Dagger2Metrics] was a good start to help us look for performance issues in Android apps which we develop. It shows us how much time is needed to initialize particular objects (and their dependencies) in graph, in Dagger 2. But there are much more things to measure than just an object initializations. And it's the reason why I've decided to make this tool a part of something bigger. Let me introduce an evolution of dagger2metrics:

# AndroidDevMetrics
Performance metrics library for Android development. 

The problem with performance is that it often decreases slowly so in day-by-day development it's hard to notice that our app (or Activity or any other view) launches 50ms longer. And another 150ms longer, and another 100ms...

With **AndroidDevMetrics** you will be able to see how performant are the most common operations like object initialization (in Dagger 2 graph), or Activity lifecycle methods (`onCreate()`, `onStart()`, `onResume()`).

It won't show you exact reason of performance issues or bottlenecks (yet!) but it can point out where you should start looking first. 

AndroidDevMetrics currently includes:

* Activity lifecycle metrics - metrics for lifecycle methods execution (`onCreate()`, `onStart()`, `onResume()`)
* Frame rate drops - metrics for fps drops for each of screens (activity)
* Dagger 2 metrics - metrics for objects initialization in Dagger 2 

![screenshot1.png](https://raw.githubusercontent.com/frogermcs/androiddevmetrics/master/art/activities_metrics.png)

![screenshot.png](https://raw.githubusercontent.com/frogermcs/androiddevmetrics/master/art/dagger2_metrics.png)

## Getting started

Script below shows how to enable all available metrics.

In your `build.gradle`:

{% gist frogermcs/56c16dc3bc8482316119 build.gradle %}

In your `Application` class:

{% gist frogermcs/56c16dc3bc8482316119 ExampleApplication.java %}

## How does it work?

Detailed description how it works under the hood can be found on wiki pages:

* [Activity lifecycle and frame drops metrics](https://github.com/frogermcs/AndroidDevMetrics/wiki/Activity-lifecycle-metrics)
* [Dagger 2 metrics](https://github.com/frogermcs/AndroidDevMetrics/wiki/Dagger-2-metrics)

## I found performance issue, what should I do next?

There is no silver bullet for performance issues but here are a couple steps which can help you with potential bugs hunting.

If measured time of object initialization or method execution looks suspicious you should definitely give a try to [TraceView](http://developer.android.com/tools/debugging/debugging-tracing.html). This tool logs method execution over time and shows execution data, per-thread timelines, and call stacks. Practical example of TraceView usage can be found in this blog post: [Measuring Dagger 2 graph creation performance](http://frogermcs.github.io/dagger-graph-creation-performance/).

---

If it seems that layout or view can be a reason of performance issue you should start with those links from official Android documentation:

* [Improving Layout Performance](http://developer.android.com/training/improving-layouts/index.html)
* [Optimizing Layout Hierarchies](http://developer.android.com/training/improving-layouts/optimizing-layout.html)

--- 

Finally, if you want to understand where most of performance issues come from, here *is a collection of videos focused entirely on helping developers write faster, more performant Android Applications.*

* [Android Performance Patterns](https://www.youtube.com/playlist?list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE)

## Example app

You can check [GithubClient](https://github.com/frogermcs/githubclient) - example Android app which shows how to use Dagger 2. Most recent version uses **AndroidDevMetrics** for measuring performance.

## License

    Copyright 2016 Miroslaw Stanek

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.


## Source code

Source code of Dagger2Metrics is available on Github [project site].

### Author 

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com
[Dagger 2 - graph creation performance]:http://frogermcs.github.io/dagger-graph-creation-performance/
[Traceview]:http://tools.android.com/tips/traceview
[Dagger2Metrics]:https://github.com/frogermcs/dagger2metrics
[project site]:https://github.com/frogermcs/androiddevmetrics
