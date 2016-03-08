---
layout: post
title: Dependency injection with Dagger 2 - Introduction to DI
tags: [android, dependency injection, dagger, dagger 2]
---

Some time ago, at Google I/O 2015 Extended in [Tech Space] in Cracow I had a [presentation] about dependency injection with Dagger 2. In a time of preparation I realized that there is a lot of things to talk about and there is no chance to cover everything in dozen of slides. But it could be a good entry point to start the new series of posts - Dependency Injection in Android.

<script async class="speakerdeck-embed" data-id="4b55a6f3efac405d9209aa731c9a74c9" data-ratio="1.33333333333333" src="//speakerdeck.com/assets/embed.js"></script>

In this post I'd like to summarize my presentation by going through it. Maybe not step by step - I think it's time to break with the past and not come back to solutions which are not or shouldn't be used. **Jake Wharton** [talked] about history (Guice, Dagger 1), **Gregory Kick** [too] (almost a half presentation about Spring, Guice, Dagger 1). I also spent a couple of minutes talking about former solutions. But now it's time to start from the point where we are currently.

# Dependency injection

Dependency injection is all about constructing objects and passing them where they are needed. I won't delve into theory (visit Wikipedia's [DI definition] for it). Instead just imagine simple class: `UserManager` with two dependencies `UserStore` and `ApiService`. Without dependency injection this class would look like this:

![UserManager without DI](/images/13/user_manager_no_di.png "UserManager without DI")

Both, `UserStore` and `ApiService` are constructed and provided inside `UserManager` class:

{% gist frogermcs/907485126fcb2bda8978 UserManagerNoDi.java %}

Why this code is a bit problematic? Let's imagine that, you want to change `UserStore` implementation and use `SharedPreferences` as a storing mechanism. It needs at least `Context` object to create an instance so it should be passed to our `UserStore` via constructor. It means that also `UserManager` class has to be updated to handle new `UserStore` constructor. Now imagine dozen of classes which use `UserStore` - all of them have to be updated.

Now take a look at our `UserManager` class with dependency injection:

![UserManager with DI](/images/13/user_manager_di.png "UserManager with DI")

Its dependencies are created and provided from the outside of the class:

{% gist frogermcs/907485126fcb2bda8978 UserManagerDi.java %}

Now in the similar situation - we're changing the implementation of one of its dependencies - there is no need to update `UserManager` source code. All its dependencies are provided from the outside so the only place in code we have to update is the place in which we're constructing `UserStore` object.

So what are the advantages of dependency injection usage?

### Construction/usage seperation

We're constructing instances of classes once - usually in other places that these objects are used. Thanks to this approach our code is more modular - all dependencies can be replaced in easy way (as long as they have the same interface) with no impact on logic of our application. Want to change `DatabaseUserStore` to `SharedPrefsUserStore` ? Fine, just take care about public API (to be the same as `DatabaseUserStore`) or just implement the same interface. 

### Unit testing

The true unit testing assumes that the class has to be tested in complete isolation - without knowledge about its dependencies. In practice, based on our `UserManager` class here is an example of unit tests which we would write:

{% gist frogermcs/907485126fcb2bda8978 UserManagerTests.java %}

And this is possible only with DI - thanks to that `UserManager` is completely independent of `UserStore` and `ApiManager` implementations. We can provide mocks of these classes (in short - mocks are classes with the same public API which does nothing in method calls and/or returns values which we expect) and test `UserManager` in isolation from the real implementations of its dependencies.

### Independend/concurrent development

Thanks to code modularity (`UserStore` can be implemented independently from `UserManager`) it's easy to split the code between programmers. Only the interface of `UserStore` has to be known by everyone (especially public methods of `UserStore` used in `UserManager`). The rest (implementation, logic) can be tested i.e. by unit tests.

# Dependency injection frameworks

Besides advantages dependency injection pattern has some drawbacks. One of them is bigger boilerplate. Just imagine simple `LoginActivity` class which is implemented with MVP (model-view-presenter) pattern. This class could look like this:

![LoginActivity diagram](/images/13/login_activity_diagram.png "LoginActivity diagram")

Source code which is responsible only for `LoginActivityPresenter` initialization could look like below:

{% gist frogermcs/907485126fcb2bda8978 LoginActivity.java %}

Doesn't look so friendly, does it?

And this is the problem which DI frameworks resolve. The same code which uses them can look like this one:

{% gist frogermcs/907485126fcb2bda8978 LoginActivityWithDi.java %}

Much simpler right? Of course DI frameworks don't take objects from nowhere - they still have to be initialized and configured in some place in our code. But objects construction is separated from the usage (actually this is the premise of DI pattern). And DI frameworks care about how to wire everything together (how to deliver objects to places in which they are requested).

# To be continued

Everything I described above was a light background to Dagger 2 - dependency injection framework which can be used in Android and Java development. In next post I'll try to go through whole Dagger 2 API. In case you don't want to wait just try my [Github client example] which is built on top of Dagger 2 and was used with my presentation. Just a hint - `@Modules` and `@Components` are the places to construct/provide objects. `@Inject` are the places where our objects are used.  
More detailed description - soon.

#References

- [DAGGER 2 - A New Type of dependency injection ](https://www.youtube.com/watch?v=oK_XtfXPkqw) by Gregory Kick
- [The Future of Dependency Injection with Dagger 2 ](https://www.parleys.com/tutorial/the-future-dependency-injection-dagger-2) by Jake Wharton
- [Dagger 1 to 2 migration process ](http://frogermcs.github.io/dagger-1-to-2-migration/)

###Author

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo]

[Tech Space]:http://gototech.space/
[talked]:https://www.parleys.com/tutorial/5471cdd1e4b065ebcfa1d557/
[too]:https://www.youtube.com/watch?v=oK_XtfXPkqw
[presentation]:https://speakerdeck.com/frogermcs/dependency-injection-with-dagger-2
[DI definition]:http://en.wikipedia.org/wiki/Dependency_injection
[Github client example]:https://github.com/frogermcs/GithubClient
[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo]:https://azimo.com
