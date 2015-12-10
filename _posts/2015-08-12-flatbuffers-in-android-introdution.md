---
layout: post
title: FlatBuffers in Android - introduction
tags: [FlatBuffers, JSON]
---

JSON - probably everyone knows this lightweight data format used in almost all modern servers. It weights less, is more human-readable and in general is more dev-friendly than old-fashined, horrible xml. JSON is language-independend data format but parsing data and transforming it to e.g. Java objects costs us time and memory resources.  
Several days ago Facebook announced big performance improvement in data handling in its Android app. It was connected with dropping JSON format and replacing it with FlatBuffers in almost entire app. Please check [this article] to get some basic knowledge about FlatBuffers and results of transition to it from JSON.

While the results are very promising, at the first glance the implementation isn't too obvious. Also facebook didn't say too much. That's why in this post I'd like to show how we can start our work with FlatBuffers. 

#FlatBuffers
In short, [FlatBuffers] is a cross-platform serialization library from Google, created specifically for game development and, as Facebook showed, to follow the [16ms rule] of smooth and responsive UI in Android. 

*But hey, before you throw everything to migrate all your data to FlatBuffers, just make sure that you need this. Sometimes the impact on performance will be imperceptible and sometimes [data safety] will be more important than a tens of milliseconds difference in computation speed.*

What makes FlatBuffers so effective? 

- Serialized data is accessed without parsing because of flat binary buffer, even for hierarchical data. Thanks to this we don't need to initialize parsers (what means to build complicated field mappings) and parse this data, which also takes time.
- FlatBuffers data doesn't need to allocate more memory than it's used by buffer itself. We don't need to allocate extra objects for whole hierarchy of parsed data like it's done in JSON.

For the real numbers just check again [facebook article] about migrating to FlatBuffers or [Google documentation] itself.

#Implementation

This article will cover the simplest way of using FlatBuffers in Android app:

- JSON data is converted to FlatBuffer format *somewhere* outside the app (e.g. bin ary file is delivered as a file or returned directly from API)
- Data model (Java classes) is generated by hand, with **flatc** (FlatBuffer compiler)
- There are some limitations for JSON file (null fields cannot be used, Date format is parsed as a String)

Probably in the future we'll prepare more complex solution.

##FlatBuffers compiler

At the beginning we have to get **flatc** - FlatBuffers compiler. It can be built from source code hosted in Google's [flatbuffers repository]. Let's download/clone it. Whole build process is described on [FlatBuffers Building] documentation. If you are Mac user all you have to do is:

1. Open downloaded source code on `\{extract directory}\build\XcodeFlatBuffers.xcodeproj`
2. Run **flatc** scheme (should be selected by default) by pressing **Play** button or `⌘ + R`
3. **flatc** executable will appear in project root directory.

Now we're able to [use schema compiler] which among the others can generate model classes for given schema (in Java, C#, Python, GO and C++) or convert JSON to FlatBuffer binary file.

##Schema file
Now we have to prepare schema file which defines data structures we want to de-/serialize. This schema will be used with flatc to create Java models and to transform JSON into Flatbuffer binary file.

Here is a part of our JSON file:

{% gist frogermcs/804af402e444e76bc012 repos_json_fragment.json %}

Full version is available [here]. It's a bit modified version of data which can be taken from Github API call: [https://api.github.com/users/google/repos](https://api.github.com/users/google/repos).

Writing a FlatBuffer schema is very well [documented], so I won't delve into this. Also in our case schema won't be very complicated. All we have to do is to create 3 tables: `ReposList`, `Repo` and `User`, and define `root_type`. Here is the important part of this schema:

{% gist frogermcs/804af402e444e76bc012 repos_schema_fragment.fbs %}
 
Full schema file is [available here].

## FlatBuffers data file

Great, now all we have to do is to convert `repos_json.json` to FlatBuffers binary file and generate Java models which will be able to represent our data in Java-friendly style (all files required in this operation are [available] in our repository):

```
$ ./flatc -j -b repos_schema.fbs repos_json.json
```

If everything goes well, here is a list of generated files:

- repos_json.bin (we'll rename it to repos_flat.bin)
- Repos/Repo.java
- Repos/ReposList.java
- Repos/User.java

## Android app

Now let's create our example app to check how FlatBuffers format works in practice. Here is the screenshot of it:

![ScreenShot](/images/17/screenshot.png "ScreenShot")

ProgressBar will be used only to show how incorrect data handling (in UI thread) can affect smoothness of user interface.

`app/build.gradle` file of our app will look like this:

{% gist frogermcs/804af402e444e76bc012 build.gradle %}

Of course it's not necessary to use Rx or ButterKnife in our example, but why not to make this app a bit nicer 😉 ?

Let's put repos_flat.bin and repos_json.json files to `res/raw/` directory. 

Here is [RawDataReader] util which helps us to read raw files in Android app. 

At the end put `Repo`, `ReposList` and `User` somewhere in project's source code.

###FlatBuffers library

FlatBuffers provides java library to handle this data format directly in java. Here is [flatbuffers-java-1.2.0-SNAPSHOT.jar] file. If you want to generate it by hand you have to move back to downloaded FlatBuffers source code, go to `java/` directory and use Maven to generate this library:

```
$ mvn install
```

Now put .jar file to your Android project, into `app/libs/` directory.

Great, now all we have to do is to implement `MainActivity` class. Here is the full source code of it:

{% gist frogermcs/804af402e444e76bc012 MainActivity.java %}

Methods which should interest us the most:

- `parseReposListJson(String reposStr)` - this method initializes Gson parser and convert json String to Java objects.
- `loadFlatBuffer(byte[] bytes)` - this method converts bytes (our repos_flat.bin file) to Java objects.

#Results

Now let's visualize differences between JSON and FlatBuffers loading time and consumed resources. Tests are made on Nexus 5 with Android M (beta) installed.

## Loading time

Measured operation is conversion to Java files and iteration over all (90) elements.

JSON        - 200ms (range: 180ms - 250ms) - average loading time of our JSON file (weight: 478kB)
FlatBuffers - 5ms (range: 3ms - 10ms) - average loading time of FlatBuffers binary file (weight: 362kB)

Remember our [16ms rule] ? We're calling those method with a reason in UI thread. Take a look at how our interface would behave in this case:

### JSON loading

![JSON](/images/17/json.gif "JSON")

### FlatBuffer loading

![FlatBuffers](/images/17/flatbuffers.gif "FlatBuffers")

See the difference? Json loading freezes ProgressBar for a while, making our interface unpleasant (operation takes more than 16ms). 

### Allocations, CPU etc.

Whant to measure more? Maybe it's a good time to give a try to [Android Studio 1.3] and new features like Allocation Tracker, Memory Viewer and Method Tracer.

##Source code

Full source code of described project is available on Github [repository]. You don't need to deal with FlatBuffers project - all you need is in `flatbuffers/` directory.

###Author

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[16ms rule]:https://www.youtube.com/watch?v=CaMTIgxCSqU
[data safety]:https://publicobject.com/2014/06/18/im-not-switching-to-flatbuffers/
[FlatBuffers]:https://github.com/google/flatbuffers
[flatbuffers repository]:https://github.com/google/flatbuffers
[FlatBuffers Building]:https://google.github.io/flatbuffers/md__building.html
[flatbuffers-java-1.2.0-SNAPSHOT.jar]:https://github.com/frogermcs/FlatBuffs/blob/master/app/libs/flatbuffers-java-1.2.0-SNAPSHOT.jar
[use schema compiler]:https://google.github.io/flatbuffers/md__compiler.html
[documented]:https://google.github.io/flatbuffers/md__schemas.html
[Android Studio 1.3]:http://android-developers.blogspot.com/2015/07/get-your-hands-on-android-studio-13.html
[here]:https://github.com/frogermcs/FlatBuffs/blob/master/flatbuffers/repos_json.json
[available]:https://github.com/frogermcs/FlatBuffs/tree/master/flatbuffers
[available here]:https://github.com/frogermcs/FlatBuffs/blob/master/flatbuffers/repos_schema.fbs
[facebook article]:https://code.facebook.com/posts/872547912839369/improving-facebook-s-performance-on-android-with-flatbuffers/
[this article]:https://code.facebook.com/posts/872547912839369/improving-facebook-s-performance-on-android-with-flatbuffers/
[Google documentation]:http://google.github.io/flatbuffers/
[RawDataReader]:https://github.com/frogermcs/FlatBuffs/blob/master/app/src/main/java/frogermcs/io/flatbuffs/utils/RawDataReader.java
[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com
[repository]:https://github.com/frogermcs/FlatBuffs