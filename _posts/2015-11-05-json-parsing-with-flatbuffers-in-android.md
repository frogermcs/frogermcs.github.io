---
layout: post
title: JSON parsing with FlatBuffers in Android
tags: [FlatBuffers, JSON, NDK]
---

[FlatBuffers] library is getting more and more popular recently. If you check last [Android Performance Patterns] series, **Colt McAnlis** mentions about it a couple times in different videos. Probably you remember Facebook's [announcement] about migration to FlatBuffers. Also some time ago I published [post] about how to start using it in Android.

In short we should prepare schema, generate java classes and... ask our backend developers to prepare API which returns FlatBuffers format instead of JSON. We proved that this is probably the most efficient way of sending and handling data from API, in our mobile apps.

But what if there is no chance to prepare proper responses (we use public API with returns JSON format like [Github API]). We can try to parse this response on mobile side from JSON string to FlatBuffer binary format. Simple? Not really. According to the [documentation]:

*Text parsing*

*There currently is no support for parsing text (Schema's and JSON) directly from Java or C#, though you could use the C++ parser through native call interfaces available to each language. Please see the C++ documentation for more on text parsing.*

No Java support, C++, native call interfaces... Looks strange.

But it's not. We'll prove in this post that we can setup working project with a minimal knowledge about NDK and C++ programming. We'll build everything on top of [FlatBuffs] project, created in one of my previous posts.

# NDK in Android Studio

## NDK package

First of all we have to download NDK package, which is not included in Android SDK . To do this just visit [NDK Downloads] and get your version. Now we have to unpack this somewhere in your environment (I did it in `~/dev/android-ndk-r10e` on my Mac). 

Then open **FlatBuffs** project in Android Studio and in Project Structure (`File -> Project Structure -> SDK Location`) fill **Android NDK Location** (in my case it's `/Users/froger_mcs/dev/android-ndk-r10e`).

![NDK path](/images/19/ndk.png "NDK path")

Great, thanks to this our project can use NDK code. 

## NDK Android Library module

Let's assume that we don't want to mess in our app code and create external module with C++/JNI code. And thanks to new [gradle-experimental] plugin we can do this with Android Studio support. 

*By the way if you want a good overview of what we can and cannot do with new NDK support in Android Studio there is a great post about it ([ph0b's blog]).*

Let's add new Android Library Module (can be done via Android Studio: `File -> New -> New Module... -> Android Library`).  
I picked **FlatBuffersParser** name and `frogermcs.io.flatbuffersparser` package.

Here is the project structure with with all important directories and files:

{% highlight bash %}
FlatBuffs
├── app
|   ├── src
|   |   └── main
|   |       └── java
|   |
|   └── build.gradle
|
├── flatbuffersparser
|   ├── src
|   |   └── main
|   |       ├── java
|   |       └── jni
|   |
|   └── build.gradle
|
├── build.gradle
|
{% endhighlight %}

## Gradle configuration

Now we need to configure our gradle build system to have both, stable and experimental gradle plugin.

*In case you have any issues with this approach (e.g you want to use custom gradle plugins like Hugo, Fabric or Data Binding and it doesn't work) here is my [question] on StackOverflow with the answer how it can be solved.*

Let's start form `gradle.build`:

{% gist frogermcs/e385aa7376e1076558d7 build.gradle %} 

You can see here the most recent beta and alpha versions of gradle plugins in a moment of writing this post. There is a possibility that when you're reading this, newer versions are available. What is imporant - this particular configuration doesn't produce any issues in build time.

`app/build.gradle` should look like here:

{% gist frogermcs/e385aa7376e1076558d7 appbuild.gradle %}

The only important line is `compile project(':flatbuffersparser')` which means that our **FlatBuffersParser** module will be compiled with our main app.

Now let's see `flatbuffersparser/build.gradle`:

{% gist frogermcs/e385aa7376e1076558d7 libbuild.gradle %}

You can see, that the structure and syntax is bit different than on stable gradle plugin. For more info please check [gradle-experimental] and **Migrating from Traditional Android Gradle Plugin**. `android.ndk` section is just a configuration for NDK code. 

What is important here:

`cppFlags += "-I${file("src/main/jni/flatbufferslib")}".toString()`

This line is a workaround for gradle-experimental which doesn't support native libraries yet. But thanks to that we have access to FlatBuffers [source code] we can embed it in our project and make it statically compile with our code.

Now let's copy some sources from FlatBuffers repository. The simplest way is to download the source code and copy needed files to our project. Let's update our directories tree:

{% highlight bash %}
├── flatbuffersparser
|   └── src
|       └── main
|           ├── java
|           └── jni
|               ├── core (here we will put our native code)
|               └── flatbufferslib (we have to use the same directory name like in android.ndk section)
{% endhighlight %}

We have to copy all FlatBuffers headers from repository and place them in `flatbufferslib` dir. You can find them in `include/flatbuffers`. After this operation our flatbufferslib structure should look like this:

{% highlight bash %}
├── flatbufferslib
|   └── flatbuffers
|       ├── flatbuffers.h
|       ├── hash.h
|       ├── idl.h
|       ├── reflection_generated.h
|       ├── reflection.h
|       └── util.h
{% endhighlight %}

Then we need two more source files used in JSON parsing process: idl_gen_text.cpp and idl_parser.cpp (you can find them in `src` directory in FlatBuffers repo). Put them directly into flatbufferslib directory:

{% highlight bash %}
├── flatbufferslib
|   ├── idl_gen_text.cpp
|   ├── idl_parser.cpp
|   └── flatbuffers
|       ├── ...
|       ...
{% endhighlight %}

## Native source code

Great, now we can write some native code. Let's create two files inside our `jni/core` directory: **main.h** and **main.cpp**. 

**main.h** should have a declaration of JNI method which will be visible from our Java code:

{% gist frogermcs/e385aa7376e1076558d7 main.h %}

If you are not familiar with JNI code here is a couple hints:

- **Java_frogermcs_io_flatbuffersparser_FlatBuffersParser_parseJsonNative** this method name should reflect our Java code. How? In short there is Java keyword, package (with underscores instead of dots), class name and native method name. 

- returned type should be Java object (in our example Java's `byte[]` is `jByteArray` because arrays in Java are also an objects).

- `extern "C"` is required to use C++ methods in JNI calls (removing mangling, see [more] on StackOverflow)

Here is our `FlatBuffersParser` java class which will make native calls:

{% gist frogermcs/e385aa7376e1076558d7 FlatBuffersParser.java %}

And now all we have to do is to implement native JSON parsing process. Fortunately FlatBuffers library has a lot of different samples and tests (btw on your Mac OS you can run them from `build/Xcode/FlatBuffers.xcodeproj`). In this particular case example code can be found in [samples/sample_text.cpp].

Based on this, here is our `main.cpp` class:

{% gist frogermcs/e385aa7376e1076558d7 main.cpp %}

In short this code parses json string and schema string to Java byte[] array.

# Usage

Now all we have to do is to use this FlatBuffers parser in our Java code. We get Java byte[] with parsed data which is compatible with Java FlatBuffer models so implementation is pretty straightforward. We will just add another button button which will call our Parser. Whole source code can be found on FlatBuffs repo ([MainActivity]) class.

## Results

Here is how our app currently looks like. 

![FlatBuffs screenshot](/images/19/app.png "FlatBuffs screenshot")

As you can see parse time isn't as good as while using plain FlatBuffers format. But it's still a bit faster (about 30-40%) than Java parsing with Gson. But it's also worth mentioning than after this process we'll have pure FlatBuffers format which can be cached and/or used in our app. 

On FlatBuffers docs site you can find more [benchmarks] data.

## Source code

Full source code of described project is available on Github [repository]. To compile it you will have download Android NDK package.

### Author

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[post]:http://frogermcs.github.io/flatbuffers-in-android-introdution/
[FlatBuffers]:http://google.github.io/flatbuffers/
[Android Performance Patterns]:https://www.youtube.com/playlist?list=PLWz5rJ2EKKc9CBxr3BVjPTPoDPLdPIFCE
[announcement]:https://code.facebook.com/posts/872547912839369/improving-facebook-s-performance-on-android-with-flatbuffers/
[Github API]:https://developer.github.com/v3/
[documentation]:http://google.github.io/flatbuffers/md__java_usage.html
[FlatBuffs]:https://github.com/frogermcs/FlatBuffs
[NDK Downloads]:http://developer.android.com/ndk/downloads/index.html
[gradle-experimental]:http://tools.android.com/tech-docs/new-build-system/gradle-experimental
[ph0b's blog]:http://ph0b.com/new-android-studio-ndk-support/
[question]:http://stackoverflow.com/questions/33509243/mixing-android-plugins-from-gradle-and-gradle-experimental
[source code]:https://github.com/google/flatbuffers/
[more]:http://stackoverflow.com/questions/1041866/in-c-source-what-is-the-effect-of-extern-c
[samples/sample_text.cpp]:https://github.com/google/flatbuffers/blob/master/samples/sample_text.cpp
[MainActivity]:https://github.com/frogermcs/FlatBuffs/blob/master/app/src/main/java/frogermcs/io/flatbuffs/MainActivity.java#L94-L121
[benchmarks]:http://google.github.io/flatbuffers/md__benchmarks.html
[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com
[repository]:https://github.com/frogermcs/FlatBuffs
