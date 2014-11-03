---
layout: post
title: MultiDex solution for 64k limit in Dalvik.
tags: [android, multidex, dalvik]
---

Almost every Android developer knows sad true - Dalvik, Android's virtual machine used by applications and some system services has one major limit - single .dex file (bytecode interpreted by Dalvik VM) can have **only** 64k (exatly 65536) methods. 

What does it meat (for those who haven't faced with this limit, yet)? I short, if your application contains more methods and you invoke one of these which are placed after 65536 position your appllication will crash with error:

{% highlight bash %}
Unable to execute dex: method ID not in [0, 0xffff]: 65536
Conversion to Dalvik format failed: Unable to execute dex: method ID not in [0, 0xffff]: 65536
{% endhighlight %}

In great article: "[DEX Sky’s the limit? No, 65K methods is]" you can find more details and explanations about this proble.

In here I want to show you how to face this problem once and for all.

64k it's a huge number. Do I really need to care about this limit?
===
Yes, you do. Android grows very fast. Also from development side. Libraries evolve, Google releases new **Play Services** in which every new version has couple hundreds or even thousands methods more then previous one. 

To prove this problem I created small example project. 

Let us suppose that we want to create MVP product of app with:
* Simple REST (json) client
* clean project structure and good programming practises (In case of our app has been successful and we don't want to create it from scratch after that)
* some simple and fancy animations and modern UI elements
* Login by facebook
* Android 4 and 5 compatibility.

Reaching the limit
---
Ok then, let's create simple project in Android Studio with Blank Activity and [#minSdkVersion="15"] (in time of writing this post I use Android Studio ver.0.8.14 and Android Gradle plugin ver.0.13).

Now I'll to add my favorite libraries. Here you have whole dependencies list from our `<project>/app/build.gradle`:

{% highlight bash %}
dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])

    compile 'com.android.support:support-v4:21.+'
    compile 'com.android.support:support-v13:21.+'
    compile 'com.android.support:appcompat-v7:21.0.+'
    compile 'com.android.support:palette-v7:+'
    compile 'com.android.support:recyclerview-v7:+'
    compile 'com.google.android.gms:play-services:6.1.+'
    compile 'com.google.guava:guava:18.0'
    compile 'com.google.code.gson:gson:2.3'

    compile 'com.netflix.rxjava:rxjava-android:0.20.5'

    provided 'com.squareup.dagger:dagger-compiler:1.2.2'
    compile 'com.squareup.retrofit:retrofit:1.7.0'
    compile 'com.squareup.dagger:dagger:1.2.2'
    compile 'com.squareup.picasso:picasso:2.3.4'
    compile 'com.squareup:otto:1.3.5'

    compile 'com.jakewharton:butterknife:5.1.2'
    compile 'com.jakewharton.timber:timber:2.4.0'

    compile 'com.newrelic.agent.android:android-agent:3.+'
    compile('com.crashlytics.sdk.android:crashlytics:2.0.0@aar') {
        transitive = true;
    }

    compile 'se.emilsjolander:stickylistheaders:2.5.1'
    compile 'com.astuetz:pagerslidingtabstrip:1.0.1'
    compile 'com.facebook.rebound:rebound:0.3.6'
}
{% endhighlight %}

What do we have here? In short: 
* Play Services and support libraries 
* Dagger for dependency injection and clean project architecture
* Some tools for fast and fun programming (Guava, Butterknife, Timber)
* Some UI/Animation tools (Rebound - superb animations tools from facebook, RecyclerView, Palette, SlidingTabStrip etc.)
* Retrofit, Gson, Picasso for clean and simple networking
* RxJava for simple and clean asynchronous tasks
* NewRelic and Crashlytics for crash reporting, performance and usage statistics etc.

Here you have [commit] with all dependencies. And also [another one] with facebook SDK.

Great, now let's start counting all used methods (yeah, that's right, 64k limit includes also methods from all dependencies used in project).

First of all build/run project (we have to generate .apk or/and .dex files. You can do this by clicking ▶️ in Android Studio or by running
{% highlight bash %}
$ ./gradlew assembleDebug
{% endhighlight %}
in project root directory.

After that our .apk file should be placed in `<project>/app/builds/outputs/apk/app-debug.apk`.

Now let's analyze it. For this I used [dex-method-counts] tool. After we download it we'll have to run two commands (copied from README):
{% highlight bash %}
$ ./gradlew assemble
$ ./dex-method-counts path/to/App.apk # or .zip or .dex or directory
{% endhighlight %}

Results
---
The result surprised me as well because I've just updated Google Play Services from 5 to 6 and... well, better see yourself:
{% highlight bash %}
Read in 63897 method IDs.
<root>: 63897
    android: 14275
        support: 11454
        	...
    butterknife: 161

    com: 40866
        crashlytics: 631
            ...
        facebook: 221
            ...
        google: 36603
            android: 21839
			...
        newrelic: 2579
            ...
        squareup: 511
            okhttp: 8
            otto: 57
            picasso: 446
            
    io: 1125
        fabric: 1102
            ...
        github: 23
            froger: 23
                hellomultidex: 23
                
    java: 1928

    retrofit: 474

    rx: 3337

    timber: 65
        log: 65
        
Overall method count: 63897
{% endhighlight %}
Yes, it's true. We haven't written any line of code yet but we've almost reached dex limit with our **63897** methods.

Oh, I almost forgot. We haven't attached any Analytics library. So let's add **FlurryAnalytics**. And in case of fast datastore prototyping lets add **Parse SDK**. Just in case.

Let's build it again and... 💥*boom*💥:
{% highlight bash %}
UNEXPECTED TOP-LEVEL EXCEPTION:
com.android.dex.DexIndexOverflowException: method ID not in [0, 0xffff]: 65536
	at com.android.dx.merge.DexMerger$6.updateIndex(DexMerger.java:502)
	at com.android.dx.merge.DexMerger$IdMerger.mergeSorted(DexMerger.java:283)
	at com.android.dx.merge.DexMerger.mergeMethodIds(DexMerger.java:491)
	at com.android.dx.merge.DexMerger.mergeDexes(DexMerger.java:168)
	at com.android.dx.merge.DexMerger.merge(DexMerger.java:189)
	at com.android.dx.command.dexer.Main.mergeLibraryDexBuffers(Main.java:454)
	at com.android.dx.command.dexer.Main.runMonoDex(Main.java:302)
	at com.android.dx.command.dexer.Main.run(Main.java:245)
	at com.android.dx.command.dexer.Main.main(Main.java:214)
	at com.android.dx.command.Main.main(Main.java:106)
{% endhighlight %}
Yes, it happend. And we still haven't written any line of code.

The cure
===
Of course the most correct solution is **ProGuard**. But again - we're working on MVP version of our app and we don't have time to deal with:
{% highlight bash %}
FATAL EXCEPTION: main
    java.lang.NoClassDefFoundError: (...)
{% endhighlight %}
in almost every library which we have in our project.

Fortunatelly, Google has another solution for us. We can split our project into more than one .dex files and load the at runtime. But until now this process wasn't too straightforward and clear (couple years ago Google published [article] about this).

android.support.multidex
---
With Android 5.0 Lollipop release, new support library appeared in Android SDK. It contains only two classes:`MultiDex` and `MultiDexApplication` and simplify whole multidex loading process. According to [MultiDex documentation] it does nothing on Android 5.0 which provide built-in support for secondary dex files. On previous system versions it adds to classloader additional .dex files attached in .apk archive.

Where using MultiDex is pretty simple it's some important things which we should care about. 

###Configuring project
First of all we have to configure build instructions to split our project into multiple dex files. 

In `app/build.gradle` file we have to add:
{% highlight groovy %}
afterEvaluate {
    tasks.matching {
        it.name.startsWith('dex')
    }.each { dx ->
        if (dx.additionalParameters == null) {
            dx.additionalParameters = []
        }
        dx.additionalParameters += '--multi-dex'
        dx.additionalParameters += "--main-dex-list=$projectDir/<filename>".toString()
    }
}
{% endhighlight %}

We have two params:
* `--multi-dex` enables splitting mechanism in build process
* `--main-dex-list` (not required) - file with list of classes which have to be attached in main dex file.

For now we can comment second parameter. We'll back to it later.

After that our app will be splitted to multiple dex files. 

Now we have to merge it (load additional dex files) inside our app. At the beginning we have to attach `android-support-multidex.jar` library into our project. You can find it in:
`.../Android SDK directory/extras/android/support/multidex/library`.
After that we have three ways for loading .dex files in our app:

* Declare MultiDexApplication class in AndroidManifest.xml file:
{% highlight xml %}
<application
    android:icon="@drawable/ic_launcher"
    android:label="@string/app_name"
    android:theme="@style/AppTheme"
    android:name="android.support.multidex.MultiDexApplication">
    ...
</application>
{% endhighlight %}
* Extend MultiDexApplication in our Application class (I picked this way):
{% highlight java %}
public class HelloMultiDexApplication extends MultiDexApplication {
    @Override
    public void onCreate() {
        super.onCreate();
    }
}
{% endhighlight %}
{% highlight xml %}
<application
    android:icon="@drawable/ic_launcher"
    android:label="@string/app_name"
    android:theme="@style/AppTheme"
    android:name=".HelloMultiDexApplication">
    ...
</application>
{% endhighlight %}

* If we can't extend `MultiDexApplication` we can install multiple dex files manually by overriding `attachBaseContext(Context base)` method in our Application class:
{% highlight java %}
public class HelloMultiDexApplication extends Application {
    @Override
    public void onCreate() {
        super.onCreate();
    }

    @Override
    protected void attachBaseContext(Context base) {
        super.attachBaseContext(base);
        MultiDex.install(this);
    }
}
{% endhighlight %}

In general this configuration should be sufficient. But after we build and run our project we get error:
{% highlight bash %}
	UNEXPECTED TOP-LEVEL EXCEPTION:
	com.android.dex.DexException: Library dex files are not supported in multi-dex mode
		at com.android.dx.command.dexer.Main.runMultiDex(Main.java:337)
		at com.android.dx.command.dexer.Main.run(Main.java:243)
		at com.android.dx.command.dexer.Main.main(Main.java:214)
		at com.android.dx.command.Main.main(Main.java:106)
{% endhighlight %}
If you have any additional libraries in your project (in our example we have **Facebook SDK**) we have to be sure that we disabled pre-dexing on them. Unfortunatelly `--multi-dex` option is not complatible with pre-dexed libs.

You can do this by adding:
{% highlight groovy %}
android {
//  ...
    dexOptions {
        preDexLibraries = false
    }
}
{% endhighlight %}
inside your `app/build.gradle` file.

Now after we build and run our project everything works as it should.

Here you have [3rd commit] with complete changes required for enabling MultiDex support.

###Possible problems
It's good to keep in mind these important things:
* Additional .dex files are loaded in `Application.attachBaseContext(Context)` method (by `MultiDex.install(Context)` invokation). It means, that before this moment we can't use classes from them. So i.e. we cannot declare static fields with types attached out of main .dex file. Otherwise we'll get `java.lang.NoClassDefFoundError`. 
* The similar case is with methods in our application class. We have to be sure that that we don't want to access classes and methods loaded from secondary .dex files. But this isse is easy to workaround by moving all invokation to inner, anonymous class. Here you have how it could look like:
{% highlight java %}
public class HelloMultiDexApplication extends MultiDexApplication {

    @Override
    public void onCreate() {
        super.onCreate();
        new Runnable() {
            @Override
            public void run() {
                //Your code
            }
        }.run();
    }
}
{% endhighlight %}
Why it will work this way? In short, because ClassLoader looks for dependencies while class is initialized. So when we're looking for classes and methods inside `Runnable` class we have secondary .dex files loaded.

Another way to deal with problems described above is `--main-dex-list` param described earlier. It points file with list of classes which we want to put in main .dex file.

It's always good use this option in case of `--multi-dex` put `android.support.multidex` package do secondary .dex files and efficiently stop us from merging them. 

Example of described file (named multidex.keep in example project):
{% highlight bash %}
android/support/multidex/BuildConfig/class
android/support/multidex/MultiDex$V14/class
android/support/multidex/MultiDex$V19/class
android/support/multidex/MultiDex$V4/class
android/support/multidex/MultiDex/class
android/support/multidex/MultiDexApplication/class
android/support/multidex/MultiDexExtractor$1/class
android/support/multidex/MultiDexExtractor/class
android/support/multidex/ZipUtil$CentralDirectory/class
android/support/multidex/ZipUtil/class
{% endhighlight %}

##Source code
Ful source code of described example is available on Github [repository].

*Author: [Miroslaw Stanek]*

[DEX Sky’s the limit? No, 65K methods is]:https://medium.com/@rotxed/dex-skys-the-limit-no-65k-methods-is-28e6cb40cf71
[#minSdkVersion="15"]:http://dannyroa.com/2013/10/17/why-its-time-to-support-only-android-4-0-and-above/
[commit]:https://github.com/frogermcs/HelloMultidex/commit/a016ac4d3be532c77d4ae4572d758c1ae0c06311
[another one]:https://github.com/frogermcs/HelloMultidex/commit/fc5fd52ce0ad58211de9aaf7906b6c99f37faa66
[3rd commit]:https://github.com/frogermcs/HelloMultidex/commit/233544b79ec22ac52feac46590e61ab06e0506d2
[dex-method-counts]:https://github.com/mihaip/dex-method-counts
[article]:http://android-developers.blogspot.com/2011/07/custom-class-loading-in-dalvik.html
[MultiDex documentation]:http://developer.android.com/reference/android/support/multidex/MultiDex.html
[repository]:https://github.com/frogermcs/HelloMultidex
[Miroslaw Stanek]:http://about.me/froger_mcs
