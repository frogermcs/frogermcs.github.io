---
layout: post
title: InstaMaterial concept (part 7) - Navigation Drawer
tags: [android, material design, ui, DrawerLayout, sliding panel, menu, DrawerLayoutInstaller, Navigation Drawer]
---

This post is a part of a series of posts showing Android implementation of [INSTAGRAM with Material Design concept]. Today we'll create Navigation Drawer - left sliding panel which shows global application menu. This element is presented between the 32nd and 35th second of the concept video.

Also we'll create [DrawerLayoutInstaller] - simple tool for injecting DrawerLayout into Activity layout without messing with xml file.

The most recent app version is available in Google Play Store

[!["Android app on Google Play](https://developer.android.com/images/brand/en_app_rgb_wo_60.png)](https://play.google.com/store/apps/details?id=io.github.froger.instamaterial)

<!-- more -->

Here is the final effect described in today's post:

<iframe width="420" height="315" src="//www.youtube.com/embed/rRYN1le1-ZM" frameborder="0" allowfullscreen></iframe>

#Introdution

Navigation Drawer is a very well known design pattern used in Android (and other mobile platforms). It's a panel that transitions in from the left edge of the screen and displays the app’s main navigation options. Sometimes it transitions in from the right edge, but until you have a good reason for this, it's rather bad design implementation, which you shouldn't copy.

Navigation Drawer pattern is also well documented (from both - design and programming sides). If you want to deep into design rules just check [Navigation Drawer design guide]. Also today I won't write too much about programming this pattern. Instead of this I recommend you to read the official documentation of [Creating a Navigation Drawer].

In this post, instead of copying the documentation, we'll prepare **DrawerLayoutInstaller** - a stub of simple tool, which can help us to configure and inject `DrawerLayout` into every Activity whithout messing with each xml file. 

#Navigation Drawer implementation

##Preparation

Before we start, let's make some preparation by adding resources, and less meaning boilerplate.

First of all we have to add all icons used in app's menu:

![New resources](/images/8/new-resources.png "New resources")

Not each of them looks like on the [concept video], but all are taken from the [Material Design icons by Google].

Our menu is built on top of `ListView`, so we have to prepare layout for list items used in it. Here are three different layouts:

* Header with user avatar and nickname  
![Menu header](/images/8/menu_header.png "Menu header")

* Menu button  
![Menu item](/images/8/menu_item.png "Menu item")

* Divider (used between menu buttons)  
{% gist frogermcs/44f57794ed7e86bf51c3 item_menu_divider.xml %}

Here is the [full commit with all new resources].

##GlobalMenuView
Now let's prepare layout for our global menu. Basically it's a simple `ListView` and we could build it in xml file. But in case when we want to inject this view into more Activities it's better to prepare it programmatically.

Requirements for this view are simple - just display options list and user profile. So the implementation isn't too complicated:

{% gist frogermcs/44f57794ed7e86bf51c3 GlobalMenuView.java %}

As you probably noticed, user profile view is used as ListView's header. That's why we have to add custom listener for onClick handling. The rest is pretty straightforward.

`GlobalMenuAdapter` is even simpler, so I won't paste source code here. Instead, check this [commit with GlobalMenuView implementation].

##Navigation Drawer
Now it's time to create Navigation Drawer implementation. First of all let's prepare xml root view for our `DrawerLayout`:

{% gist frogermcs/44f57794ed7e86bf51c3 drawer_root.xml %}

This generic layout can host custom root view (in **vContentFrame** element) and custom left drawer item (**vLeftDrawer** element).

###DrawerLayoutInstaller

Now it's time to prepare our tool for injecting `DrawerLayout` into `Activity` layout tree. Probably **DrawerLayoutInstaller** will be developed in the future but now the requirements list is short and simple:

* Injecting DrawerLayout into existing Activity layout without messing with its xml file,
* Possibility to define custom DrawerLayout xml root and custom left item view,
* Some customization (left item width, custom toggler for opening/closing drawer

The first point is done with three simple steps:

1. Get root view from given Activity (we can get it by `activity.findViewById(android.R.id.content)`) and take its only child
2. Put drawer layout as a new child in root view
3. Put taken child into drawer's content view

Implementation:

{% gist frogermcs/44f57794ed7e86bf51c3 DrawerLayoutInstaller_drawerinject.java %}

The rest you can find in full [DrawerLayoutInstaller implementation].

Finally we can put everything together. Firstly in `BaseActivity` we have to construct our `GlobalMenuView` and then just put it into DrawerLayout and inject everything to Activity layout:

{% gist frogermcs/44f57794ed7e86bf51c3 BaseActivity_setupDrawer.java %}

And the last thing - implement `onGlobalMenuHeaderClick()` which should open `ProfileAcitivty`:

{% gist frogermcs/44f57794ed7e86bf51c3 BaseActivity_globalMenuHeaderClick.java %}

In this case handler delays opening until DrawerLayout closes. Here is the final effect:

![Navigation Drawer](/images/8/navigation_drawer.gif "Navigation Drawer")

That's all for today. It was super fast and simple, as always it should be. Thanks for reading! 😄

##Source code
Full source code of described project is available on Github [repository].

*Author: [Miroslaw Stanek]*

[INSTAGRAM with Material Design concept]:https://www.youtube.com/watch?v=ojwdmgmdR_Q
[concept video]:https://www.youtube.com/watch?v=ojwdmgmdR_Q
[Navigation Drawer design guide]:https://developer.android.com/design/patterns/navigation-drawer.html
[Creating a Navigation Drawer]:https://developer.android.com/training/implementing-navigation/nav-drawer.html
[Material Design icons by Google]:https://github.com/google/material-design-icons
[full commit with all new resources]:https://github.com/frogermcs/InstaMaterial/commit/4f42f9693fb74839261dd26759f15b04d0034073
[commit with GlobalMenuView implementation]:https://github.com/frogermcs/InstaMaterial/commit/cfb4ee819fea9fad852a2eeda671b15ccac6ade2
[DrawerLayoutInstaller]:https://github.com/frogermcs/DrawerLayoutInstaller
[DrawerLayoutInstaller implementation]:https://github.com/frogermcs/DrawerLayoutInstaller
[repository]:https://github.com/frogermcs/InstaMaterial
[Miroslaw Stanek]:http://about.me/froger_mcs