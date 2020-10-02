**Table of Contents**

1. [One line](#one-line)
2. [Glide](#glide)
3. [GlideBuilder](#glidebuilder)
4. [RequestManagerRetriever](#requestmanagerretriever)
5. [RequestManager](#requestmanager)
6. [RequestBuilder](#requestbuilder)
7. [Request](#request)
8. [Target](#target)
9. [GlideModule](#glidemodule)
10. [ModelLoader](#modelloader)
11. [DataFetcher](#datafetcher)
12. [Encoder](#encoder)
13. [ResourceDecoder](#resourcedecoder)
14. [Engine](#engine)
15. [EngineJob](#enginejob)
16. [DecodeJob](#decodejob)
17. [References](#references)
18. [Contributions](#contributions)

**The internals of the Glide Image Loading Library - Analyzing the Source Code** blog post focuses on the source code behind the one single line of Glide. This blog is in no way exhaustive. Glide library though simple is HUGE behind the scenes. We do not cover the caching mechanism here, coming soon.

### One line

Generally speaking, we use the following code to load an image:

**Glide.with(this).load(url).into(imageView);**

We will explore the source code behind this line.

![](https://i.imgur.com/boFWs3F.png)

<br><br>

### Glide

**Exploring the Source Code:**

Let us take a look at the modules in Glide.

![](https://i.imgur.com/r2q9OEV.png)

<br><br>

Understanding the main outermost classes. First, we have Glide.

Glide is a singleton class, and the instance can be obtained through the **Glide#get(Context)** method.

![](https://i.imgur.com/JV1gDLY.png)

<br><br>

**The Glide class is a global configuration class.** Encoder, Decoder, ModelLoader ... are all set here. It also provides an interface for creating RequestManager (Glide#with() method). We take a look at the RequestManager later.

When using Glide, the **Glide#with()** method is called first to create a RequestManager. The with() method in Glide has five overloads:

**The Glide#with() method delegates the creation of RequestManager to RequestManagerRetriever. RequestManagerRetriever is a singleton class and calls get(Context) to create RequestManager.**

![](https://i.imgur.com/TredNey.png)

<br><br>

### GlideBuilder

GlideBuilder is a class used to create Glide instances, which contains many getters and setter methods, such as setting BitmapPool, MemoryCache, ArrayPool, DiskCache, MemorySizeCalculator, and a few more.

![](https://i.imgur.com/JSqqt1R.png)

<br><br>

These array pools, cache implementations, etc. will eventually be used as **parameters of the Glide constructor to create Glide instances.**

![](https://i.imgur.com/tSTSsUs.png)

<br><br>

### RequestManagerRetriever

[https://www.adobe.com/in/products/aftereffects.html](https://www.adobe.com/in/products/aftereffects.html)  

**Understanding the RequestManagerRetriever**

![](https://i.imgur.com/GJ1AHGL.png)

<br><br>

The 5 overloaded Glide#with() methods mentioned earlier correspond to the 5 overloaded get() methods in RequestManagerRetriever.

**The logic for creating RequestManager is as follows:**

If the parameter of the with method is Activity or Fragment, call the **fragmentGet(Context, android.app.FragmentManager)** method in RequestManagerRetriever to create the RequestManager.

![](https://i.imgur.com/N2a8z3x.png)

<br><br>

If the parameter of the with method is Fragment or FragmentActivity, call **supportFragmentGet(Context, FragmentManager)** method to create RequestManager.

![](https://i.imgur.com/9sRhUQJ.png)

<br><br>

TIf the parameter of the with method is Context, it will determine whether the source belongs to FragmentActivity and Activity. If it is, it will be evaluated according to the above logic, otherwise, the **getApplicationManager(Context)** method will be called to create the RequestManager.

![](https://i.imgur.com/Kn8l41M.png)

<br><br>

**If the background thread calls Glide#with() or the system version is less than 17 i.e JELLY_BEAN_MR1,** the getApplicationManager(Context) method will eventually be called to create the RequestManager.

![](https://i.imgur.com/rivXZ2V.png)

<br><br>

### RequestManager

![](https://i.imgur.com/ctxazZx.png)

<br><br>

**We all know that when using Glide to load a picture if the current screen is destroyed or invisible, it will stop loading the image, but when we use Glide to load the image, we just pass a Context object, so how does Glide get the screen lifecycle through a context object?**

Through the introduction of the RequestManagerRetriever, we know that a FragmentManager parameter is required when creating a RequestManager (except for the global RequestManager), now when creating a RequestManager, an invisible Fragment will be created first, and added to the current screen through FragmentManager, and this invisible Fragment can be used to detect the lifecycle of the screen.

**The code ensures that each Activity/Fragment contains only one RequestManagerFragment and one RequestManager**.

![](https://i.imgur.com/ctxazZx.png)

<br><br>

There are many load methods to create RequestBuilder:

1. A new request builder for downloading content to cache and returning the cache File:

![](https://i.imgur.com/uoGkT38.png)

<br><br>

2. A new request builder for loading a Drawable:

![](https://i.imgur.com/sjzJmY4.png)

<br><br>

### RequestBuilder

RequestBuilder is a generic class used to build requests, such as setting RequestOption, thumbnails, etc.

Many of the overload methods in RequestManager correspond to the overload method in RequestBuilder. **After the load method, the into method is called to set the ImageView or Target. In the into method, the Request is created and started.**

![](https://i.imgur.com/ZusfpFL.png)

<br><br>

### Request

There are three main classes for Request:

**1. SingleRequest**

**2. ThumbnailRequestCoordinator**

**3. ErrorRequestCoordinator**

**SingleRequest** is a class that is responsible for executing the request and reflecting the result to the Target.

![](https://i.imgur.com/B2AozLR.png)

<br><br>

**ThumbnailRequestCoordinators** is a class that is used to coordinate two requests to load the original image and the thumbnail at the same time. The thumbnail does not need to be loaded after the original image is loaded. All these controls are all controlled by this class.

![](https://i.imgur.com/EGzl3Wj.png)

<br><br>

When we fail to load the image, we might want to continue to load another image through the network or other means, for example:

When we use error in this way, it will eventually create an **ErrorRequestCoordinator** object.

![](https://i.imgur.com/0ZYh06b.png)

<br><br>

### Target

**Target represents a resource that can be loaded by Glide and has a lifecycle.**

When we call the **RequestBuilder#into method**, the Target implementation class of the corresponding type will be created based on the incoming parameters.

Glide provides ImageViewTarget on ImageView by default, AppWidgetTarget on AppWidget, FutureTarget for synchronously loading images.

![](https://i.imgur.com/CM9JeWz.gif)

<br><br>

### GlideModule

**GlideModule is an interface allowing lazy configuration of Glide.**

The code of GlideModule is straightforward:

**You can see that the interface is annotated as deprecated. Glide recommends using AppGlideModule instead.**

The GlideModule interface itself has no code content, but it inherits the RegistersComponents and AppliesOptions interfaces.

![](https://i.imgur.com/4AveDyG.png)

<br><br>

### ModelLoader

A factory interface for translating an arbitrarily complex data model into a concrete data type that can be used by a DataFetcher to obtain the data for a resource represented by the model.

**This interface has two objectives:**

1. To translate a specific model into a data type that can be decoded into a resource.

2. To allow a model to be combined with the dimensions of the view to fetch a resource of a specific size.

![](https://i.imgur.com/seJAa7e.png)

<br><br>

### DataFetcher

DataFetcher is an interface.

The internal implementation is to initiate a network request or open a file, or open a resource, etc.

**After the loading is completed, the callback is made through the DataFetcher$DataCallback interface.**

![](https://i.imgur.com/4hi6tro.png)

<br><br>

There are two methods in DataCallback:

They Represent the data load success or load failure callback respectively.

![](https://i.imgur.com/cKcZhhu.png)

<br><br>

### Encoder

**Encoder is an interface for writing data to some persistent data store (i.e. a local File cache).**

There is only one method called encode and the comments explain it quite well.

![](https://i.imgur.com/4EtFJwa.png)

<br><br>

### ResourceDecoder

ResourceDecoder is an interface for decoding resources. It is used to decode the original data into the corresponding data type.

![](https://i.imgur.com/rapvc4e.png)

<br><br>

### Engine

**Engine is a class which is responsible for starting loads and managing active and cached resources.**

Focusing on the load method, this method mainly does the following:

**1. Build Key by request.**

**2. Obtain resources from active resources and return when obtained.**

**3. Get the resource from the cache and return it directly when you get it.**

**4. Judge whether the current request is being executed, if yes, return directly.**

**5. Build EngineJob and DecodeJob and execute.**

![](https://i.imgur.com/lKSicmU.png)

<br><br>

### EngineJob

This is mainly used to execute DecodeJob and manage the callback of loading completion and various listeners.

![](https://i.imgur.com/u195k4v.png)

<br><br>

### DecodeJob

DecodeJob is a class responsible for decoding resources either from cached data or from the original source and applying transformations and transcodes.

The loading of images is ultimately implemented through DataFetcher, but it is not directly called here. DataFetcherGenerator is used here.

![](https://i.imgur.com/NYv3MMF.png)

<br><br>

### References

[https://github.com/bumptech/glide](https://github.com/bumptech/glide)  

**Relevant Video on "Awesome Dev Notes" YouTube**

<div class="embed-responsive embed-responsive-16by9"><iframe id="player" class="embed-responsive-item" src="https://www.youtube.com/embed/3o1kGd708a4" allowfullscreen="" allow="autoplay"></iframe>

### Contributions

**_Contributions and Pull requests are welcomed at [https://github.com/androiddevnotes](https://github.com/androiddevnotes) repositories!_**

🐣
