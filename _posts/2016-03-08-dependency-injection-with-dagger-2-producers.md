---
layout: post
title: Dependency injection with Dagger 2 - Producers
tags: [android, dependency injection, dagger, dagger 2, producers]
---

This post is a part of series of posts showing Dependency Injection with Dagger 2 framework in Android. Today we're going to take a look at Dagger Producers - an extension to Dagger 2 that implements asynchronous dependency injection in Java.

# Initialization performance issues

We all know that Dagger 2 is very well optimized framework for Dependency Injection. But even with all those micro-optimizations, still there are possible performance issues with injected dependencies - the "heavy" 3rd parties and/or our main-thread-blocking code.  
Dependency injection is a process of delivering requested dependencies in the right place in the shortest possible time - and this is what Dagger 2 does very well. But DI is also about creating those dependencies. What is the point of delivering dependencies in nano seconds if we need to spend hundred of milliseconds creating them?

Maybe it's not so important later, when our app has created serious bunch of singletons which are now served instantly by Dagger 2. But still we need a time window when we create them - and in the most cases it's app launch time.

The problem (and hints how to debug it) was described wider in one of my previous blog posts: [Dagger 2 - graph creation performance](http://frogermcs.github.io/dagger-graph-creation-performance/).

In very short, let's imagine this case - your app has initial screen (SplashScreen) which does all needed things immediately after app launch:
* Initializes all tracking libs (Google Analytics, Crashlytics) and sends first bunch of data to them
* Creates whole stack for API and/or database communication
* Delivers logic to our views (Presenters in MVP, ViewModels in MVVM etc.) 

Even if our code is very well optimized still there is a chance that some of external libraries need tens or hundreds of milliseconds to initialize. Before our launch screen will be presented all requested dependencies (and their dependencies) have to be initialized and delivered. It means that launch time is a sum of initialization times of each of them.

Example stack measured by [AndroidDevMetrics](https://github.com/frogermcs/androiddevmetrics) can look like this:

![Example dependencies](/images/24/example_dependencies.png "Example dependencies")

User will see SplashActivity in about 600ms (+ additional system work) - sum of all initialization times. 

#Producers - asynchronous dependency injection

Dagger 2 has extension named **Producers** which more or less solve those problems for us.  
The idea is simple - whole initialization process can be executed on background thread(s) and delivered later to app's main thread.

Similar to Dagger synchronous injection we have a couple annotations which are used in injection process:

### @ProducerModule
Same as `@Module`, this one is used to mark classes which deliver depedencies.  Thanks to this, Dagger will know where to find requested dependencies.

### @Produces
Similar to `@Provide` this annotation is used to mark methods in `@ProducerModule` annotated class which return dependencies. `@Produces` annotated methods can return `ListenableFuture<T>` or object itself (which also will be initialized in given background thread).

### @ProductionComponent
Like `@Component`, it's responsible for dependencies delivery. It's a bridge between our code and `@ProducerModule`. The only difference from `@Component`  is that we cannot decide about scope of dependencies. It means that *each Produces method that contributes to the component will be called at most once per component instance, no matter how many times that binding is used as a dependency.*

It means that each object served by `@ProductionComponent` will be a single instance (as long as we're getting it from this particular component).

---

Producers documentation is pretty good so there is no sens to copy it here. Instead just take a look: [Dagger 2 Producers docs](http://google.github.io/dagger/producers.html).  

## The price of producers
Before we start implementation there are a couple things worth mentioning. Producers are a bit more complicated than Dagger 2 itself. It looks that mobile apps are not their main purpose of usage and it's important to know a couple things about them:
* Producers use Guava library and are built on top of ListenableFuture class. It means that you have to deal with a 15k of additional methods in your app. What probably means that also you have to deal with **Proguard** and longer compile time.
* As you will see later, creating `ListenableFutures` doesn't cost nothing. So if you count on that Producers will help you with optimizations from 10ms to 0ms you're probably wrong. But if the scale is bigger (100ms -> 10ms) you will find what you look for.
* Right now there is no way to use `@Inject` annotation, so you have to deal with ProductionComponents by hand. It can make a mess with your well standarized, clean code.  
[Here](http://stackoverflow.com/questions/35617378/injects-after-produces) you can find a good try with indirect solution for `@Inject` annotation.

# Example app

If you still want to deal with Producers let's update our [GithubClient](https://github.com/frogermcs/GithubClient/) app to use them in injection process. Before and after implementation we'll measure app launch time with [AndroidDevMetrics](https://github.com/frogermcs/androiddevmetrics) and compare the results.

[Here](https://github.com/frogermcs/GithubClient/tree/1c14683691e0e7af17b26055a0fd041d4a7df424) is the version of GithubClient app just before update with producers. And its measured average launch time looks like this:

![Before update](/images/24/before_update.png "Before update")

Our plan is to handle UserManager and its all dependencies with Producers. 

### Configuration

We'll give a try to Dagger v2.1 (but Producers are also available in current version 2.0).

Let's add new version of Dagger to our project:

***app/build.gradle***:
{% gist frogermcs/b7515bf5d69a79923c71 %}

As you can see Producers are delivered as a new dependency, next to Dagger 2 library. Also it's worth mentioning that Dagger v2.1 finally doesn't require `org.glassfish:javax.annotation:10.0-b28` dependency.

### Producer Module

Now let's move code from `GithubApiModule` to newly created `GithubApiProducerModule`. Original source code can be found here: [GithubApiModule](https://github.com/frogermcs/GithubClient/blob/1c14683691e0e7af17b26055a0fd041d4a7df424/app/src/main/java/frogermcs/io/githubclient/data/api/GithubApiModule.java).

***GithubApiProducerModule.java***

{% gist frogermcs/7721959e806e0755216b %}

Looks simillar? That's rigth, we just updated:
* `@Module` to `@ProducerModule` 
* `@Provides @Singleton` to `@Produces`. 
*Do you remember? In Producers we have single instances by default.*

`UserModule.Factory` dependency was added only with app's logic reasons.

### Production Component

Now let's create `@ProductionComponent` which will serve `UserManager` instance:

{% gist frogermcs/bee97c7a05df289dd5f3 %}

Again, pretty similar to ordinary [Dagger's @Component](https://github.com/frogermcs/GithubClient/blob/master/app/src/main/java/frogermcs/io/githubclient/AppComponent.java).

ProductionComponent is also build in very similar way to standard Component:

{% gist frogermcs/7e10758b9f083a621677 %}

Additional parameter is `Executor` instance which tells ProductionComponent where (in which thread) dependencies should be created. In our example we uses single-thread executor but of course it's not a problem to increase level of parallelism and use multi-thread Executors.

### Getting dependecies

As I said, currently there is no way to use `@Inject` annotation. Instead, we have to ask ProductionComponent directly (you can find this code in [SplashActivityPresenter](https://github.com/frogermcs/GithubClient/blob/master/app/src/main/java/frogermcs/io/githubclient/ui/activity/presenter/SplashActivityPresenter.java)):

{% gist frogermcs/5f0a64d33b68bc51e4f6 %}

What is important here, object initialization starts when you call `appProductionComponent.userManager()` for the first time. After this `UserManager` object will be cached. It means that each binding has the same lifetime as its enclosing component instance.

And pretty much that's all. Of course you should remember that userManager instance will be `null` before `Future.onSuccess()` will be called.

## Performance

At the end let's take a look how injection performance looks like right now:

![After update](/images/24/after_update.png "After update")

Yeah, that's right - average value was about 15ms in this case. It's still less than in synchronous injection (avg. 25ms) but not so little as you probably expected. It's because Producers are not as lightweight as Dagger itself. 

So now it's your decision - if it's worth to deal with Guava, Proguard and code complexity for **those** kind of optimizations.

Remember, if you feel that Producers are not a best fit for your app you can always try to wrap your injections with RxJava or another async code used in your app. 

Thanks for reading!

##Source code
Full source code of described project is available on Github [repository].

### Author 

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com
[repository]:https://github.com/frogermcs/GithubClient