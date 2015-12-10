---
layout: post
title: InstaMaterial - RecyclerView animations done right (thanks to Android Dev Summit!)
tags: [RecyclerView, animations, InstaMaterial]
---

We live in times where apps have to be not only useful but also smooth and delightful. Times slightly different than a couple years ago where all we had to do was just `notifyDataSetChanged()` on ListView adapter. Screen just blinked, new data appeared and that's all.  
Today, in a times of RenderThread, MaterialDesign animations and transitions app should show exactly what's happening. User should see when data from his collection just changed or when something new appeared or/and was removed.

---

A few weeks ago we could see (live or streamed) great Android conference - **Android Dev Summit**. During those 2 days of deep technical sessions we could see Android engineering team presenting new things - [Android Studio 2.0], new Gradle plugin, [Instant run] feature, new official emulator announcement and much more.

*By the way if you missed it, I highly recommend to watch whole [playlist] - there are a lot of great sessions about Android development, tools and solutions.*

One of those videos - [RecyclerView Animations and Behind the Scenes] is the reason of this post. [Chet Haase] and [Yigit Boyar] walked through animating items in RecyclerView and showed how to do it right. This can be good starting point how to make our `RecyclerView` more interactive and delightful.

<iframe width="420" height="315" src="https://www.youtube.com/embed/imsr8NrIAMs?list=PLWz5rJ2EKKc_Tt7q77qwyKRgytF1RzRx8" frameborder="0" allowfullscreen></iframe>

While those are only basics, presented solutions can help you with cleaning up your huge `RecyclerView.Adapter` and solve most common issues.

# InstaMaterial meets RecyclerView usage guidelines

Today we'll take a look at RecyclerView animations from practical point of view (soon I'll try to formalize those things and delve into whole RecyclerView bit deeper).

InstaMaterial source code which we want to cleanup is available under [this commit] (most recent head version is already updated with changes described below).

Also what is important - from a user perspective **we'll change nothing** in current app. But from the code we'll try do things right (or at least as clean as it's possible in example app).

## Desired effect

Code which we want to rebuild is responsible for two similar actions:

1. Like feed item by clicking on its main image
2. Like feed item by clicking on like button 

Those animations should be triggered in our RecyclerView

1. Appearance animation - feed item should slide from the bottom when objects was added for the first time
2. Big like animation - heart with circular background should be animated when user clicks on main image
3. Like button animation - heart should rotate and be filled when users clicks on like button or when user clicks on main image (and 2. animation is played)

Here are described animations (recorded from recent app):

### Appearance animation

![Add animation](/images/21/add_anim.gif "Add animation")

### Big like animation

![Big like animation](/images/21/big_like_anim.gif "Big like animation")

### Like button animation

![Big like animation](/images/21/small_like_anim.gif "Big like animation")

## The code

Previously our animations were implemented directly in [FeedAdapter], the `RecyclerView.Adapter` subclass. Everything worked, so what was wrong with this approach?

1. Animations are not what `RecyclerView.Adapter` is designed for. According to [documentation]:  
*Adapters provide a binding from an app-specific data set to views that are displayed within a RecyclerView.*   
Our adapter will have enough amount of line of binding code the have it twice more with animations stuff.
2. By doing animations in adapter we have to care about cancelling them, handling view recycling in proper way, making sure that they are played in a right moment in time and much more. Everything on our own.
3. While single item animations are something doable, objects interactions (moving/swapping items, updating items positions while new object appeared/disappeared) is something more complicated. 
4. RecyclerView creators gave us their official solution: `RecyclerView.ItemAnimator`:  
*This class defines the animations that take place on items as changes are made to the adapter.*  
It works close to `RecyclerView`, handling all mentioned above cases for us. And because of that we can care more about animations quality, than about the logic which handles them properly in scrolling lifecycle. 

Let's take a look at our [FeedAdapter] one more time.

Those lines of codes shouldn't be here:

{% gist frogermcs/feba504160bc59ffac6a interpolators.java %}

{% gist frogermcs/feba504160bc59ffac6a showLoadingView.java %}

We need to have control on when we should animate items and not (items should be animated on first appearance, but not when view is restored in activity restoring process).

---

{% gist frogermcs/feba504160bc59ffac6a likeAnimations.java %}

We should keep our animations somewhere, in case we need to check if they are playing or to cancel them in recycling process.

---

{% gist frogermcs/feba504160bc59ffac6a runEnterAnimation.java %}

Here we run enter animation every time when view is bound and check if it's right moment to do this (items should be animated once). Probably method name is bit confusing for described case.

---

{% gist frogermcs/feba504160bc59ffac6a onBindViewHolder.java %}

In a moment of binding view in `onBindViewHolder()` we're cancelling animations if they are already running. This is because view can be recycled and we don't know if there is any not finished animation.

Methods `updateLikesCounter()` and `updateHeartButton()` are responsible for setting content in both ways (animated and static).

Also there is an issue in our code. 

We're passing position to our buttons:

{% gist frogermcs/feba504160bc59ffac6a holder.java %}

To have it later in our `onClick()` method:

{% gist frogermcs/feba504160bc59ffac6a onClick.java %}

This position index not always will be correct. Especially when we use this position in both approaches: for placing context menu in a right place on a screen and for passing adapter item to it (well, theoretically in this case 😉).

Since RecyclerView can update data in an async way (item views can be moved without updating data - for example with `notifyItemMoved()`) there is a possibility that our position indicates wrong data.

This is very similar to case which [Yigit Boyar] talked about:

![Adapter position](/images/21/adapter_position.png "Adapter position")

We cannot assume that item position will be final (and code on this slide can cause issues). 

Instead we should use those two methods from `RecyclerView.ViewHolder`:

- [getLayoutPosition()]
- [getAdapterPosition()]

## New implementation

Let's start from beginning. Our Feed will be built with those pieces:

- `FeedItemAnimator` which extends `DefaultItemAnimator` (which extends `RecyclerView.ItemAnimator`). It will give us base for default animations performed by RecyclerView (mainly fade in/out) which we can extend in places which are important for us.
- `LinearLayoutManager` - like previously, to make our feed looks like standard list view.
- `FeedAdapter` - for binding data with views (and just for that).

### FeedItemAnimator

Here is the full code of [FeedItemAnimator]. 

And here we have more interesting code from it:

{% gist frogermcs/feba504160bc59ffac6a canReuseUpdatedViewHolder.java %}

When we're animating RecyclerView items we have a chance to ask RecyclerView to keep the previous ViewHolder of the item as-is and provide a new ViewHolder which will animate changes from the previous one (only new ViewHolder will be visible on our RecyclerView). But when we are writing a custom item animator for our layout we should use same ViewHolder and animate the content changes manually. This is why our method returns `true` in this case.

---

{% gist frogermcs/feba504160bc59ffac6a recordPreLayoutInformation.java %}

When we're calling `notifyItemChanged()` method we can pass additional parameters which will help us to decide what animation should be performed.

Examples from our `FeedAdapter`:

- `notifyItemChanged(adapterPosition, ACTION_LIKE_IMAGE_CLICKED);` 
- `notifyItemChanged(adapterPosition, ACTION_LIKE_BUTTON_CLICKED);`

Method `recordPreLayoutInformation()` is used for catching data before it was changed. Then RecyclerView calls `onBindViewHolder()` (in adapter) and at the end ItemAnimator calls `recordPostLayoutInformation()` which catches data after this. 

Thanks to those operations we can have informations about item state before and after its change.

At the end of everything method `animateChange()` is called, with both pre and post `ItemHolderInfo` objects. Here is how it looks in our example:

{% gist frogermcs/feba504160bc59ffac6a animateChange.java %}

We can clearly see that heart button animation is triggered always, but big photo animation is triggered only when user clicks on feed image. This is what we assumed in list of our desired effects.

---

And the second thing - appear animation. It should be triggered when we see our list for the first time. And this is how it's handled:

{% gist frogermcs/feba504160bc59ffac6a animateAdd.java %}

This `RecyclerView.ItemAnimator`'s method is called when we invoke `notifyItemRangeInserted()` in our FeedAdapter. Another way is to call `notifyItemInserted()`.

---

What else?

{% gist frogermcs/feba504160bc59ffac6a endAnimation.java %}

It's good to implement those two methods. Thanks do this RecyclerView will be able stop animations on item view, when it will disappear from a screen (and will be ready to recycle).

Also those two used methods are worth mentioning:

- [dispatchAddFinished()] - should be called when animation from animateAdd() is finished (this will inform RecyclerView that view is ready for recycle).
- [dispatchAnimationFinished()] - as above, for animateChange().

And that's all. Our updated `FeedAdapter` has 200 lines of code less than before and is responsible only for data-view binding. Here is the full [source code] of it.

## Source code

Most recent version of InstaMaterial source code is available on Github [repository].

### Author 

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[playlist]:https://www.youtube.com/playlist?list=PLWz5rJ2EKKc_Tt7q77qwyKRgytF1RzRx8
[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com
[repository]:https://github.com/frogermcs/InstaMaterial/
[Android Studio 2.0]:http://android-developers.blogspot.com/2015/11/android-studio-20-preview.html
[Instant run]:http://tools.android.com/tech-docs/instant-run
[RecyclerView Animations and Behind the Scenes]:https://www.youtube.com/watch?v=imsr8NrIAMs&index=6&list=PLWz5rJ2EKKc_Tt7q77qwyKRgytF1RzRx8
[Chet Haase]:https://twitter.com/chethaase
[Yigit Boyar]:https://twitter.com/yigitboyar
[this commit]:https://github.com/frogermcs/InstaMaterial/tree/0b321ce58c6d98cf8cd8e461fe2db004fc9ab17c
[FeedAdapter]:https://github.com/frogermcs/InstaMaterial/blob/0b321ce58c6d98cf8cd8e461fe2db004fc9ab17c/app/src/main/java/io/github/froger/instamaterial/ui/adapter/FeedAdapter.java
[documentation]:http://developer.android.com/reference/android/support/v7/widget/RecyclerView.html#nestedclasses
[FeedItemAnimator]:https://github.com/frogermcs/InstaMaterial/blob/master/app/src/main/java/io/github/froger/instamaterial/ui/adapter/FeedItemAnimator.java
[getLayoutPosition()]:http://developer.android.com/reference/android/support/v7/widget/RecyclerView.ViewHolder.html#getLayoutPosition()
[getAdapterPosition()]:http://developer.android.com/reference/android/support/v7/widget/RecyclerView.ViewHolder.html#getAdapterPosition()
[source code]:https://github.com/frogermcs/InstaMaterial/blob/984e245c0c792700e1d47a9a726f213868664ab6/app/src/main/java/io/github/froger/instamaterial/ui/adapter/FeedAdapter.java
[dispatchAddFinished()]:http://developer.android.com/reference/android/support/v7/widget/SimpleItemAnimator.html#dispatchAddFinished(android.support.v7.widget.RecyclerView.ViewHolder)
[dispatchAnimationFinished()]:http://developer.android.com/reference/android/support/v7/widget/RecyclerView.ItemAnimator.html#dispatchAnimationFinished(android.support.v7.widget.RecyclerView.ViewHolder)
