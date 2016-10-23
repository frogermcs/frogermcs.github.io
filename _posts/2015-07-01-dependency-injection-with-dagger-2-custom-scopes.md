---
layout: post
title: Dependency injection with Dagger 2 - Custom scopes
tags: [android, dependency injection, dagger, dagger 2, scopes]
---

This post is a part of series of posts showing Dependency Injection with Dagger 2 framework in Android. Today I'm going to spend some time with custom scopes - functionality which can be a bit problematic for Dependency Injection beginners.

# Scope - what does it give us?

Almost every project uses singletons - for API clients, database helpers, analytics managers etc. But since we don't care about instantiation (because of Dependency Injection framework), we shouldn't think about how to get these objects in our code. Instead of it `@Inject` annotation should provide us desirable instances.

In Dagger 2 scopes mechanism cares about keeping single instance of class as long as its scope exists. In practice it means that instances scoped in `@ApplicationScope` lives as long as Application object. `@ActivityScope` keeps references as long as Activity exists (for example we can share single instance of any class between all fragments hosted in this Activity).  
In short - scopes give us "local singletons" which live as long as scope itself.  

*Just to be clear - there are no `@ActivityScope` or/and `@ApplicationScope` annotations provided by default in Dagger 2. It's just most common usage of custom scopes. Only `@Singleton` scope is available by default (provided by Java itself).*

# Scopes - practical example

For a better understanding of scopes in Dagger 2 we go straight to the practical example. We're going to implement something a bit more complex than Application/Activity scoping. For this we'll use our [GithubClient] example from [previous post]. Our app should have three scopes:

- **@Singleton** - application scope
- **@UserScope** - scope for classes instances associated with picked user (in real app it could be logged-in user)
- **@ActivityScope** - scope for instances which live as long as the activity (presenters in our case)

Introduced `@UserScope` is the main difference between today's solution and the one from the previous post. From user experience perspective it gives nothing, but from architectural point of view it helps us to provide `User` instance without passing it as an intent parameter. Also classes which required user data in methods parameters (`RepositoriesManager` in our case) can take `User` instance as a constructor parameter (which will be provided from dependencies graph) and can be initialized on demand instead of creating it in app's launch time. It means that `RepositoriesManager` will be created after we take user from Github API (a while before `RepositoriesListActivity` is presented).

Here is a simple visualisation of scopes and components presented in our app.

![Dagger scopes](/images/15/dagger-scopes.png "Dagger scopes")

Singleton (Application scope) is the longest living scope (in practise lives as long as the application). UserScope as a sub-scope of Application scope has an access to its objects (we can always get objects from parent scope). The same with ActivityScope (which lives as long as Activity object) - it can take objects from UserScope and ApplicationScope.

## Example scopes lifecycle

Here is an example lifecycle for scopes in our app:

![Scopes lifecycle](/images/15/scopes-lifecycle.png "Scopes lifecycle")

Singletons live all the time starting from the app launch, while UserScope is created when we get `User` instance from Github API (in real app after user logs in), and destroyed when we come back to SplashActivity (after user logs out in real app). With a new picked user, another `UserScope` is created.  
Each `ActivityScope` exists as long as its Activity object.

## Implementation

In Dagger 2 scopes implementation comes down to a proper configuration of Components. In general we have two ways to do this - with `@Subcomponent` annotation or with Components dependencies. The main difference between them is an objects graph sharing. Subcomponents have access to entire objects graph from their parents while Component dependency gives access only to those which are exposed in Component interface.

I picked first way with `@Subcomponent` annotations. If you used Dagger 1 before it's almost the same as creating a subgraphs from `ObjectGraph`. Moreover we'll use similar naming convention for methods which create a subgraphs (but it's not mandatory).

Let's start from `AppComponent` implementation:

{% gist frogermcs/f1d0c916d91d1a341e4a AppComponent.java %}

It will be root for other subcomponents: `UserComponent` and Activities components. As you probably noticed (especially if you saw [AppComponent implementation] from the previous post) all public methods which return objects from dependencies graph disappeared. Since we have subcomponents we don't need to expose dependencies publicly - subgraphs have access to all of them anyway.

Instead of them we added two methods:

- `UserComponent plus(UserModule userModule);`
- `SplashActivityComponent plus(SplashActivityModule splashActivityModule);`

They mean that from our `AppComponent` we'll be able to create two subcomponents: `UserComponent` and `SplashActivityComponent`. As they are subcomponents of AppComponent, both will have access to instances produced by `AppModule` and `GithubApiModule`. 

*Naming convention for this method is: returned type is a subcomponent class, method name is arbitrary, parameters are modules required in this subcomponent.*

As you can see `UserComponent` needs another module (which is passed as a parameter of `plus()` method). Doing this we're extending `AppComponent` graph by an additional objects produced by new module. `UserComponent` class looks like below:

{% gist frogermcs/f1d0c916d91d1a341e4a UserComponent.java %}

Of course `@UserScope` annotation is created by us:

{% gist frogermcs/f1d0c916d91d1a341e4a UserScope.java %}

From `UserComponent` we'll be able to create another two subcomponents: `RepositoriesListActivityComponent` and `RepositoryDetailsActivityComponent`.

{% gist frogermcs/f1d0c916d91d1a341e4a UserComponent.java %}

And what is more important all scoping stuff happens here. All instances taken from `UserComponent` inherited from `AppComponent` still are singletons (in Application scope). But those which are produced by `UserModule` (which is a part of `UserComponent`) will be "local singletons" which live as long as this `UserComponent` instance.  
So every time we create another `UserComponent` instance by calling:

`UserComponent appComponent = appComponent.plus(new UserModule(user))`

objects taken from `UserModule` are different instances.

But what is important here - we're responsible for `UserComponent` lifecycle. So we should care about initialization and release of it. In our app example I created two additional methods for this:

{% gist frogermcs/f1d0c916d91d1a341e4a GithubClientApplication.java %}

`createUserComponent()` is called when we get `User` object from Github API (in`SplashActivity`). And `releaseUserComponent()` is called when we come from `RepositoriesListActivity` (in this moment we don't need user scope anymore).

## Scopes in Dagger 2 - under the hood

It's always good to have a look under the hood how things work. Actually in this case to be sure that there is no magic in Dagger's scopes mechanism. 

We'll start our investigation from `UserModule.provideRepositoriesManager()` method. It provides `RepositoriesManager` instance which should be scoped in `@UserScope`. Let's check where this method is called (line 8):

{% gist frogermcs/f1d0c916d91d1a341e4a UserModule_ProvideRepositoriesManagerFactory.java %}

`UserModule_ProvideRepositoriesManagerFactory` is a just implementation of Factory pattern which gets `RepositoriesManager` instance from `UserModule`. We should dig further.

`UserModule_ProvideRepositoriesManagerFactory` is used in `UserComponentImpl` - implementation of our component (line 15):

{% gist frogermcs/f1d0c916d91d1a341e4a UserComponentImpl.java %}

`provideRepositoriesManagerProvider` object is responsible for providing `RepositoriesManager` instance every time when we request for it. As we can see provider is implemented by `ScopedProvider`. Take a look at a part of its source code:

{% gist frogermcs/f1d0c916d91d1a341e4a ScopedProvider.java %}

Could it be more simple? At the first call `ScopedProvider` takes instance from factory (`UserModule_ProvideRepositoriesManagerFactory` in our case) and stores this objet like Singleton pattern. And our scoped provider is just a field in `UserComponentImpl` so in short it means that `ScopedProvider` returns single instance as long as its Component exists.

Here you can find full implementation of [ScopedProvider]. 

And that's it. We've just figured out how scopes work under the hood in Dagger 2. And now we know that they are not connected with scopes annotations in any way. Custom annotations give us only a simple way to code validation in compile time and mark classes as single/non-single instances. All scoping stuff is connected to Component's lifecycle.

And that's all for today. I hope that scopes will become even simpler to use from now. Thanks for reading!

## Source code
Full source code of described project is available on Github [repository].

### Author

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo]

[Singleton]:https://en.wikipedia.org/wiki/Singleton_pattern
[previous post]:http://frogermcs.github.io/dependency-injection-with-dagger-2-the-api/
[AppComponent implementation]:https://github.com/frogermcs/GithubClient/blob/1bf53a2a36c8a85435e877847b987395e482ab4a/app/src/main/java/frogermcs/io/githubclient/AppComponent.java
[GithubClient]:https://github.com/frogermcs/GithubClient
[ScopedProvider]:https://github.com/google/dagger/blob/master/core/src/main/java/dagger/internal/ScopedProvider.java
[repository]:https://github.com/frogermcs/GithubClient
[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo]:https://azimo.com
