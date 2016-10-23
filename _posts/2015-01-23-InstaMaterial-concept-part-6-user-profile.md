---
layout: post
title: InstaMaterial concept (part 6) - User profile
tags: [android, material design, ui, animations, activity transition, ViewPropertyAnimator, circural reveal, circural image]
---

This post is a part of a series of posts showing Android implementation of [INSTAGRAM with Material Design concept]. Today we'll create user profile presented between the 29th and 33rd second of the concept video.

<!-- more -->

This is the final effect described in today's post (for both Android Lollipop and pre-21 versions):

<iframe width="420" height="315" src="//www.youtube.com/embed/EmQM0B6QPac" frameborder="0" allowfullscreen></iframe>

<iframe width="420" height="315" src="//www.youtube.com/embed/NGyoszveMw4" frameborder="0" allowfullscreen></iframe>

# Introdution

Honestly if you read carefully my previous posts and tried to implement described effects and views, today you won't learn anything new. Even if the final effect looks very complex almost every used technique or tool were described previously. Actually it's a good news - the number of different solutions in Android platform is finite. But the way in which you use them is limited only by your imagination. 😄

# Preparation

As always let's start from adding new elements. We have to create new `UserProfileActivity` with Toolbar, RecyclerView and FloatingActionButton. Because of custom transition it should use the same style which we used in `CommentsActivity`. Also we have to add onClick listener to profile photo in `FeedAdapter`. And by the way I did small refactoring by creating BaseActivity. Here you have list of commits:

* [Add UserProfileActivity]
* [FeedAdapter update]
* [code refactoring]

# UserProfileActivity circural reveal transition 

![Circural reveal](/images/7/circural_reveal.gif "Circural reveal")

First of all let's start from implementation of transition effect for user profile. As you probably noticed, Material Design introduced fancy circular animation mainly for the reveal effect. Unfortunately for us, the developers, it looks nice in design guidelines, but still we don't have any good tool for implementing this in the real application. Let's see the possibilities which we have right now:

* **ViewAnimationUtils.createCircularReveal()**  
This is the simplest way to achieve circural reveal animation. The main problem is that it works only on Lollipop and for now there is no compatibility library for previous Android versions. Like `Ripple` effect it uses Render thread which is not available on older system versions.

* **CircularReveal library**  
There is an open source [project] which provides `ViewAnimationUtils.createCircularReveal()` to previous Android versions (>= 2.3). It's still fresh but looks very promising. It provides two ViewGroups: `RevealFrameLayout` and `RevealLinearLayout` which can by animated.

* **Maybe we don't need the circle?**  
Well, sometimes we don't have too much time to look for the ideal solution. If only we want to fast prototype reveal effect maybe square animation could be ok? We can achieve this by ViewPropertyAnimator in 30 seconds. How? Just animate scale X and Y and set starting point by setPivotX/Y like below:  
![Scale animation](/images/7/scale_animation.gif "Scale animation")

* **Use shaders**  
My favourite method, especially when we have more complex view to animate. Shaders can be a little bit harder to understand at the beginning, but it is the one of the most efective way to animate. On **Romain's Guy blog** you can find [recipe] for reveal animation made with shaders.

* **Do it in your own way**  
If none of mentioned methods work for you, just think about requirements and try to implement it in your own way. Sometimes the simplest solution can fit your expectations, and like in our example there is no difference in performance for the final result. Did I mentioned that in our app I picked this method? 😄

## Custom RevealBackgroundView
We will implement our custom view which can perform circular reveal animation. The requirements are simple:

* Reveal should be circular
* Reveal animation should start from picked point
* We want to handle animation states (not started, started, finished)
* Also we can pick finished state by hand

In our custom view we use `ObjectAnimator` described in [previous post]. It animates circle size by setting currentRadius value. For drawing circle we override `onDraw()` method which draws circle with given radius on given position or fullscreen rectangle if view's state is finished. Here is the source code for `RevealBackgroundView`:

{% gist frogermcs/178fee4ca5e368b732ec RevealBackgroundView.java %}

Now let's add starting method for user profile in `MainActivity`:

{% gist frogermcs/178fee4ca5e368b732ec MainActivity_startProfile.java %}

Now, when we have starting position, just use it in `UserProfileActivity` and animate the background in the right moment (thanks to onPreDrawListener). The final source code for `UserProfileActivity` should look like below:

{% gist frogermcs/178fee4ca5e368b732ec UserProfileActivity.java %}

What we got here?  If Activity is starting, reveal animation runs. Otherwise, if we restoring Activity, background is set to finished state by `vRevealBackground.setToFinishedFrame()`. RecyclerView with user profile is hidden while reveal animation is running, and is showed when its state changes to STATE_FINISHED (this is done in `onStateChange()` method).  `userPhotosAdapter.setLockedAnimations(true)` disables profile animations when user starts scrolling or Activity is restoring.

# User profile layout
User profile showed in [concept video] can be splited to 3 main parts:

* User profile header with all user data and statistics,
* User profile options
* User photos

First of all let's prepare layout for each of them. Actually it's nothing special. Here you have screenshots. If you want, just try to reproduce this on your own, or check [this commit] for the source code.

### User profile header

![User profile header](/images/7/profile_header.png "User profile header")

### User profile options

![User profile options](/images/7/profile_options.png "User profile options")

### User photo

![User photo](/images/7/profile_photo.png "User photo")

Next we can start implementation of `UserProfileAdapter`. User profile will be presented byl `RecyclerView`. First two elements (header and options) should take whole width. Photos should be presented below them in rows with maximum 3 elements inside. That's why we used `StaggeredGridLayoutManager` in our Activity. It can define how much elements should contain one row (for vertical scroll) or one column (in horizontal scroll). Also thanks to this manager we can define how much place should take every RecyclerView item.

Let's start to build our adapter. First of all we have to define items types:

{% gist frogermcs/178fee4ca5e368b732ec UserProfileAdapter_types.java %}

Every type should have his own `ViewHolder`:

{% gist frogermcs/178fee4ca5e368b732ec UserProfileAdapter_holders.java %}

We use them in `onCreateViewHolder()` method, where we have to inflate our views and setup them for our layout manager:

{% gist frogermcs/178fee4ca5e368b732ec UserProfileAdapter_createholders.java %}

In lines 6, 12 and 20 by `setFullSpan()` we can point out which views should take whole row width.

The last thing we should do (before the animations) is binding ViewHolders:

{% gist frogermcs/178fee4ca5e368b732ec UserProfileAdapter_bindholders.java %}

We won't focus ond `bind...` methods, but one thing is worth mentioning. For photo loading we use [Picasso] library. It has one very powerful functionality. We can transform image in the background thread before it will be put in ImageView. 

## Circular user photo

As you probably noticed, user profile photo has circular shaper with white outline. This effect can by simply achieved by image transformation in Picasso, with a little usage of shaders. All we have to do is to implement of `Transformation` interface which modifies given bitmap. For circular shape we can use shaders (check another [Romain Guy's recipe] for simple example of shader usage).  
Our code is pretty simple:

{% gist frogermcs/178fee4ca5e368b732ec CircleTransformation.java %}

Our `CircularTransformation` draws circle bitmap with the white outline on top of it.

Full source code of `UserProfileAdapter` described up to this moment is availabe [here]. And this is the final effect of current implementation:

![User profile](/images/7/user_profile.png "User profile")

# User profile intro animation

![User profile animation](/images/7/profile_animation.gif "User profile animation")

And the final thing - intro animation. On the [concept video] it looks complex but I think that it is the simplest part od today's post. Actually all we have to do is to animate each view with `ViewPropertyAnimator` in the right moment, with the right timing and effects.

Profile header and options animations should be started immediately, so the best moment is a callback from PreDrawListener. Animations are pretty simple - they just change translation or alpha in views. Everything is done in about 20 lines of code:

{% gist frogermcs/178fee4ca5e368b732ec UserProfileAdapter_headeranimations.java %}

As I said - the most important things are timings and right moments of start. I this case It is achieved by trial and error method. 😄

A little bit different are photos animations. We are not sure how much time it will take to load them from the Internet, so the best place to run these animations is the moment when the photo is displayed in. Fortunately Picasso has a simple callback with `onSuccess()` and `onError()` methods which will be used for firing the animations.  
Also we have to consider the second option - when the photo is loaded faster (i.e. from cache), before the profile animation will be finish. In this case all we have to do is a simple logic which calculates proper delay value.  
The final code can look like below:

{% gist frogermcs/178fee4ca5e368b732ec UserProfileAdapter_photoanimation.java %}

And the full commit with all required changes is [available here].

That's all for today. We've just finished user profile with opening transition and all animations. thanks for reading! 😄

## Source code
Full source code of described project is available on Github [repository].

*Author: [Miroslaw Stanek]*

[INSTAGRAM with Material Design concept]:https://www.youtube.com/watch?v=ojwdmgmdR_Q
[concept video]:https://www.youtube.com/watch?v=ojwdmgmdR_Q
[Add UserProfileActivity]:https://github.com/frogermcs/InstaMaterial/commit/97dc05bc340709f0995052ed98a4693cb634f96d
[FeedAdapter update]:https://github.com/frogermcs/InstaMaterial/commit/5942df826b14355ddc6fe7dc6dfbce98a9eb02e4
[code refactoring]:https://github.com/frogermcs/InstaMaterial/commit/aa89624a8404532f0fd71993914a7bb0ade2316b
[project]:https://github.com/ozodrukh/CircularReveal
[previous post]:http://frogermcs.github.io/InstaMaterial-concept-part-5-like_action_effects/
[this commit]:https://github.com/frogermcs/InstaMaterial/commit/7dc54888b37ae29651b9f242ca15c62ed6ccdb28
[Picasso]:square.github.io/picasso/
[here]:https://github.com/frogermcs/InstaMaterial/commit/fb2d3689d2c6b6f28e4ad05908a96d4661872604
[available here]:https://github.com/frogermcs/InstaMaterial/commit/d352d689e6ba1666b1042c52434638b0bf7654d2
[Romain Guy's recipe]:http://www.curious-creature.com/2012/12/11/android-recipe-1-image-with-rounded-corners/
[recipe]:http://www.curious-creature.com/2012/12/13/android-recipe-2-fun-with-shaders/
[repository]:https://github.com/frogermcs/InstaMaterial
[Miroslaw Stanek]:http://about.me/froger_mcs