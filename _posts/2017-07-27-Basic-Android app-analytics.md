---
layout: post
title: Basic Android app analytics in <60min
tags: [Android, Analytics, Data Driven Development]
---

Every big tech company today is **data-driven**. Products are more often built based on collected data rather than internal opinions. It’s very likely that in this moment some of apps on your device are serving you A/B test variants, checking how: new layout, text or even functionality affect your activity and engagement.  
The biggest companies have dedicated Business Intelligence teams, their own data warehouses, custom analytics tools and big flat screens in conference rooms showing realtime charts.  
And the most important — endless audience waiting to be analysed by pie charts or bars 📊.

So if you are member of small mobile-dev team, your app just touched the market or even won’t go out of Android Studio, you may think that data analysis isn’t your problem yet.

You can’t be more wrong. 🙂

There won’t be any analytics people in the future who will join your team just to add some data loggers here and there. Because as usual — everything starts with developers - it depends only on you how quickly you’ll make your app **data-driven**. Believe me, there won’t be easier than now to start collecting the data and in case you are not doing it yet, I have prepared shortlist of propositions how in less than 60 minutes make your app more friendly for data analyst in the future.

## Proper abstraction, basic events

Most of mobile analytics tools are able to track basic data almost automatically. In a few lines of code you will be able to capture installation, app launch events, screen views, crashes and probably even more. Whole implementation usually takes no more than 15 minutes so most of us choose this way as a first step. But in long term perspective it has some flaws:

- It’s not easy to migrate this solution. And there will be many reasons to do this in the future: too big costs of current service, too limited list of features, new data wizard in company has his own toolset and many more.
- Automatically tracked data is not enough. Sooner or later you will need to log more details what in most cases means explicit implementation.
- External SDK is not testable. At some point in the future you will need to guarantee that key metrics are logged properly after every app release.

So how we could make it better?

### Base implementation

At the beginning just make sure that your analytics tools are available “everywhere”. At the beginning you can simply inject them into your `BaseActivity` class:

{% gist frogermcs/f23d071438e98b2383ed44f759505607 BaseActivity.java %}

Now every subclass will always require screen name (`getScreenNameForAnalytics()` method implementation). But in return you are sure that launch event for every new screen will be always logged, no matter of used tools.

{% gist frogermcs/56b5c8a40bdb07661710d1fcba9def53 MainActivity.java %}

And the basic implementation of `AnalyticsTools`. Do you really need to use automatic tracking when this code is so simple?

{% gist frogermcs/a302df15f632019dbecc94243ed9b526 AnalyticsTools.java %}

And that’s it. With just 100 lines of code in two files you are able to log: **screen launches** (automatically), **button clicks** and **custom events** (on demand).  
Now when you decide to replace Mixpanel with Flurry or Google Analytics with your custom BigQuery implementation, you will just need a few amendments in source code. Logged data will remain the same. 
Oh, and If you know **Dependency Injection** concept you probably noticed that `AnalyticsTools` is fully testable with unit tests (you can inject fake GoogleAnalytics and make sure that tracked events are passed to there properly).

### Next steps

It completely depends on you but I would recommend those:

- **DRY_RUN mode** — even as simple boolean flag which makes sure that your data isn’t logged in analytics services (keep logging them in LogCat) when you build your app or test it.

- **Async tracking** — most of analytics services are pretty good in using background threads but some of them still have problems with blocking initialisation. Even 200–500ms can be critical when tools are loaded during app launch. So instead of `CustomAnalyticsSDK` you can inject `Observable<CustomAnalyticsSDK>`. Why and what for? [Check my blog post](https://medium.com/@froger_mcs/async-injection-in-dagger-2-with-rxjava-e7df503343c0).

## Buttons

Logging every button click manually isn’t the most convince way. Even with our implementation you always have to look for `AnalyticsTools` reference in your code. If you build your apps in MVP pattern you will probably end with call chain: presenter —- *calls* -—> activity —- *calls* -—> analytics tools.  
Of course only if you won’t forget to add logging to each newly added button.

Can we automate it somehow? Maybe it’s not the most elegant way but you can always build your `CustomButton`:

{% gist frogermcs/1647cd3cd4430afded8a5388b59ba7c7 CustomButton.java %}

It’s not as simple as `BaseActivity`, but it’s still less than 100 lines of code. Let’s see:

- `AnalyticsTools` are injected to every instance of `CustomButton` (you can use Dagger 2 for this — check posts on this blog for more articles about it).

- In this implementation there are two attributes which will be used in button click event: `CustomButton_analyticsLabel` and `CustomButton_analyticsScreenName`. First one is mandatory and if it’s not set, Line 33 will make sure that app will crash when you open it for the first time. The second one is overridden by current Activity screen name (our `getScreenNameForAnalytics()` method).

- Event is logged whenever user clicks on button. Current implementation logs label and screen name so it’s easy to distinguish “OK” button on main screen and the one on user settings.

To use custom attributes we need to declare them in **res/values/attrs.xml**:

{% gist frogermcs/1647cd3cd4430afded8a5388b59ba7c7 attrs.xml %}

Now all you have to do is to add your CustomButton into **activity.xml** file:

{% gist frogermcs/cf44a491df2c0241ea15c4057fae468e activity.xml %}

Property `app:analyticsLabel` can be both: String or String reference (good practice is to create another file: **res/values/analytics_labels.xml** to not mix labels with real texts in our app).

## Less logic, more data

Soon or later you will need to track custom events in your app (e.g. conversions like sign-up, buy, invite a friend). There is no single way to do this, so let me give you small advice. Stick to one rule: **always try to log as simple events as possible without any logic behind**. Instead of building custom rules around tracked events try to log a bit more data — logic will be added later, during analysis.

So whenever some comes to you asking for: `custom_event_user_signed_up_from_usa` just send: `custom_event_user_signed_up` with additional parameter: `country=”usa”` instead. Otherwise you will end up explaining what are exact conditions for every custom event which is sent from app. Almost every analytics tool is really good in filtering and segmentation so let’s move some responsibility out of you. 🙂

## Privacy

The last hint is for your conscience. Data-driven development is really great approach because the goal is to make user more happy with your app. But keep in mind that there is thin border between analysing and spying. Also please remember that whenever you use external analytics services, you send users data to other companies. So please, make sure that you are not sharing too much of it and respect your customers privacy.  
And yes, you can always analyse anonymised data. Apple calls it [Differential Privacy](https://www.wired.com/2016/06/apples-differential-privacy-collecting-data/).

Thanks for reading! 😊

### Author 

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com
