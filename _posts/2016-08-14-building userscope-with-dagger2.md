---
layout: post
title: Building UserScope with Dagger2
tags: [android, dagger 2, component, user scope, subcomponent]
---

Custom scopes in Dagger 2 can give better control on dependencies which should live unusual amount of time (different than application and screen lifetime). But to implement it properly in Android app we need to keep in mind a couple things like: scope cannot live longer than application process, process can be killed by system and restored in the middle of user flow with new objects instances and more. Today we'll go through all of them and try to implement production ready UserScope.

-

In one of my blog posts about Dagger 2 I [wrote](http://frogermcs.github.io/dependency-injection-with-dagger-2-custom-scopes/) about building **custom scopes** and **Subcomponents**. As an example I used `UserScope` which should live as long as user is logged in.  
Example of scopes lifetime:

![Scopes lifecycle](/images/15/scopes-lifecycle.png "Scopes lifecycle")

While it looks pretty simple, its implementation showed in that post was far away from code which can be production ready. That’s why I’d like to go through this subject again — but more in implementation context.

# Example app

We’ll build 3-screens application which will be able to get user details from [Github API](https://developer.github.com/v3/) (let’s assume that this would be authentication event in production app).

App will have 3 scopes:

Application scope 

- (**@Singleton**) — dependencies which live as long as application.

- **@UserScope** — dependencies which live as long as user session is active (during single application launch).  
**Important**: this scope won’t live longer than application itself. Each new app instance will create new @UserScope (even if user session was not finished between app launches).

- **@ActivityScope** — dependencies which live as long as Activity screen.

## How it works? 

When app gets data from Github API for given username, new screen is opened (UserDetails). Under the hood `LoginActivityPresenter` asks `UserManager` to start session (get data from Github API). When operation succeeds, user is saved to `UserDataStore` and `UserManager` creates `UserComponent`.

{% gist frogermcs/95a06693ef1710ada28f28df9cd274cf UserManager.java %}

**UserManager** is a singleton object so it lives as long as Application and is responsible for `UserComponent` which reference is kept as long as user session exists. 
When user decides to close session, component is removed so all objects which are `@UserScope` annotated should be ready for GC.

{% gist frogermcs/e593714bd539b5a1763b24dce8abe985 UserManager_closeSession.java %}

## Components hierarchy

In our app all subcomponents use `AppComponent` as a root component. Subcomponents for screens which show user related content use `UserComponent` which keeps `@UserScope` annotated objects (one instance per user session).

![Scopes](/images/28/scopes.png "Scopes")

Example component hierarchy for `UserDetailsActivityComponent` can look like this:

{% gist frogermcs/19a8c125b2b9fe54934a97035bf3f57c AppComponent.java %}

For `UserComponent` we use [Subcomponent builder](http://google.github.io/dagger/subcomponents.html#subcomponent-builders)	 pattern to make our code cleaner and have possibility to inject this builder to `UserModule` (`UserComponent.Builder` is used to create component `UserModule.startUserSession` method showed above).

## Restoring UserScope between app launches 

As I mentioned earlier UserScope cannot live longer that application process. So if we assume that user session can be stored between app launches, we need to handle state restoring operation for our `UserScope`. And there are two scenarios which we need to keep in mind:

### User launches app from scratch

This is the most common case which should be pretty straightforward to handle. User launches app from scratch (e.g. by clicking on app icon). `Application` object is created and then first `Activity` (**LAUNCHER**) is started. We need to provide simple logic checking if we have any saved user in our data store. If yes user is forwarded to proper screen (`UserDetailsActivity` in our case) where UserComponent is automatically created.

### Application process was killed by the system

But there is also a case which we very often forget about. Application process can be killed by the system (e.g. because of low memory). It means that all application data is destroyed (application, activities, static fields). Unfortunately this cannot be handled nicely — we don’t have any callbacks in Application lifecycle. To complicate it even more, Android saves activities stack. What means that when user decides to launch app which was killed somewhere in the middle of flow, system will try to bring user back to this screen. 

For us it means that we need to be ready to restore `UserComponent` from any screen in which it’s used.

Let’s consider this example:

1. User minimised app on `UserDetailsActivity` screen
2. Whole app is killed by the system (later I will show how to simulate it)
3. User opens Task Switcher, clicks on our app screen.
4. System creates new instance of `Application`. What means that also new `AppComponent` is created. And then instead of opening `LoginActivity` (which is our launcher activity) system opens `UserDetailsActivity` immediately.

For us it means that `UserComponent` has to be brought back (new instance has to be created). And this is our responsibility. Example solution to do this can look like this:

{% gist frogermcs/c22f41b80db816623fd0795d520cd683 BaseUserActivity.java %}

Each Activity which use UserComponent has to extend our `BaseUserActivity` class (`setupActivityComponent()` method is called in `onCreate()` method in `BaseActivity`).

`UserManager` is injected from `AppComponent` which was created with Application. Session is started in this way:

{% gist frogermcs/ae6fdc70a4192810aa36f0143df23b23 isUserSessionStartedOrStartSessionIfPossible.java %}

### What if user doesn’t exist anymore?

So there is another case to handle — what if user was logged out (e.g. by `SyncAdapter`) and `UserComponent` cannot be created? That’s why we have those lines in our `BaseUserActivity`:

{% gist frogermcs/a98f2da2bade8c74bef650ccd56e2f2b setupUserComponent.java %}

But if there is a chance that UserComponent cannot be created we have to remember that dependency injection won’t happen. That’s why we need to check it every time in `onCreate()` method to prevent from NullPointerExceptions on injected dependencies.

{% gist frogermcs/b5a54d5089a9dcd5cbfaa9b7175b53cd UserDetailsActivity.java %}

## How to simulate application process kill by system

1. Open app on screen which you would like to test
2. Minimize app with system Home button.
3. In Android Studio, Android Monitor select your application and click Terminate
4. Now on your device open Task Switcher, find your app (you still should see preview of last visible screen).
5. Your app was launched, new Application instance was created and Activity was restored.

## Source code

Source code with working example showing how to create and use `UserComponent` is available on Github: [Dagger2 recipes — UserScope](https://github.com/frogermcs/Dagger2Recipes-UserScope).

Thanks for reading!

### Author 

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com
