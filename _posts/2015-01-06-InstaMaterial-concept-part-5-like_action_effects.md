---
layout: post
title: InstaMaterial concept (part 5) - Like action effects
tags: [android, material design, ui, animations, property animations]
---

This post is part of a series of posts showing Android implementation of [INSTAGRAM with Material Design concept]. Today we'll create like effects in feed items which are presented between the 20th and 27th second of the concept video.

<!-- more -->

This is the final effect described in today's post (for both Android Lollipop and pre-21 versions):

<iframe width="420" height="315" src="//www.youtube.com/embed/P9-K5staofc" frameborder="0" allowfullscreen></iframe>

<iframe width="420" height="315" src="//www.youtube.com/embed/KbJQ99EY5Yk" frameborder="0" allowfullscreen></iframe>

#Like the feed item

Based on the [concept video], in our project we have two ways for liking feed item - like by heart button and by click on photo (actually I'm pretty sure that it should work after double click, just like in the real Instagram app, but let it be small exercise for you to implement double-click handler 😄).

As usual we have to update and add some resources used in our project. Today we have to add a bunch of hearts:

![Small blue heart](/images/6/heart_small_blue.png "Small blue heart")
![Read heart](/images/6/heart_red.png "Read heart")
![White heart outline](/images/6/heart_outline_white.png "White heart outline")

(starting from left: heart for likes counter, heart for liked button, heart for photo like animation).

##Likes counter
Let's start from the simplest effect. Like counter should be updated with in/out animation (old value goes up, while the new one comes from down). This is how it looks in practice:

![Likes counter](/images/6/likes_counter.gif "Likes counter")

Looks familiar? That's right - this is the same effect like in send comments button described in [third post of series].

This time we'll use [TextSwitcher] - ViewGroup class which helps to animate between labels composed from a couple of TextViews. For our needs we'll use two of them (we'll animate between them with updated text value by calling `TestSwitcher.setText()` method).

First of all let's update our `item_feed.xml` layout by adding like counter to the bottom panel:

{% gist frogermcs/cb79a9111fbbdabaa4d9 item_feed_likes_counter.xml %}

And here is the visual preview from Android Studio for added view:

![Likes counter preview](/images/6/likes_counter_preview.png "Likes counter preview")

`TextSwitcher` used in our code will animate between its two child TextViews. We don't have to worry about hiding them - it's made automatically.

`android:inAnimation` and `android:outAnimation` values are used for transition between previous and next child.

###0dp for better performance

`LinearLayout` which wraps our likes counter uses `layout_weight="1"` (keep in mind that its *parent* is also the `LinearLayout`). When only one child of `LinearLayout` uses weight value it will absorb all the remaining space in parent (like in our case - look on the screenshot above). By using `layout_width="0dp"` (or height in vertical oriented LinearLayouts) we made a little perfromance improvement because our view doesn't have to measure its own size first.

Now let's add some code to our `FeedAdapter`. The only worth mentioning code is `updateLikesCounter()` method (here is the [full commit with all changes]):

{% gist frogermcs/cb79a9111fbbdabaa4d9 FeedAdapter_updateLikesCounter.java %}

This method is used for updating likes counter in two ways:

- **with animation** (for dynamic update - while user performs like action) - it uses `TextSwitcher.setText()` method
- **without animation** (used by `onBindViewHolder()` for setting current likes count for feed item) - uses `TextSwitcher.setCurrentText()` - this method update currently showed TextView's text.

Also, as you probably noticed, we are using *Quantity Strings* for proper handling words with quantity. Here you can find more details about using [Plurals] in Android.

##Button like animation
Now we'll focus on like button animation. Its behavior is a little bit fancier than on [concept video] and looks like on the screenshot below:

![Heart button](/images/6/heart_button.gif "Heart button")

###Bundling multiple animations
This effect is composed from multiple animations bundled in one set. That's why we cannot use `ViewPropertyAnimator` like we did before. Fortunately there is a simple way to play multiple animations which are depends on each other. Thanks to [AnimatorSet] we can bundle animations together and play them simultaneously, sequentially or in the mixed way. It's described wider there: [Choreographing Multiple Animations]

Our like button effect is composed from 3 animations:

- **360 degree rotation** - played at the beginning of our animations set
- **Bounce animation** - composed from two animations played together (`scaleX` and `scaleY` animations), fired right after rotation finishes.

Here is the full source code of method which is responsible for setting like button image (in animated and static way, like in likes counter):

{% gist frogermcs/cb79a9111fbbdabaa4d9 FeedAdapter_updateHeartButton.java %}

In lines 25 and 26 we compose our final animation.

This time we used `ObjectAnimator` which can animate properties of target object. We can use `int`, `float` and `argb` values to animate and use every property of object which has setter which accepts selected type of value. So in our case using `ObjectAnimator.ofFloat()` method with `rotation` value animates rotation by calling `setRotation(Float f)` method.

Here is full commit which adds [like button animation].

##Photo like animation

A little bit more complex is photo like animation. This time we animate values of two different objects (but still in one `AnimatorSet`):

- Circle background below animated heart (scaling up and fading out)
- Heart icon (scalling up and down).

The effect looks like below:

![Photo like](/images/6/photo_like.gif "Photo like")

And here is the source code for this animation:

{% gist frogermcs/cb79a9111fbbdabaa4d9 FeedAdapter_animatePhotoLike.java %}

Again we used `ObjectAnimator` and `AnimatorSet` (composition in lines 40-41).

And that's all for today, thanks for reading! 😄

##Source code
Full source code of described project is available on Github [repository].

*Author: [Miroslaw Stanek]*

[INSTAGRAM with Material Design concept]:https://www.youtube.com/watch?v=ojwdmgmdR_Q
[concept video]:https://www.youtube.com/watch?v=ojwdmgmdR_Q
[third post of series]:http://frogermcs.github.io/InstaMaterial-concept-part-3-feed-and-comments-buttons/
[TextSwitcher]:http://developer.android.com/reference/android/widget/TextSwitcher.html
[full commit with all changes]:https://github.com/frogermcs/InstaMaterial/commit/090cf15ac0a00ef9bbc5f952fd9c9339838a580f
[Plurals]:http://developer.android.com/guide/topics/resources/string-resource.html#Plurals
[AnimatorSet]:http://developer.android.com/reference/android/animation/AnimatorSet.html
[Choreographing Multiple Animations]:http://developer.android.com/guide/topics/graphics/prop-animation.html#choreography
[like button animation]:https://github.com/frogermcs/InstaMaterial/commit/1dcbcf491436c70cae8772bbc3cb57098810aa07
[repository]:https://github.com/frogermcs/InstaMaterial
[Miroslaw Stanek]:http://about.me/froger_mcs