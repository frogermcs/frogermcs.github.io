---
layout: post
title: InstaMaterial concept (part 3) - Feed and comments buttons
tags: [android, material design, ui, animations, layout, ripple, button]
---

This post is part of a series of posts showing Android implementation of [INSTAGRAM with Material Design concept]. Today we'll take care about some details which we skipped previously. It means that we are still on 9 to 13 seconds time period from the concept video.

<!-- more -->

This is the final effect described in today's post (for both Android Lollipop and pre-21 versions):

<iframe width="420" height="315" src="//www.youtube.com/embed/GWKiN3la_CQ" frameborder="0" allowfullscreen></iframe>

<iframe width="420" height="315" src="//www.youtube.com/embed/6ill09PZOFg" frameborder="0" allowfullscreen></iframe>

#Initial config

Nothing special is happening here. We only need to add two images for buttons in feed element (**like** and **comment** icons) and modify current mock images. It was done in [this commit]. After that, but still before we'll start to implement the new stuff, we have to work on...

#Bugfixes and performance improvements
That's right, even in small projects like **InstaMaterial** you can always find something to fix or to improve. The same is here.

##Toolbar theme
First of all I missed the Toolbar styling. And that's why selector for menu button (left button on the Toolbar) has default, dark version (screen below). 

![Toolbar without styling](/images/4/toolbar_without_styling.png "Toolbar without styling")

Inbox button (right button on the Toolbar) has light version because it uses custom view with selector defined by `android:background="@drawable/btn_default_light"` in `menu_item_view.xml`. 

This inconsistency appears only in Android Lollipop and the fix is quite simple. All we need to do is to add single line to Toolbar in `activity_comments.xml` and `activity_main.xml`:

`app:theme="@style/ThemeOverlay.AppCompat.Dark.ActionBar"` 

Thanks to this line all items hosted inside Toolbar view will have default styling taken from `Dark.Actionbar` theme. 

By the way, if you are interested in styling and theming in Android and want to know what are the main differences between them, there is worth metioning article - [Styling Views on Android (Without Going Crazy)].

After we'll apply new them menu selector should look like below (Lollipop only):

![Toolbar with styling](/images/4/toolbar_with_styling.png "Toolbar with styling")

##RecyclerView elements preloading
Another problem I found in project is feed scroll smoothness right after we launch the app. Almost every time we scroll to the second item, layout stutters for a while. Fortunately the reason of this problem is quite simple. As you probably know `RecyclerView` (and other adapter views like `ListView`, `GridView` etc.) uses recycle mechanism for reusing views (in short, system keeps in memory layout only for items which are visible on screen, and reuse them instead of creating new ones if you scroll it).

The problem with our project is that we have only one feed element visible on the screen after we start the application. When we start scrolling, there are no views to reuse by `RecyclerView`, so in the moment when we reach the second element it has to be layed out. Unfortunately it takes some time, so our `RecyclerView` stutters for this moment. After that scroll becomes smooth because we had at least two elements in memory to reuse (which we don't need to create from scratch).

###How to fix this? 

In our example it's pretty straightforward thanks to `LinearLayoutManager` which we use. All we have to do is to override `getExtraLayoutSpace()` method. According to the [documetation], it should return amount of extra space (in **pixels**) that should be laid out by our LayoutManager:

{% gist frogermcs/58a327605fc020354658 MainActivity_extra_layout_space.java %}

This is how it looks in practice:

Before override `getExtraLayoutSpace()`:

![RecyclerView stuttering](/images/4/stuttered.gif "RecyclerView stuttering")

and after it:

![RecyclerView stuttering fixed](/images/4/not_stuttered.gif "RecyclerView stuttering fixed")

#Feed buttons
Let's start with feed buttons (**like** and **comments** for now). As we can see on the [concept video] they have fancy, circle-shaped selector. 

![Ripple effect](/images/4/ripple.gif "Ripple effect")

##Ripples in Android Lollipop.
Selectors used in feed buttons are nothing more than the **Ripple** effect introduced in Android Lollipop. There are a lot of tutorials and blog posts about ripples so I don't want to dive into this topic so deeply. here are a couple links:

* [Ripples – Part 1] and [Ripples – Part 2] from **Styling Android blog**
* [Implementing Material Design in Your Android app] from official **Android Developers Blog**

Ripple implementation in our example is short and simple:

`res/drawable-v21/btn_feed_action.xml`:

{% gist frogermcs/58a327605fc020354658 btn_feed_action_v21.xml %}

And this is how we use it in `feed_item.xml` as a buttons background (line 17 and 24):

{% gist frogermcs/58a327605fc020354658 item_feed.xml %}

##Don't use ripples in pre-21 Android version
I know that I promised to reproduce almost everything from the [concept video], but sometimes it is better to not do something rather than do something which doesn't meet the expectations. And so it is with the ripple effect in pre-21 Android versions.

Of course right now we have some libraries which [mimic ripple effect] on older Androids, but none of them work as it should. They need extra code (i.e. additional layout wrapper), don't use native **Ripple** effect in Android Lollipop, have performance issues etc.

But why is so hard to copy **Ripple** effect into pre-21 Android?

###Ripples - under the hood
Before Android Lollipop whole UI was managed in one "main" thread - UI Toolkit thread. Almost everyone knows [ANR] dialogs, the most of us faced with [NetworkOnMainThreadException] and knows the golden rule - move all long-running operations (networking, database access, image processing) to worker threads and handle only the results in UI thread. And everything would be ok except one rule from first sentence - whole UI have to be managed in main thread.

With more complex and rich apps layouts, The UI itself becomes much more demanding and needs more time for measure, draw and layout operations. And here is the problem. If we are in the middle of animation and start another UI task (like inflating layout for newly opened Acitvity) the problem is that both of these tasks are performed in the same thread, so that animation is going to stop while the activity launches.

Render thread introduced in Android Lollipop helps with these situation by breaking apart two processes of rendering. In short we have list of atomic animations created in UI toolkit thread and then we send them into render thread which exists separately. Thanks that it will continue to perform these atomic animations even if the UI toolkit thread is doing an expensive operations (like inflating an activity for example).

And actually this is how ripples work. They are executed in render thread, completely autonomous of the UI Toolkit thread, thanks that they cannot be interrupted or stopped even if the new activity window is coming up.

And that's why there is no (simple) way to achieve ripple effect in pre-21 Android system.

##Pre-ripple selector in older Android 
Because wa cannot do ripples let's make something similar. We create circle selector with enter and exit fade duration. Maybe it's not as effecitve as ripples, but still looks ok. Here is an example:

And the implementation of `res/drawable/btn_feed_action.xml`:

{% gist frogermcs/58a327605fc020354658 btn_feed_action.xml %}

#Send comment button
Much more interesting for us is **SendCommentButton**. As you can notice on the [concept video] it has two states with simple translate animation between them and ripple onClick selector. 

![Send comment button](/images/4/send_comment.gif "Send comment button")

For this button we'll use a couple Android elements:

* [ViewAnimator] as a base class for our button. It has one functionality which is the most interesting for us - it can perform enter and exit animations when switching between its child views.
* Custom view composed in .xml and inflated in code
* `<merge>` tag for eliminate redundant view groups

##Implementation

Ok, let's start from the animations. Actually we need four of them - 2 (enter and exit) for Send state and 2 (enter and exit, but reversed) for Done state:

* Send state:

	* Slides out at the top:

	{% gist frogermcs/58a327605fc020354658 slide_out_send.xml %}

	* Slides in from the top:

	{% gist frogermcs/58a327605fc020354658 slide_in_send.xml %}

* Done state:

	* Slides in from the bottom:

	{% gist frogermcs/58a327605fc020354658 slide_in_done.xml %}

	* Slides out at the bottom:

	{% gist frogermcs/58a327605fc020354658 slide_out_done.xml %}

Now let's implement layout for our button:

{% gist frogermcs/58a327605fc020354658 view_send_comment_button.xml %}

We used `<merge>` tag because our `SendCommentButton` will be view group for itself. Thanks that we can eliminate one redundant ViewGroup.

And the last thing - the implementation. For now we have only one requirement for our button - after we switch state to the `STATE_DONE`, it should switch it back to the `SEND_STATE` after two seconds. 

The whole code is pretty straightforward:

{% gist frogermcs/58a327605fc020354658 SendCommentButton.java %}

* `init()` method (line 29) inflates our previously created layout directly inside our `ViewAnimator`. By the way check this great article about [proper Layout Inflation] and its parameters which you probably didn't care about before.
* `onDetachedFromWindow()` removes `revertStateRunnable` callback in case you scheduled one, but haven't finished yet (i.e. you destroy the Acitivty before button state switches back).

The rest is quite simple. By invoking `setInAnimation()` and `setOutAnimation()` we switch animation for enter end exit action. 

Great, we've just finished `SendCommentButton` implementation.

##Design
The same as in feed action buttons, we have to prepare two different selectors for our `SendCommentButton`. For Lollipop version again we'll use ripple effect (this time bounded in button frame). For pre-21 Android we'll create standard version with pressed state and shadow emulated in the same way like FAB button described in the first post of this series.

Here is the code for both .xml files:

`res/drawable-v21/btn_send_comment.xml`:
{% gist frogermcs/58a327605fc020354658 btn_send_comment_v21.xml %}

`res/drawable/btn_send_comment.xml`:
{% gist frogermcs/58a327605fc020354658 btn_send_comment.xml %}

##Bonus - shake on error
At the end we'll add one more functionality not showed in the [concept video] - shake animation in case comment text field will be empty. Here is the preview of this effect (it looks much better on the real device):

![Error shaking](/images/4/shake_error.gif "Error shaking")

Implementation is pretty simple - we have to create shake animation and custom `CycleInterpolator` for repeating this animation. Everything is in a few lines of code:

`res/anim/shake_error.xml`:
{% gist frogermcs/58a327605fc020354658 shake_error.xml %}

`res/anim/cycle_2.xml`:
{% gist frogermcs/58a327605fc020354658 cycle_2.xml %}

If we want to use it in code we have start this animation of View like below:

`btnSendComment.startAnimation(AnimationUtils.loadAnimation(this, R.anim.shake_error));`

And that's all for today! Here is the [last commit] with our `SendCommentButton` usage (it's only a boilerplate not worth mentioning). In the next post we'll finally move on to the next period in concept video.

##Source code
Full source code of described example is available on Github [repository].

*Author: [Miroslaw Stanek]*

[INSTAGRAM with Material Design concept]:https://www.youtube.com/watch?v=ojwdmgmdR_Q
[concept video]:https://www.youtube.com/watch?v=ojwdmgmdR_Q
[this commit]:https://github.com/frogermcs/InstaMaterial/commit/570645d2ec2f792be3f2f4523af083fe9df2799d
[documetation]:https://developer.android.com/reference/android/support/v7/widget/LinearLayoutManager.html#getExtraLayoutSpace(android.support.v7.widget.RecyclerView.State)
[Styling Views on Android (Without Going Crazy)]:http://blog.danlew.net/2014/11/19/styles-on-android/
[Ripples – Part 1]:https://blog.stylingandroid.com/ripples-part-1/
[Ripples – Part 2]:https://blog.stylingandroid.com/ripples-part-2/
[Implementing Material Design in Your Android app]:http://android-developers.blogspot.com/2014/10/implementing-material-design-in-your.html
[mimic ripple effect]:https://android-arsenal.com/search?q=ripple
[ANR]:http://developer.android.com/training/articles/perf-anr.html
[NetworkOnMainThreadException]:http://developer.android.com/reference/android/os/NetworkOnMainThreadException.html
[ViewAnimator]:http://developer.android.com/reference/android/widget/ViewAnimator.html
[proper Layout Inflation]:http://www.doubleencore.com/2013/05/layout-inflation-as-intended/
[Miroslaw Stanek]:http://about.me/froger_mcs
[repository]:https://github.com/frogermcs/InstaMaterial
[last commit]:https://github.com/frogermcs/InstaMaterial/commit/1dac9dd2ac6f5f89e09c2791b6db06738e6b6b93