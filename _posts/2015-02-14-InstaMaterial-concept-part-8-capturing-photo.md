---
layout: post
title: InstaMaterial concept (part 8) - Capturing photo
tags: [android, material design, ui, camera, capturing photo]
---

This post is a part of a series of posts showing Android implementation of [INSTAGRAM with Material Design concept]. Today we'll create flow for photo capturing. This functionality is presented between the 38th and 41st second of the concept video. We'll omit some details (like FAB button animation, colors, icons) because of a little more work with camera implementation. We'll back to them in next (probably the last) post of series.

According to [previous post], I have to mention that InstaMaterial application has been removed from Google Play Store because of violation of the intellectual property. I fully undertand this reason and in close future I'll probably prepare version with different layout which is completely different from Instagram application (but still has all features presented in concept video).

APK file built from code described in today's post is available [here].

<!-- more -->

Now let's back to this post - here is the final effect which we want to achieve today:

<iframe width="420" height="315" src="//www.youtube.com/embed/0w3lGJIISTo" frameborder="0" allowfullscreen></iframe>

# Introdution

Almost all solutions and animations described today were presented in previous posts. That's why I won't focus on detailed descriptions. But also there will be completely new element - camera. Those who have worked with it before know that this compoment is a little bit problematic for implementation. Why? In short - Android as an opened platform can handle very wide range of devices with completely different hardware (especially camera) specification. So if you are starting your work with picture capturing here is short list of things you should care:

* **Different preview and captured images sizes**  
Every device has a finite number of supported sizes (for both - preview and captured image). You should keep in mind that for example some devices don't handle sizes with 1:1 ratio (squared image) - actually Nexus 5 is one of that devices. So be prepared for dealing with bitmap operations.
* **Back/front camera**  
Every device can have back and front camera. Of course there are devices with only one - back, but keep in mind that some of them has only front camera (i.e. Nexus 7 2012). Of course each of them can have different supported sizes list.
* **Different supported features**  
Some of devices don't support autofocus, another don't have zoom. Before us start using any feature you should check if it exists.
* **Handling camera operations is very expesive**  
You should keep in mind that images (espacially those taken in big resolutions) are very expesive to maintain by device. Image processing isn't the lightest thing so you should take care about memory management and computation outside of main UI thread. Another minus is that camera isn't ready immediately after calling for it - sometimes you should wait for initialization. 

According to some points described above, instead of writing my own implementation I decided to use external library for handling camera operations. I picked [CWAC-Camera library] which is also a little bit complicated for implementation but it takes care about a lot of edge cases. Even if author plans to [completely rewrite it], in my opinion there isn't any better start point to deal with photo/video capturing.

And the last thing before we start. Camera implemetation in InstaMaterial app is very limited. There is no logic for picture file handling and captured image displaying is very tricky. A lot of work should be done before this code will be ready for production (especially with bitmap cropping, picking right size for image etc.). But I trust that it could be good codebase for more complicated projects. 😄

# Preparation
As always let's add some resources and less important boilerplate. Also some libraries have been updated. From now project has also new structure:

![Project structure](/images/9/project_structure.png "Project structure")

Don't treat it as a final reference, but in real projects this is my favorite structure where ui/logic/data has separated trees (another option is to spearate trees by functionality but in smaller projects it can be redundant).

Here is list of commits:

* [libraries update]
* [project structure update]
* [new required resources]

## CWAC-Camera configuration
Camera library installation is very simple. All we have to do is add new maven repository in `<project>/build.gradle` file:

{% highlight groovy %}
repositories {
    maven {
        url "https://repo.commonsware.com.s3.amazonaws.com"
    }
}
{% endhighlight %}

and new dependency in `<project>/app/build.gradle`:

{% highlight groovy %}
dependencies {
	//...
    compile 'com.commonsware.cwac:camera:0.6.12'
}
{% endhighlight %}

Finally we have to add new permissions in `AndroidManifest.xml` file:

{% gist frogermcs/e2d00a89cef336779bf1 AndroidManifest.xml %}

# Photo capture screen
Now we can start implementation of capture screen. I considered two ways of implementation - two different activities for taking photo and editing it or one activity with these states. First option is more correct in my opinion (and probably I would use it in production code) but finally I took second one. Why? Because there is no simple way (but it isn't impossible!) to make transition from camera preview to taken photo preview without little layout hickup. By default camera saves taken picture on external storage and another activity should read it from that place. Unfortunately it generates a little delay so we would have to figure out some solution for passing bitmap between two activities (something more complex than sending bitmap via Intent's bundle or saving bitmap as a static field).

Let's start from intro animation. For Activity transition we can use `RevealBackgroundView` introduced in [one of the previous posts]. One more time we have to pass starting location (taken from FAB button this time). The only difference is that we have to add new style which hides status bar in `TakePhotoActivity`:

{% gist frogermcs/e2d00a89cef336779bf1 styles.xml %}

and set it in `AndroidManifest` file:

{% gist frogermcs/e2d00a89cef336779bf1 AndroidManifest_TakePhotoActivity.xml %}

Commit with all described changes is [available here].

# TakePhotoActivity layout
Now let's prepare layout for capture screen. As I mentioned it should handle two states - capturing and configuring photo. That's why we should implement elements transitions. For this we'll use `ViewSwitcher` which can handle animation between his children (in our project we used `TextSwitcher` for likes counter animation which extends `ViewSwitcher` class).

Before we start layout implementation we should prepare background for capture button:

![Capture button](/images/9/btn_capture.png "Capture button")

and capture options:

![Capture options button](/images/9/btn_capture_options.png "Capture options button")

First one can be created in .xml file. We can use `layer-list` with two circles and circular stroke on top of them:

{% gist frogermcs/e2d00a89cef336779bf1 btn_capture.xml %}

Second one is even simpler - it's only simple circular stroke (inside images are provide in project's resources).

{% gist frogermcs/e2d00a89cef336779bf1 btn_capture_options.xml %}

Remember that both of them don't handle onClick and enable/disable states. You can do it on your own. 😄

Now we have all required elements for TakePhotoActivity layout. Implementation is simple but a little bit complex. That's why I don't show source code. Instead of this here is the visualisation of its layout:

![Take photo layout](/images/9/take_photo_layout.png "Take photo layout")

Top and bottom panels slide from the right side of screen (thanks to `ViewSwitcher` parent). Taken image (white square on the right) appears on top of CameraView when photo is taken.

Here is the [full source code] of .xml file.

# Photo capturing view implementation
Finally we can implement the most important thing - photo capturing. In short here is the list of requirements:

* Intro animation (sliding-in top and bottom panels right after background reveal animation)
* Capturing photo from camera
* Animating between two states - capturing and editing photo

First point is pretty straightforward. Just hide top and bottom panels on Acitivity start and show them when background reveal animation finishes:

{% gist frogermcs/e2d00a89cef336779bf1 TakePhotoActivity_intro.java %}

## Photo capturing
Now we have to configure `CameraView`. According to [CWAC-Camera documentation] we have to integrate this view with Activity lifecycle (onResume() and onPause() mehtods):

{% highlight java %}
@Override
protected void onResume() {
    super.onResume();
    cameraView.onResume();
}

@Override
protected void onPause() {
    super.onPause();
    cameraView.onPause();
}
{% endhighlight %}

If we use `CameraView` in our layout (instead of using `CameraFragment`) our Activity must implement `CameraHostProvider` interface. It provides `getCameraHost()` method which should return `CameraHost` implementation. This interface is resposible for our camera configuration. For our needs we create very simple `MyCameraHost` inner class:

{% gist frogermcs/e2d00a89cef336779bf1 MyCameraHost.java %}

In short it takes care about putting the same size to preview and captured image (thanks to that taken image won't be stretched in compare to preview). `useFullBleedPreview()` means almost the same as centerCrop scale type in ImageView. So if `CameraView` preview has different size than view layout params (in our example it has!) preview will be cropped and fill whole available area.  
`saveImage()` method is called at the end of capture operation (so even if we modify given bitmap it won't affect on file saved on phone memory). But keep in mind that `saveImage()` method will be called only when we call `takePicture()` with proper parameters.

I won't deep into rest of `CameraHost` implementation. Just check the [CWAC-Camera documentation] for more informations.

All we have to do now is to call `takePicture()` methods. It has two boolean arguments: `needBitmap` and `needByteArray`. If first is set to true `saveImage(PictureTransaction xact, final Bitmap bitmap)` will be called in our CameraHost implementation. Second calls `saveImage(PictureTransaction xact, byte[] image)`.

Of course if we want to see shutter effect we have to implement it on our own. Here is how it looks in our project:

{% gist frogermcs/e2d00a89cef336779bf1 TakePhotoActivity_take_photo.java %}

And the last thing - we should handle taken photo and show it. In our example it works in this way: on top of CameraView we have hidden ImageView. If only we get new bitmap from `saveImage()` method we put it into `ImageView` and show it on top of `CameraView`. After pressing back ImageView is hidden and user can see CameraView again. Here is how it looks in our code:

{% gist frogermcs/e2d00a89cef336779bf1 TakePhotoActivity_setup_photo.java %}

Here is the full source code of [TakePhotoActivity].

And that's all for today. Thank you all for the reading! 😄

## Source code
Full source code of described project is available on Github [repository].

*Author: [Miroslaw Stanek]*

[INSTAGRAM with Material Design concept]:https://www.youtube.com/watch?v=ojwdmgmdR_Q
[concept video]:https://www.youtube.com/watch?v=ojwdmgmdR_Q
[previous post]:http://frogermcs.github.io/InstaMaterial-concept-part-7-navigation-drawer/
[here]:https://github.com/frogermcs/frogermcs.github.io/raw/master/files/9/instamaterial-debug.apk
[CWAC-Camera library]:https://github.com/commonsguy/cwac-camera
[completely rewrite it]:http://commonsware.com/blog/2014/12/01/my-mistakes-cwac-camera.html
[libraries update]:https://github.com/frogermcs/InstaMaterial/commit/cabb984c918e11fddb333bf4e5a2593020f7d633
[project structure update]:https://github.com/frogermcs/InstaMaterial/commit/9c62d60682990ba549c3fe72b82c956088a61188
[new required resources]:https://github.com/frogermcs/InstaMaterial/commit/36381c47e47d4da1ae3ce32702fba598fd96346f
[one of the previous posts]:http://frogermcs.github.io/InstaMaterial-concept-part-6-user-profile/#custom-revealbackgroundview
[available here]:https://github.com/frogermcs/InstaMaterial/commit/e19395ef14a0a29a558a43d36de050958351f99c
[full source code]:https://github.com/frogermcs/InstaMaterial/blob/884b6b3219fc8175a1db364135926efd482bf2c4/app/src/main/res/layout/activity_take_photo.xml
[CWAC-Camera documentation]:https://github.com/commonsguy/cwac-camera
[TakePhotoActivity]:https://github.com/frogermcs/InstaMaterial/blob/29dfdc0ae8ff74d3508f9ae84bbec8cb22c80223/app/src/main/java/io/github/froger/instamaterial/ui/activity/TakePhotoActivity.java
[repository]:https://github.com/frogermcs/InstaMaterial
[Miroslaw Stanek]:http://about.me/froger_mcs