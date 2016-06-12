---
layout: post
title: Inject everything - ViewHolder and Dagger 2 example
tags: [android, dagger 2, multibinding, autofactory]
---

The main purpose of Dependency Injection pattern which is implemented by Dagger 2 is that DI separates the creation of a client's dependencies from the client's behavior. In practice it means that all calls of `new` operator, `newIstance()` and others shouldn't be invoked in places other than Dagger's Modules. 

### *The price of Dagger-everything*

*The purpose of this post is to show what we can do, not what we should do. That's why it's important to know what we give in return > for separating creation from behavior.*
*If you want to use Dagger 2 for almost everything in your project you will quickly see that big piece of 64k methods count limit is > used by generated code for injections.*

#Inject everything example

Probably the best known code example of Dagger 2 shows how to inject simple object (e.g. Presenter) to Activity and looks similar to this:

{% gist frogermcs/397c3aae28921a86eee1318c2276f6b5 %}

When we start extending this code (let's say we work on Activity which shows list of items) sometimes we take a shortcuts. It means that we don't always @Inject new objects (like adapter in this case):

{% gist frogermcs/a421ff40c8ad3b030e95696b46455ecf %}

And with this approach we act against DI and profits given by it. Calling Adapter constructor inside Activity class means that every time when we need to change something in its implementation (new constructor argument, maybe completely new implementation which shows grid instead of list), we have to update Activity code. 

Sometimes taking a shortcut like this is intended (we know that this code won't be extended more in the future). But today we'll try to make all adapter related code a part of dependency injection approach. As an example we'll extend our [GithubClient](https://github.com/frogermcs/GithubClient) app - its screen which shows list of repositories for given username. 

![Repositories list](/images/27/repositories_list.png "Repositories list")

To make it more complex and closer to real apps our Adapter will have three different view types: normal, extended and featured.

## The beginning

Here you can see starting code base for our example. For me this is how the first iteration of implemented Adapter looks like in most cases. 

**RepositoriesListActivity**

{% gist frogermcs/c1137169de22d237ad667f047e7b06df %}

**RepositoriesListAdapter**

{% gist frogermcs/c4954681c0803523f781e7b80e192e3b %}

## Adapter injection

At the beginning we'll inject our adapter object instead of calling its constructor in Activity class:

**RepositoriesListActivity**

{% gist frogermcs/398409987d2a07be31a6e1ba1dd445df %}

To make this possible we have to initialize `RepositoriesListAdapter` object in our Activity Module:

{% gist frogermcs/d40499eb8689e1895b5b3eef493fa4da %}

Pretty straightforward. Now let's do some refactoring of our `RepositoriesListAdapter` class. Inner static classes for ViewHolders should be moved out of Adapter code:

{% gist frogermcs/13e713582fefcd87a9a90a055b2a9768 %}

To do it fast in Android Studio just put cursor on class name and click F6 (*Move Inner to Upper Level*)

![Refactoring](/images/27/refactoring.png "Refactoring")

## Assisted injection, Auto-factory

In the next step of our refactoring process we would like to move out construction process from `onCreateViewHolder()` method:

{% gist frogermcs/d8b9a95d114e78a6f17be62b9d82f9a4 %}

It's not as straightforward as in previous example. As you can see our ViewHolders have mandatory argument which is View object (`RecyclerView.ViewHolder` class has only one constructor: `public ViewHolder(View itemView)`). It means that every time when we want to create new object, we need to provide view (which is created during runtime so we cannot configure this in advance in Module classes).

This problem is known as an **Assisted Injection** and was wider described on [StackOverflow](http://stackoverflow.com/questions/16040125/using-dagger-for-dependency-injection-on-constructors) and [Dagger Discuss group](https://groups.google.com/forum/#!topic/dagger-discuss/QgnvmZ-dH9c).

Final solution (until there is no official way of doing this in Dagger 2 framework) is to inject Factory object which takes those dependencies as an arguments and creates intended object. In mentioned StackOverflow thread you can see how to do this by hand, but also Google provides us a solution to generate those factories automatically.  
[AutoFactory](https://github.com/google/auto/tree/master/factory) generates factories that can be used on their own or with JSR-330-compatible dependency injectors from a simple annotation. To use it in our project we have to add new dependency in **build.gradle** file:

{% gist frogermcs/d5b7c56da46cd3b636aad6744c0c0ad4 %}

Now all we have to do is to annotate classes for which we would like to create factories:

{% gist frogermcs/eadb4a653bde0430abdd023a135326e5 %}

Arguments of `@AutoFactory` annotation can make our Factory class extending given class or implementing given interface. In our case it will be:

{% gist frogermcs/246f13498fdaba4ec98ea73074978e49 %}

After this update Adapter's `onCreateViewHolder()` method looks like this:

{% gist frogermcs/41f110d2110eef0376aac915f5828c19 %}

Now our code is a bit cleaner but still we call constructors inside it.

## Multibinding

Final step in our refactoring is to initialize our Factory objects in Module class to get rid of constructors calls in Adapter class. We can simply inject them as a `RepositoriesListAdapter` constructor parameters. But it would mean that every time when we decide to add/remove new type of ViewHolder we still would need to update Adapter code by hand.

Instead we can make use from [Multibinding](http://google.github.io/dagger/multibindings.html). Thanks to this feature Dagger allows us to bind several objects into a collection (even when the objects are bound in different modules).  
Our ViewHolder factories have one thing in common: interface `RepositoriesListViewHolderFactory`. It means that we can inject map of Integers (type of Repository object is represented as a `static final int`) and RepositoriesListViewHolderFactory:

{% gist frogermcs/d66e990d059199797bed17e293fcbeaf %}

And here you can see how our map is created:

{% gist frogermcs/dee5923bb9facedacb738ff503a44c4c %}

`@IntoMap` annotation makes those objects a part of map and puts them under keys given in `@IntKey` ([here](https://google.github.io/dagger/api/latest/dagger/multibindings/package-summary.html) you can find more annotations for different types of keys):

Finally, after all of those code improvements `onCreateViewHolder()` method is as simple as possible:

{% gist frogermcs/01e558603e94b4da79798b39e2016acc %}

No constructors, no if-else statements. Everything is done by proper Dagger graph configuration.

And that's all - our adapter and its ViewHolders are now a part of Dagger 2 graph. We are another step closer to **inject everything** in our code. With **AutoFactory** and **Multibinding** it will be much simpler.

Thank you for reading! 🙂

## Complete source code

Here you can find source code of described classes after refactoring:

* [RepositoriesListActivity](https://github.com/frogermcs/GithubClient/blob/4ee4a68bf2170295823020c6354976722b1a632a/app/src/main/java/frogermcs/io/githubclient/ui/activity/RepositoriesListActivity.java)
* [RepositoriesListActivityAdapter](https://github.com/frogermcs/GithubClient/blob/4ee4a68bf2170295823020c6354976722b1a632a/app/src/main/java/frogermcs/io/githubclient/ui/adapter/RepositoriesListAdapter.java)
* [RepositoriesListActivityModule](https://github.com/frogermcs/GithubClient/blob/4ee4a68bf2170295823020c6354976722b1a632a/app/src/main/java/frogermcs/io/githubclient/ui/activity/module/RepositoriesListActivityModule.java)

Full source code of described project GithubClient is available on Github [repository](https://github.com/frogermcs/GithubClient).

### Author 

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com