---
layout: post
title: Hello world Google Home
tags: [Google Home, Google Actions, Java, Android, Google Assistant]
---

## Github Actions — building first agent for Google Assistant in Java

Some time ago I [published](https://medium.com/@froger_mcs/building-google-actions-with-java-696cffedbd01) unofficial Google Actions SDK written in Java. Source code and documentation can be found on Github: [Google-Actions-Java-SDK](https://github.com/frogermcs/Google-Actions-Java-SDK). Library can be also downloaded from Bintray jCenter:

{% gist frogermcs/5dd507e968d30d89e4090d6297520b81 gactions.gradle %}

The goal for this project is to give Android/Java developers possibility to build solutions for Google Home without learning new language (official Actions SDK is written in Node.js). 

## Google Assistant Action from scratch

In this post we’ll go through Google Assistant agent implementation. We’ll build simple Github client which will tell us what is the one of the hottest repositories from last days.

Source code for final project can be found here: [Github Google Actions](https://github.com/frogermcs/Github-Google-Actions)

### Environment configuration

Our Github Actions agent will be built in Java and hosted on App Engine.  
At the beginning we need to create new project in [Google Cloud Platform dashboard](https://console.cloud.google.com/home/dashboard).  
When it’s done remember your project ID, it will be needed during configuration. _In example described in this post id is: *githubactions*_.

Then we need to install CLIs for Google Cloud and Google Actions:

- [Instruction for Google Cloud SDK](https://cloud.google.com/sdk/docs/)

- [Instruction for gactions](https://developers.google.com/actions/tools/gactions-cli)

### Project init

Now let’s create our Java/gradle project (I use **[IntelliJ IDEA](https://medium.com/r/?url=https%3A%2F%2Fwww.jetbrains.com%2Fidea%2F)**, free community edition). For better code structure, I also created module app. Final structure looks like this (some files/tries aren’t show for better readability):

{% gist frogermcs/816d0d05d0058fe236b5e2eb75bed2fc structure1 %}

When it’s done, run `$ gactions init` from `/app` directory. It should create example config file: `app/actions.json`:

{% gist frogermcs/b4ad8dc0d89c11d77c5289a642be9572 example.json %}

This config file defines our agent’s Actions. It’s mostly self-describing but in case you would like to know better what is happening here, please visit official [documentation](https://developers.google.com/actions/develop/sdk/actions).

Now let’s update it with our configuration. We would like to have an agent which is able to handle two actions/intents:

- **assistant.intent.action.MAIN** (defined in Google Actions SDK)  
Action is launched when user starts talking, but without clear intention e.g.: “Ok Google, talk to Github”. In most cases this should be used as an agent introduction.

- **com.frogermcs.githubactions.intent.TRENDING** (defined by us)  
In response to this request, our agent will find one of the hottest repositories from last days. There are a couple utterances which should fire this intent:  
“Ok Google, ask Github what are the hottest repositories”  
“…trending repositories”  
“…hot projects”  
…  
and others. List of queries is defined in **actions.json**. Here is final configuration for our agent:

{% gist frogermcs/ada1f417c83ff08dbbc36a92f165dabd githubactions.json %}

###The code

Now let’s build service to handle requests from Google Assistant. We will build simple REST server, which handles POST requests (based on configuration above, both intent requests will be sent to https://githubactions.appspot.com/). 


#### Project configuration

Our app requires a couple dependencies defined in `app/build.gradle` file:

- App Engine SDK

- [Google Actions Java SDK](https://github.com/frogermcs/Google-Actions-Java-SDK/) (my unofficial library)

- [Google HTTP Client Library for Java](https://github.com/google/google-http-java-client) — to make calls to Github API. Not a good news for Android devs familiar with Okhttp and Retrofit — because of App Engine restrictions in threading, it’s really hard to make it working here.

- [Dagger 2](https://google.github.io/dagger/) — Dependency injection framework to make our code a bit cleaner and easy to extend.

Final gradle files (able to build, run and deploy our GAE code) can be found here:

- [app/build.gradle](https://github.com/frogermcs/Github-Google-Actions/blob/master/app/build.gradle)

- [build.gradle](https://github.com/frogermcs/Github-Google-Actions/blob/master/build.gradle)

- [settings.gradle](https://github.com/frogermcs/Github-Google-Actions/blob/master/settings.gradle)

#### Servlet

Now let’s build code for processing requests from Google Assistant. At this level, it’s nothing more than just proper configuration of [Google Actions Java SDK](https://github.com/frogermcs/Google-Actions-Java-SDK/):

- Build response handler. This object will be used to pass final responses to Google Assistant. Nothing really interesting, it just meets communication requirements (headers, format) Final code: [AppEngineResponseHandler](https://github.com/frogermcs/Github-Google-Actions/blob/master/app/src/main/java/com/frogermcs/githubactions/assistant/AppEngineResponseHandler.java).

- Build request handlers. Those are responsible for handling particular requests (our **MAIN** and **TRENDING** intents). Request handlers should process incoming request and generate tell/ask/permission response. In our app we’ll have only tell responses: greeting (MAIN) and randomly picked hot repository (TRENDING). In the future we’ll build more complex solutions on top of ask and permission responses.
Final code for [MainRequestHandler](https://github.com/frogermcs/Github-Google-Actions/blob/master/app/src/main/java/com/frogermcs/githubactions/assistant/requestHandlers/MainRequestHandler.java) and [TrendingRequestHandler](https://github.com/frogermcs/Github-Google-Actions/blob/master/app/src/main/java/com/frogermcs/githubactions/assistant/requestHandlers/TrendingRequestHandler.java) (take a look at returned variable from getResponse() methods — in both cases we use `ResponseBuilder.tellResponse(…)`.

- Intents mapped to proper request handlers:

{% gist frogermcs/f0c54881471ffe99295244639ac3ad9e AssistantModule.java %}

All dependencies for Assistant Actions can be found in [AssistantModule](https://github.com/frogermcs/Github-Google-Actions/blob/master/app/src/main/java/com/frogermcs/githubactions/assistant/AssistantModule.java) class.

Final servlet code should be similar to this:

{% gist frogermcs/7355dcde7ccd51f9a2db6d1869f8dd2b GithubActionsServlet.java %}

Line 15 is the place where all magic happens:

1) POST request comes from Google Assistant. It is parsed to `RootRequest` object (lines: 18–20) and passed to `assistantActions`.

2) Based on defined intents map, `assistantAction` passes `RootRequest` object to `MainRequestHandler` or `TrendingRequestHandler`.

3) Picked request handler generates tell response: `RootResponse`.

4) Response is passed to `AppEngineResponseHandler` which sends it back to **Google Assistant**.

#### Servlet Configuration

At the end you need to add App Engine servlet configuration to your project (files: **web.xml** and **appengine-web.xml**):

{% gist frogermcs/81114876931412d66b723185f475acda structure2 %}

Their content should be self-explaining:

{% gist frogermcs/acdb87f4516e81b396b2048ecc88c09f web.xml %}

{% gist frogermcs/7136f0a9f306a7ddd36248ae3007440a appengine-web.xml %}

Now our app is ready to deploy. If the configuration is correct, all you have to do is:

`$ ./gradlew app:appengineDeploy`

### Testing Assistant Actions

![Web Simulator](/images/31/web_simulator.png "Web Simulator")

Even if you don’t have Google Home device, it’s still possible to simulate you newly created Assistant Actions. To do this you can use **Google Home Web Simulator**. All you need to do is:

1) `$ gactions preview -action_package=app/action.json -invocation_name=Github` called from project’s root directory.

2) Sign-in and start: [Web Simulator](https://developers.google.com/actions/tools/web-simulator)

More details can be found in [official guide](https://developers.google.com/actions/tools/testing).

And that’s all! We’ve just built our very first preview version of Actions for Google Assistant. In future publications we’ll see how to extend its functionality.

Thanks for reading! 😊

## Source code

Final source code of described project can be found on [Github](https://github.com/frogermcs/Github-Google-Actions).

### Author 

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com
