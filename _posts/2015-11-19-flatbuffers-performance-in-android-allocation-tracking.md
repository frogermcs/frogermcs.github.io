---
layout: post
title: FlatBuffers performance in Android - allocation tracking
tags: [FlatBuffers, performance, allocation tracking]
---

After my [last post] about parsing JSON in Android with FlatBuffers parser (implemented in C++, connected via NDK to our project) great discussions started about the real performance of FlatBuffers and comparising them to another serialization solutions. You can find those debates on [Reddit] or in comments section on [Jessie Willson's blog].

I won't delve into them but the conclusions are very bright. FlatBuffers are not a silver bullet (like any other solution!), they shouldn't be directly compared, especially with object serializers. But still they are fast. Built on top of `byte[]` array can beat any Object-based solution.

That's why in this post we'll take a look at some performance metrics. While `byte[]` array is lightweight datastore, represented data is not in the final form. It means that somewhere later in our app we'll have to spend some time on objects initialization. It could mean that FlatBuffers are not fast but they just delay allocations. And this is where we start - with memory allocation metrics.

# Intro - Allocation Tracker

Allocation Tracker records an app's memory allocations and lists allocated objects for the profiling cycle with there size, allocating code and call stack. Since [Android Studio 1.3 release] we can use it directly from our IDE to see how allocation looks like in short period of time in our app.

Full instruction how to use it (and description of other memory profilling tools) can be found in official [Allocation Tracker] documentation. 

In very short version, to track your app's memory allocation you have to:

1. Run your app from Android Studio on connected device/emulator.
2. Open Android Monitor panel from the bottom of Android Studio window, on Memory tab.  
![Android Monitor](/images/20/android_monitor.jpg "Android Monitor")

3. You should see real time memory usage chart for your app. When you are ready to measure allocation, just press **Start Allocation Tracking** button on the left bottom corner of this view.  
![Start allocation](/images/20/start_allocation.jpg "Start allocation")

4. When you press again (**Stop Allocation Tracking**), after short while you should see preview of recorded data (`.alloc` file).
5. Now you can customize this view by grouping call stack or showing allocation charts (linear or sunburst).  
![Alloc file](/images/20/alloc_file.jpg "Alloc file")

# Prepare the app

For performance tests we'll use our [FlatBuffs] app which was created in previous posts. We'll add new screen - `ReposListActivity` which will show ListView with parsed repositories. It will use 3 adapters - one for Json parsed objects, two for FlatBuffers objects (optimized and non-optimized version).

![FlatBuffs](/images/20/flatbuffsapp.png "FlatBuffs")

Updated source code is available on Github [repository]. 

# Memory Allocation Tracking

In this post we'll try to measure how many memory allocations happen during some operations made on our JSON and FlatBuffers data. We'll use prepared earlier data: [JSON with repositories list] and its [representation converted to FlatBuffers].

Our files contain 90 entries with repositories and weight: Json - **478kB**, FlatBuffers - **362kB** (**25% less** in this particular case).

## Parsing process

Probably for this process we should prepare another post which will describe it more detailed in all possible combinations. As was mentioned earlier Json parsing shouldn't be compared in direct way with Flatbuffers. Why? 

- Json parsing (via GSON for example) gives us way to convert String to Java objects. But not only this process is time consuming. We should also initialize our parser (configure fields mappers) which also takes time. Do we do it once? Or everytime when we need to convert Json to Java objects via `new Gson().fromJson();`?
- FlatBuffers can be handled in a couple different ways:
	- Pure FlatBuffers format (bytes array comes from API for example). In this case actually there is no parsing. No object is created (except one big `byte[]` array to store whole data stream). It means that instead of hundreds/thousands objects created by Json parser we make only one allocation. But! Pure bytes array in most cases gives us nothing. Somewhere later we have to extract at least part of byte array data and convert it to real Java object. It can be problematic for example in ListViews. We'll see this case later in this post.
	- Json to FlatBuffers parsing. In this case we need to use our NDK FlatBuffers parser which will convert JSON string to FlatBuffers bytes array. In this case objects are created in both Native and Java heap and comparison to pure Java parser won't be obvious.

But hey, from pure curiosity we'll take a look at Allocation Tracker anyway.

### Json parsing

Meaured code:

{% gist frogermcs/ccfeff9b3341a59fa38c ReposListJson.java %} 

String `reposStr` contains [repos_json.json] content.

#### Results

[Allocations_JsonParsing.alloc] *this file can be opened in Android Studio.*

Allocation chart:

![Allocations_JsonParsing](/images/20/Allocations_JsonParsing.png "Allocations_JsonParsing")

Memory allocated in measured time: **2.27M**, total allocations: **51066**.

- AbstractStringBuilder: **60% (1.38M)**, **27360** allocations
- JsonReader: **14% (326K)**, **10025** allocations

It means that **~500kB** JSON allocated about **1.5M** of memory (mostly by transient objects). 

### FlatBuffers parsing

Meaured code:

{% gist frogermcs/ccfeff9b3341a59fa38c ReposListFlat.java %} 

`bytes` contains [repos_json.flat] content.

#### Results

[Allocations_FlatBuffersParsing.alloc] *this file can be opened in Android Studio.*

Allocation chart:

![Allocations_FlatBuffersParsing](/images/20/Allocations_FlatBuffersParsing.png "Allocations_FlatBuffersParsing")

Memory allocated in measured time: **8.56K** (yes, it's **290x** less), total allocations: **251**.

- ByteBuffer: **4.5% (384B)**, **6** allocations
- ReposList: **1** allocation

It means that almost nothing was done to provide FlatBuffers data to our app. But wait, where is our **362K** file? Memory was allocated earlier, in a moment of reading `repos_flat.bin` file from raw resources - in `RawDataReader.loadBytes()` method [(see source)].

## Passing data between Activties via Intent bundle

Test case:

Allocation tracker is started just before creating intent and starting acitvity. Stopped after activity is opened, and ListView is presented.

### Json objects

To make this test possible, models: `RepoListJson`, `RepoJson`, `UserJson` implemented [Parcelable] interface. Code was generated by [ParcelableGenerator] plugin in Android Studio.

Meaured code (+ code called in a moment of transition between Activities) :

{% gist frogermcs/ccfeff9b3341a59fa38c openWithReposJson.java %} 

{% gist frogermcs/ccfeff9b3341a59fa38c setupJsonAdapter.java %} 

#### Results

[Allocations_JsonObject_IntentBundle.alloc] - this file can be opened in Android Studio.

Allocation chart:

![Allocations_JsonObject_IntentBundle](/images/20/Allocations_JsonObject_IntentBundle.png "Allocations_JsonObject_IntentBundle")

Memory allocated in measured time: **454.90K**, total allocations: **7539**.

- Parcel: **66.6% (303.54K)**, **5009** allocations
- frogermcs.io.flatbuffs.model.json.* package (all models): **5.38% (24.56K)**, **452** allocations.

### FlatBuffers objects

Meaured code (+ code called in a moment of transition between Activities) :

{% gist frogermcs/ccfeff9b3341a59fa38c openWithReposFlat.java %} 

{% gist frogermcs/ccfeff9b3341a59fa38c setupEasyFlatAdapter.java %} 

#### Results

[Allocations_FlatBuffers_IntentBundle.alloc] - this file can be opened in Android Studio.

Allocation chart:

![Allocations_FlatBuffers_IntentBundle](/images/20/Allocations_FlatBuffers_IntentBundle.png "Allocations_FlatBuffers_IntentBundle")

Memory allocated in measured time: **491.25K**, total allocations: **2149**.

- Parcel (mainly byte[]): **73.68% (361.94K)**, **1** allocation (it's clearly our FlatBuffers bytes array)
- ReposList: **1** allocation.

While allocated memory size is very similar (**455K** JSON vs **491K** FlatBuffers) we can see much less allocations in FlatBuffers usage. Again we needed only **1** allocation for whole bytes buffer and another **1** for our root object.

## Displaying data in ListView (while scrolling)

This can be the most interesting. As it was mentioned FlatBuffers format moves objects allocation process from parsing to usage time. It can be damaging for our app smoothness, especially when this data has to be continuously processed. But used wisely it doesn't have to be. 

Test case:

`ReposListAcitvity` is opened and we can see list of first repositories. Allocation Tracker measures scrolling process (from first to last element, by 1 fling gesture).

Here is measured `getView()` method which will be called tens of times in scrolling process (it looks simillar on all adapters):

{% gist frogermcs/ccfeff9b3341a59fa38c getView.java %} 

Of course most interesting code for us is:

{% gist frogermcs/ccfeff9b3341a59fa38c getViewHolder.java %} 

### Json objects

Meaured code (`ViewHolder` class):

{% gist frogermcs/ccfeff9b3341a59fa38c RepositoryHolderJson.java %} 

It's pretty straightforward - `name` and `description` strings are just passed to TextViews.

#### Results

[Allocations_JsonObjects_List.alloc] - this file can be opened in Android Studio.

Allocation chart:

![Allocations_JsonObjects_List](/images/20/Allocations_JsonObjects_List.png "Allocations_JsonObjects_List")

Memory allocated in measured time: **32.18K**, total allocations: **523**.

- CharBuffer: **6.43K (19.99%)**, **134** allocations
- JsonRepositoriesListAdapter$RepositoryHolder: **2** allocations

Only those two classes are interesting. The rest of measured objects are common for JSON and FlatBuffers adapters.

### FlatBuffers objects

Meaured code (ViewHolder class):

{% gist frogermcs/ccfeff9b3341a59fa38c RepositoryHolderFlat.java %} 

The same like in JSON ViewHolder, except `name` and `description` are not String fields, but methods.

#### Results

[Allocations_FlatBuffers_List.alloc] - this file can be opened in Android Studio.

Allocation chart:

![Allocations_FlatBuffers_List](/images/20/Allocations_FlatBuffers_List.png "Allocations_FlatBuffers_List")

Memory allocated in measured time: **50.66K**, total allocations: **892**. **Whoa! It's about 40% more in just one fling gesture!**

- CharBuffer: **6.43K (12.70%)**, **134** allocations
- String: **12.48K (15.70%)**, **140** allocations **Those allocations don't occur in JSON objects ListView**
- frogermcs.io.flatbuffs.model.flat.* package (all models): **9.30% (1.33K)**, **83** allocations.

And this is interesting. `CharBuffer` objects allocations number stay unchanged. This is because those objects are created by TextView, in `setText()` method (just dig deeper in Android source code).

But it seems that our adapter created at least **140** new `String` objects and about **80** `Repo` objects which didn't have to be created in JSON ViewHolder. What was happened? 

Just go back to [FlatBuffers documentation] for a while:

*Note that whenever you access a new object like in the pos example above, a new temporary accessor object gets created.*

In short it means that everytime when we call `reposList.repos(position)` new `Repo` object is created. What about strings?

*The default string accessor (e.g. monster.name()) currently always create a new Java String when accessed, since FlatBuffer's UTF-8 strings can't be used in-place by String.*

The same - `repository.name()` and `repository.description()` create new Strings everytime we call for them.

And this is why FlatBuffers can be dangerous for our app smoothness. Everytime when we do scroll (or something what calls `onDraw()` method) new objects are created. Finally it will cause GC event somewhere in scrolling process.

Does it mean that we've just found inexcusable issue? Not at all.

## Don't drop FlatBuffers

*This is a bit hacky way, you do it on your own risk*  😉

One more time let's go back to [FlatBuffers documentation].

*If your code is very performance sensitive (you iterate through a lot of objects), there's a second pos() method to which you can pass a Vec3 object you've already created. This allows you to reuse it across many calls and reduce the amount of object allocation (and thus garbage collection) your program does.*

Reuse it? Sounds familliar? Yeah, this is what `ViewHolder` pattern is used for. 

Let's see at `ReposList.repos()` method. There is `repos(Repo obj, int j)` version which gets already created `Repo` object and fill it with data from `j` position. Our optimized ViewHolder implementation could look like this one:

{% gist frogermcs/ccfeff9b3341a59fa38c RepositoryHolderFlatOptimized.java %}

`Repo` objects are created only in ViewHolder initialization time. Then `bindItemOnPosition()` method reuse them. It means that we've just reduced `Repo` objects allocations from **83** to **2**. 

But still we stayed with **140** Strings allocations. Does FlatBuffers documentation save us this time?

*Alternatively, use monster.nameAsByteBuffer() which returns a ByteBuffer referring to the UTF-8 data in the original ByteBuffer, which is much more efficient. The ByteBuffer's position points to the first character, and its limit to just after the last.*

How it can help us?  
Quick look at [TextView] documentation and here it is - `setText (char[] text, int start, int len)` method which doesn't require String object.

Whole implementation is bit tricky - we have to create `char[]` temporary arrays, fill them with data from FlatBuffer array (which is in `byte` type, so we have to cast it to `char`) and pass to TextView. Instead of describing it step by step here is important source code:

- [OptimizedFlatRepositoriesListAdapter] (see `FlatRepositoryViewHolder`)
- [Repo] - yes, unfortunatelly we had to update generated source code. But why not to ask [FlatBuffers] creators to provide us those methods in generated code in the future? Or fork this code and add it ourself? Look for `/*--- Added for ViewHolder ---*/` comment to see what methods I wrote about (position of first char in string and string length)

#### Results

[Allocations_FlatBuffers_List_Optimized.alloc] - this file can be opened in Android Studio.

Allocation chart:

![Allocations_FlatBuffers_List_Optimized](/images/20/Allocations_FlatBuffers_List_Optimized.png "Allocations_FlatBuffers_List_Optimized")

Memory allocated in measured time: **35.68K**, total allocations: **543**. **It's very close to 523 allocations in JSON ViewHolder**

- CharBuffer: **6.43K (25.78%)**, **140** allocations
- String: **no allocations**
- frogermcs.io.flatbuffs.model.flat.* package: **2** allocations.
- char[]: **12** additional allocations made for our temporary arrays for name and description handling.

And it seems that we did it! FlatBuffers are almost the same effective as JSON parsed object in ListView scrolling. Yes, it's still bit tricky, probably not yet production ready but it works.

## Source code

Full source code of described project is available on Github [repository]. To compile it you have to download Android NDK package.

### Author

[Miroslaw Stanek]  
*Head of Mobile Development* @ [Azimo Money Transfer]

[OptimizedFlatRepositoriesListAdapter]:https://github.com/frogermcs/FlatBuffs/blob/master/app/src/main/java/frogermcs/io/flatbuffs/view/adapter/OptimizedFlatRepositoriesListAdapter.java
[Repo]:https://github.com/frogermcs/FlatBuffs/blob/master/app/src/main/java/frogermcs/io/flatbuffs/model/flat/Repo.java#L41-L54
[(see source)]:https://github.com/frogermcs/FlatBuffs/blob/master/app/src/main/java/frogermcs/io/flatbuffs/utils/RawDataReader.java#L58-L69
[Allocations_FlatBuffersParsing.alloc]:https://github.com/frogermcs/FlatBuffs/blob/master/captures/Allocations_FlatBuffersParsing.alloc
[Allocations_FlatBuffers_IntentBundle.alloc]:https://github.com/frogermcs/FlatBuffs/blob/master/captures/Allocations_FlatBuffers_IntentBundle.alloc
[Allocations_FlatBuffers_List.alloc]:https://github.com/frogermcs/FlatBuffs/blob/master/captures/Allocations_FlatBuffers_List.alloc
[Allocations_FlatBuffers_List_Optimized.alloc]:https://github.com/frogermcs/FlatBuffs/blob/master/captures/Allocations_FlatBuffers_List_Optimized.alloc
[Allocations_JsonObject_IntentBundle.alloc]:https://github.com/frogermcs/FlatBuffs/blob/master/captures/Allocations_JsonObject_IntentBundle.alloc
[Allocations_JsonObjects_List.alloc]:https://github.com/frogermcs/FlatBuffs/blob/master/captures/Allocations_JsonObjects_List.alloc
[Allocations_JsonParsing.alloc]:https://github.com/frogermcs/FlatBuffs/blob/master/captures/Allocations_JsonParsing.alloc
[last post]:http://frogermcs.github.io/json-parsing-with-flatbuffers-in-android/
[Reddit]:https://www.reddit.com/r/androiddev/comments/3rrplu/flatbuffers_arent_fast_theyre_lazy_jesse_wilson/
[TextView]:http://developer.android.com/reference/android/widget/TextView.html
[Jessie Willson's blog]:https://publicobject.com/2015/11/06/flatbuffers-arent-fast-theyre-lazy/
[Android Studio 1.3 release]:http://android-developers.blogspot.com/2015/07/get-your-hands-on-android-studio-13.html
[Allocation Tracker]:http://developer.android.com/tools/performance/allocation-tracker/index.html
[FlatBuffs]:https://github.com/frogermcs/FlatBuffs
[JSON with repositories list]:https://github.com/frogermcs/FlatBuffs/raw/master/app/src/main/res/raw/repos_json.json
[repos_json.json]:https://github.com/frogermcs/FlatBuffs/raw/master/app/src/main/res/raw/repos_json.json
[representation converted to FlatBuffers]:https://github.com/frogermcs/FlatBuffs/raw/master/app/src/main/res/raw/repos_flat.bin
[repos_json.flat]:https://github.com/frogermcs/FlatBuffs/raw/master/app/src/main/res/raw/repos_flat.bin
[Parcelable]:http://developer.android.com/reference/android/os/Parcelable.html
[ParcelableGenerator]:https://github.com/mcharmas/android-parcelable-intellij-plugin
[FlatBuffers documentation]:http://google.github.io/flatbuffers/md__java_usage.html
[FlatBuffers]:http://google.github.io/flatbuffers/
[Miroslaw Stanek]:http://about.me/froger_mcs
[Azimo Money Transfer]:https://azimo.com
[repository]:https://github.com/frogermcs/FlatBuffs
