---
layout: post
title: Async Injection in Dagger 2 with RxJava
tags: [android, dagger 2, rxjava, asynchronous]
---

A couple weeks ago I [wrote a post](https://medium.com/@froger_mcs/dependency-injection-with-dagger-2-producers-c424ddc60ba3) about asynchronous dependency injection in Dagger 2 with **Producers**. Objects initialization execution on background thread(s) has one great advantage - it doesn't block main thread which is responsible for drawing UI in real-time ([60 frames per second](https://www.youtube.com/watch?v=CaMTIgxCSqU) for keeping smooth interface).

It's worth pointing out, that slow initialization in not everyone's issue. But even if you really care about it in your code there is always a chance that any external library does disc/network operations in constructor or any `init()` method. If you are not sure about it I suggest you to give a try [AndroidDevMetrics](https://github.com/frogermcs/AndroidDevMetrics) -  my performance metrics library for Android. It can tell you how much time was needed to show particular screens in app and (if you use Dagger 2) how much time was consumed to provide each object in you dependencies graph.

Unfortunately Producers are not designed for Android and have a couple flaws:

- Use Guava as a dependency (can cause 64k methods issue and increase build time)
- Aren't extremely fast (injection mechanisms can block main thread for a few to dozens of milliseconds depending on device)
- Don't use @Inject annotation (a little more mess in code)

While we cannot do much with last two, the first one can compromise Producers in Android projects.

## Async injection with RxJava

Fortunately a lot of Android devs use RxJava (and [RxAndroid](https://github.com/ReactiveX/RxAndroid)) for creating asynchronous code in our apps. Let's try to use it in Dagger 2 asynchronous injection.

### Async @Singleton injection

Here is our heavy object:

{% gist frogermcs/e88824181b15477403bb26bb9d0af419 %}

Now let's create additional `provide...()` method which returns `Observable<HeavyExternalLibrary>` object, which will call this code asynchronously:

{% gist frogermcs/49e0039d249c040a8e3a64301519678d %}

Let's analyze it line by line:

- `@Singleton` - it's important to keep in mind that it will be single instance of `Observable` object, not `HeavyExternalLibrary`. Singleton also prevents from creating additional Observable object.
- `@Provides` - because this method is also a part of `@Module` annotated class. Do you remember [Dagger 2 API](http://frogermcs.github.io/dependency-injection-with-dagger-2-the-api/) 😉? 
- `Lazy<HeavyExternalLibrary> heavyExternalLibraryLazy` object prevents from initializing HeavyExternalLibrary object internally by Dagger (otherwise in a moment of calling `provideHeavyExternalLibraryObservable()` object would be already created).
- `Observable.create(...)` code - it will return `heavyExternalLibrary` object by calling `heavyExternalLibraryLazy.get()` everytime when this Observable will be subscribed.
- `.subscribeOn(Schedulers.io()).observeOn(AndroidSchedulers.mainThread());` - by default RxJava code is executed on a thread in which Observable is created. That's why we should move execution to background thread (`Schedulers.io()` in this case) and just watch for the results on main thread (`AndroidSchedulers.mainThread()`).

Our Observable is injected like any other object in graph. But object `heavyExternalLibrary` itself will be available a bit later:

{% gist frogermcs/ce55b336b86b1a497ac951a1393a88b3 %}

### Async new instance injection

Code above presented how to inject singleton objects. What if we want to inject asynchronously new instances?

Make sure that our object is not @Singleton annotated anymore:

{% gist frogermcs/877e358790c8b21b043e0935db2125b0 %}

Also our `Observable<HeavyExternalLibrary>` provider method should be changed a bit. We cannot use `Lazy<HeavyExternalLibrary>` because it creates new instance only for the first time when `get()` method is called (see [Lazy documentation](http://google.github.io/dagger/api/latest/dagger/Lazy.html) for more details).

Here is the updated code:

{% gist frogermcs/94a92cc9894db97bfbd6221b344ee627 %}

Our `Observable<HeavyExternalLibrary>` can be a singleton but everytime when we call subscribe() on it, we will get new instance of `HeavyExternalLibrary`in onNext() call:

{% gist frogermcs/162044b19463fbca3431f52a91ad42c9 %}

#### Complete async injection

There is another way to do asynchronous injection in Dagger 2 with RxJava. We can just simply wrap whole injection process with `Observable`.

Let's say our injection is performed in this way (code is taken from [GithubClient](https://github.com/frogermcs/GithubClient/) example project):

{% gist frogermcs/83d9f38a57826d637eaff3353c716ca2 %}

To make it asynchronous we just need to wrap `setupActivityComponent()` method with Observable:

{% gist frogermcs/f32faa96c4d97610ad0ac2f46cbc18c1 %}

As it's explained, all `@Inject` annotated objects will be injected in some time in the future. In return injection process will be asynchronous and won't have big impact on main thread.

Of course creating `Observable` objects and additional threads for `subscribeOn()` is not completely free - it will also take some time. It's pretty similar to impact generated by **Producers** code.

![Splash metrics](/images/26/splash_metrics.png "Splash metrics")

Thanks for reading!


### Author 

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com