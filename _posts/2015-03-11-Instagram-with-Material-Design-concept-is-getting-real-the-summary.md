---
layout: post
title: Instagram with Material Design concept is getting real - The Summary
tags: [android, material design, ui, navigation, drawing]
---

*This is just a summary of series of posts showing Android implementation of INSTAGRAM with Material Design concept. If you read previous posts about InstaMaterial project, you can skip this one.*

A couple months ago I started to work on project which implements almost all visual effects showed in the [video created by designer Emmanuel Pacamalan]. This video presents new Google design guidelines - [Material Design]. Everything looked great, but some people started to worry about the bigges problem of Android platform - fragmentation. Material Design was presented with the newest Android system - 5.0 Lollipop which adoption rate looks like below:

![Android Statistics](/images/11/android-statistics.png "Android Statistics")

Still **less than 5%** at this moment (first half of March 2015). Not so good, right?

But actually it was the main reason to start this series. I wanted to prove that we can create beautiful and smooth application which will work on almost all available devices. That's why InstaMaterial project supports all systems versions, starting from 4.0 Ice Cream Sandwich. Thanks to this our app can work on more than 90% of all Android devices (according to the presented statistics). 

To be clear, not every presented effect is available for both Android 4.0 and 5.0 platforms, but the difference is almost invisible.

One more time, take a look at the concept video:

<iframe width="420" height="315" src="//www.youtube.com/embed/ojwdmgmdR_Q" frameborder="0" allowfullscreen></iframe>

And here are the results of our work:

<iframe width="420" height="315" src="//www.youtube.com/embed/VpLP__Vupxw" frameborder="0" allowfullscreen></iframe>

Did you find some similarities? 😃

# Informations and links for InstaMaterial project

The most recent app is available [here] as an .apk file. I cannot put it into Google Play store because of violation of the intellectual property, what is completely understandable. 

This is an open source project and full code is available on [Github repository]. Project isn't 100% bugs free, so feel free to contribute.

## List of posts

Here is the full list of series posts:

- [Instagram with Material Design concept is getting real] - project init configuration, feed layout and intro animation
- [InstaMaterial concept (part 2) - Comments window transition] - custom activity transition for comments view
- [InstaMaterial concept (part 3) - Feed and comments buttons] - ripples, buttons effects
- [InstaMaterial concept (part 4) - Feed context menu] - animated context menu for feed items
- [InstaMaterial concept (part 5) - Like action effects] - animation techniques, ObjectAnimator and AnimatorSet usage
- [InstaMaterial concept (part 6) - User profile] - reveal activity transition for user profile, RecyclerView items animations
- [InstaMaterial concept (part 7) - Navigation Drawer] - patterns for NavigationDrawer implementation
- [InstaMaterial concept (part 8) - Capturing photo] - quick tour and sample implementation of camera 
- [InstaMaterial concept (part 9) - Photo publishing] - custom view drawing
 
And that's all. I hope you enjoyed my work. One more time thank you for your comments, kind words and support. See you soon! 😃

*Author: [Miroslaw Stanek]*

[INSTAGRAM with Material Design concept]:https://www.youtube.com/watch?v=ojwdmgmdR_Q
[Material Design]:http://www.google.com/design/spec/material-design/introduction.html
[concept video]:https://www.youtube.com/watch?v=ojwdmgmdR_Q
[video created by designer Emmanuel Pacamalan]:https://www.youtube.com/watch?v=ojwdmgmdR_Q
[here]:https://github.com/frogermcs/frogermcs.github.io/raw/master/files/10/InstaMaterial-release-1.0.1-2.apk
[Instagram with Material Design concept is getting real]:http://frogermcs.github.io/Instagram-with-Material-Design-concept-is-getting-real/
[InstaMaterial concept (part 2) - Comments window transition]:http://frogermcs.github.io/Instagram-with-Material-Design-concept-part-2-Comments-transition/
[InstaMaterial concept (part 3) - Feed and comments buttons]:http://frogermcs.github.io/InstaMaterial-concept-part-3-feed-and-comments-buttons/
[InstaMaterial concept (part 4) - Feed context menu]:http://frogermcs.github.io/InstaMaterial-concept-part-4-feed-context-menu/
[InstaMaterial concept (part 5) - Like action effects]:http://frogermcs.github.io/InstaMaterial-concept-part-5-like_action_effects/
[InstaMaterial concept (part 6) - User profile]:http://frogermcs.github.io/InstaMaterial-concept-part-6-user-profile/
[InstaMaterial concept (part 7) - Navigation Drawer]:http://frogermcs.github.io/InstaMaterial-concept-part-7-navigation-drawer/
[InstaMaterial concept (part 8) - Capturing photo]:http://frogermcs.github.io/InstaMaterial-concept-part-8-capturing-photo/
[InstaMaterial concept (part 9) - Photo publishing]:http://frogermcs.github.io/InstaMaterial-concept-part-9-photo-publishing/
[Github repository]:https://github.com/frogermcs/InstaMaterial
[Miroslaw Stanek]:http://about.me/froger_mcs