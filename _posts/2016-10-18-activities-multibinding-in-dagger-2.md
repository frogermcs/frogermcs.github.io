---
layout: post
title: Activities Subcomponents Multibinding in Dagger 2
tags: [android, dagger 2, multibinding, activity, testing, espresso, subcomponents]
---

A couple months ago, during [MCE³](http://2016.mceconf.com/) conference, Gregory Kick in his [presentation](https://www.youtube.com/watch?v=iwjXqRlEevg) showed a new concept of providing Subcomponents (e.g. to Activities). New approach should give us a way to create ActivitySubcomponent without having AppComponent object reference (which used to be a factory for Activities Subcomponents).
To make it real we had to wait for a new release of Dagger: [version 2.7](https://github.com/google/dagger/releases/tag/dagger-2.7).

## The problem

Before Dagger 2.7, to create Subcomponent (e.g. `MainActivityComponent` which is Subcomponent of `AppComponent`) we had to declare its factory in parent Component:

{% gist frogermcs/e33c129125c98931bfeea32c6a6efee7 AppComponent.java %}

Thanks to this declaration we Dagger knows that `MainActivityComponent` has access to dependencies from `AppComponent`.

Having this, injection in `MainActivity` looks similar to:

{% gist frogermcs/f1f303ca47cce6d5c6deece22d39ae49 ActivityComponent.java %}

The problems with this code are:

- Activity depends on `AppComponent` (returned by `((MyApplication) getApplication()).getComponent())` — whenever we want to create Subcomponent, we need to have access to parent Component object.

- `AppComponent` has to have declared factories for all Subcomponents (or their builders), e.g.: 
`MainActivityComponent plus(MainActivityComponent.ModuleImpl module);`.

### Modules.subcomponents
Starting from Dagger 2.7 we have new way to declare parents of Subcomponents. `@Module` annotation has optional [subcomponents](http://google.github.io/dagger/api/2.7/dagger/Module.html#subcomponents--) field which gets list of Subcomponents classes, which should be children of the Component in which this module is installed.

**Example**:

{% gist frogermcs/ebc20e703efef051cbb34cbca77a0d91 ActivityBindingModule.java %}

`ActivityBindingModule` is installed in `AppComponent`. It means that both: `MainActivityComponent` and `SecondActivityComponent` are Subcomponents of `AppComponent`.  
Subcomponents declared in this way don’t have to be declared explicitly in `AppComponent` (like was done in first code listing in this post). 

## Activities Multibinding

Let’s see how we could use `Modules.subcomponents` to build Activities Multibinding and get rid of AppComponent object passed to Activity (it’s also explained at the end of this [presentation](https://www.youtube.com/watch?v=iwjXqRlEevg&feature=youtu.be&t=1693)). I’ll go only through the most important pieces in code. 
Whole implementation is available on Github: [Dagger2Recipes-ActivitiesMultibinding](https://github.com/frogermcs/Dagger2Recipes-ActivitiesMultibinding).

Our app contains two simple screens: `MainActivity` and `SecondActivity`. We want to be able to provide Subcomponents to both of them without passing `AppComponent` object.

Let’s start from building a base interface for all Activity Components builders:

{% gist frogermcs/6aacee420032d4d4bd868eac9021a988 ActivityComponentBuilder.java %}

Example Subcomponent: `MainActivityComponent` could look like this:

{% gist frogermcs/17155715cb88f84064969e7752261f20 MainActivityComponent.java %}

Now we would like to have `Map` of Subcomponents builders to be able to get intended builder for each Activity class. Let’s use `Multibinding` feature for this:

{% gist frogermcs/cd3117b2afce35d71026305e0e3bd2bb ActivityBindingModule.java %}

`ActivityBindingModule` is installed in `AppComponent`. Like it was explained, thanks to this `MainActivityComponent` and `SecondActivityComponent` will be Subcomponent of `AppComponent`.

{% gist frogermcs/05ad5b0e0c21322ae85d285bfc0bc091 ActivityBindingModule.java %}

Now we can inject Map of `Subcomponents` builder (e.g. to `MyApplication` class):

{% gist frogermcs/8fbb97357819e6a276593bd30909fc48 MyApplication.java %}

To have additional abstraction we created `HasActivitySubcomponentBuilders` interface (because `Map` of builders doesn’t have to be injected into `Application` class):

{% gist frogermcs/4025e0bd9ff5110bcb8a1e244bba7bfb HasActivitySubcomponentBuilders.java %}

And the final implementation of injection in Activity class:

{% gist frogermcs/3c926bba2353d8835dd64e7f2d421b4c MainActivity.java %}

It’s pretty similar to our very first implementation, but as mentioned, the most important thing is that we don’t pass `ActivityComponent` object to our Activities anymore.

##Example of use case — instrumentation tests mocking

Besides loose coupling and fixed circular dependency (Activity <-> Application) which not always is a big issue, especially in smaller projects/teams, let’s consider the real use case where our implementation could be helpful — mocking dependencies in instrumentation testing.

Currently one of the most known way of mocking dependencies in Android Instrumentation Tests is by using [DaggerMock](https://medium.com/@fabioCollini/android-testing-using-dagger-2-mockito-and-a-custom-junit-rule-c8487ed01b56#.eh5zfyou5) (Github [project link](https://github.com/fabioCollini/DaggerMock)). While DaggerMock is powerful tool, it’s pretty hard to understand how it works under the hood. Among the others there is some reflection code which isn’t easy to trace.

Building Subcomponent directly in Activity, without accessing AppComponent class gives us a way to test every single Activity decoupled from the rest of our app.  
Sounds cool, now take a look at code.

Application class used in our instrumentation tests:

{% gist frogermcs/5232f6fb0775c929a4924cdb66c32d0c ApplicationMock.java %}

Method `putActivityComponentBuilder()` gives us a way to replace implementation of ActivityComponentBuilder for given Activity class.

Now take a look at our example Espresso Instrumentation Test:

{% gist frogermcs/1d8d067e0610d2bace0c73933d6e62ad MainActivityUITest.java %}

Step by step:

- We provide Mock of `MainActivityComponent.Builder` and all dependencies which have to be mocked (just `Utils` in this case). Our mocked `Builder` returns custom implementation of `MainActivityComponent` which injects `MainActivityPresenter` (with mocked `Utils` object in it).

- Then our `MainActivityComponent.Builder` replaces the original Builder injected in `MyApplication` (line 28): `app.putActivityComponentBuilder(builder, MainActivity.class);`
- Finally test — we mock `Utils.getHardcodedText()` method. Injection process happens when Activity is created (line 36): `activityRule.launchActivity(new Intent());` and at the and we’re just checking the results with Espresso. 

And that’s all. As you can see almost everything happens in MainActivityUITest class and the code is pretty simple and understandable. 

## Source code

If you would like to test the implementation on your own, source code with working example showing how to create **Activities Multibinding** and mock dependencies in Instrumentation Tests is available on Github: [Dagger2Recipes-ActivitiesMultibinding](https://github.com/frogermcs/Dagger2Recipes-ActivitiesMultibinding).

Thanks for reading!

### Author 

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com
