---
layout: post
title: Building Google Actions with Java
tags: [Google Home, Google Actions, Java, Android, Google Assistant]
---

Voice interfaces are definitely the future of interaction between people and the technology. Even if they won’t replace mobile apps (at least in next years), for sure they extend their possibilities. It means that for many mobile programmers, assistants like [Actions on Google](https://developers.google.com/actions/) or [Amazon Alexa](https://developer.amazon.com/alexa) will be next platforms to build their solutions.


# Google Actions SDK

Currently if we want to build app for Google Home or Google Assistant we have to use Node.js [library](https://developers.google.com/actions/develop/sdk/). But under the hood, Assistant Platform uses JSON-formatted HTTP requests as a communication standard. It means that it’s relatively easy to build SDK in any other language. 

And so I started building unofficial [Google Actions Java SDK](https://github.com/frogermcs/Google-Actions-Java-SDK). If you are like me — an old-fashioned Android developer who builds in Java, there is a big chance that you have a lot of great code powering your apps which can be reused in Assistant Actions. And this was the main reason for this SDK — to enable as much developers as possible to write code for Assistant Platform.


## About project

Since Google Actions Java SDK is on the proof-of-concept stage and still there is a lot of work to do (including documentation and tests), working implementation is all about:

- Handling [RootRequest](https://github.com/frogermcs/Google-Actions-Java-SDK/blob/master/google-actions-java-sdk/src/main/java/com/frogermcs/gactions/api/request/RootRequest.java) objects (incoming requests from Google Assistant)

- Preparing proper [RootResponse](https://github.com/frogermcs/Google-Actions-Java-SDK/blob/master/google-actions-java-sdk/src/main/java/com/frogermcs/gactions/api/response/RootResponse.java) objects and sending them back to Assistant.


The rest are just better code architecture/scalability and utils (e.g. [ResponseBuilder](https://github.com/frogermcs/Google-Actions-Java-SDK/blob/master/google-actions-java-sdk/src/main/java/com/frogermcs/gactions/ResponseBuilder.java) and all others not yet written).

### AppEngine ActionsServlet

As a working example we have simple AppEngine app written in Java which can be used as a replacement for Node.js server, and be easily deployed on Google Cloud. I won’t describe Servlet deployment process - it’s just a mix of these tutorials:

- [https://developers.google.com/actions/develop/sdk/#set_up_your_environment](https://developers.google.com/actions/develop/sdk/#set_up_your_environment) — GActions CLI

- [https://cloud.google.com/sdk/downloads](https://cloud.google.com/sdk/downloads) — Google Cloud SDK

- [https://github.com/GoogleCloudPlatform/java-docs-samples/tree/master/appengine/helloworld-new-plugins](https://github.com/GoogleCloudPlatform/java-docs-samples/tree/master/appengine/helloworld-new-plugins) — Example Gradle config for AppEngine

Our servlet is pretty simple — when user starts interaction (app receives `assistant.intent.action.MAIN` intent after interpreting “Talk to hello action” utterance), Assistant should ask him/her to tell something. Then next utterance (intent: `assistant.intent.action.TEXT`) is echoed by Assistant. And conversation is over.

From code perspective here is the whole flow:

1) [ActionsServlet](https://github.com/frogermcs/Google-Actions-Java-SDK/blob/master/google-actions-java-sample/src/main/java/com/frogermcs/gactions/sample/ActionsServlet.java) receives POST JSON-formatted request, which is parsed to RootRequest object.

2) Thanks to intents mapping, `RequestHandler` passes `RootRequest` to `MainRequestHandler`.

{% gist frogermcs/d6f04f2e62ab617a788665a7d6110d08 AssistantActions_mapping.java %}

3) *Ask* response is generated and passed to `ResponseHandler` which sends response to Assistant.

{% gist frogermcs/c376dff4c05b09e593b3fa594c55a314 MainRequestHandler.java %}

{% gist frogermcs/a1f6e8ec608a1f4078a241d54c1b7812 AppEngineResponseHandler.java %}

4) 1–3 steps are repeated for next utterance which then is echoed to user.

{% gist frogermcs/91031b932cc84fb694ed67e8c67be888 TextRequestHandler.java %}

Here you can see output from [Web Simulator](https://developers.google.com/actions/tools/web-simulator) which additionally can shows the whole debug output for communication between Assistant and our Servlet.

![Web Simulator](/images/30/web_simulator.png "Web Simulator")

## Next steps

Presented example is very simple yet powerful. If you already have any rest client implemented in your Android app (e.g. [Github API](https://developer.github.com/v3/) client) you can put this code to our example Servlet and give user possibility to access this informations via voice interface. As mentioned at the beginning — all you have to do is proper handling for `RootResponse` and `RootRequest`.  

At the end I highly encourage you to contribute in [Google Actions Java SDK](https://github.com/frogermcs/Google-Actions-Java-SDK) development. Let’s enable as many developers as possible to build their brilliant ideas for Google Assistant and Home!

Thanks for reading!

### Author 

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com
