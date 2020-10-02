**Table of Contents**

1. [Glide Cache Categories](#glide-cache-categories)
2. [Glide Cache Responsibilities](#glide-cache-responsibilities)
3. [Glide Cache Key](#glide-cache-key)
4. [Memory Cache Read](#memory-cache-read)
5. [LRU Algorithm and Weak References](#lru-algorithm-and-weak-references)
6. [Acquired Variable](#acquired-variable)
7. [Memory Cache Write](#memory-cache-write)
8. [Memory Cache Overview](#memory-cache-overview)
9. [Disk Cache](#disk-cache)
10. [ModelLoader](#modelloader)
11. [DataFetcher](#datafetcher)
12. [Encoder](#encoder)
13. [ResourceDecoder](#resourcedecoder)
14. [Engine](#engine)
15. [EngineJob](#enginejob)
16. [DecodeJob](#decodejob)
17. [References](#references)
18. [Contributions](#contributions)

**The internals of the Glide Caching Mechanism** blog post focuses on the source code behind the Glide Cache. This blog acts as an overview of the components involved.

### Glide Cache Categories

Glide cache is divided into memory cache and disk cache. **The memory cache is composed of weak references + LruCache.**

<div class="row  justify-content-center">
  <img class="img-fluid text-center" src = "https://i.imgur.com/2DPPvQV.png">
</div>

<br><br>

**The image resources that Glide needs to cache are divided into two categories:**

**The original picture (Source):** the initial size & resolution of the picture source

**Converted picture (Result):** the picture after size scaling and size compression

When using Glide to load pictures, Glide compresses and converts the pictures according to the View by default, and does not display the original pictures 

**The access to the cache is in the Engine class.**

**Glide.with(this).load(url).into(imageView);**

<br><br>

### Glide Cache Responsibilities

**The main function of memory cache is to prevent applications from repeatedly reading image data into memory.**

**The main function of the disk cache is to prevent applications from repeatedly downloading and reading data from the network or other places.**

<br><br>

### Glide Cache Key

**Caching is to solve the problem of repeated loading,** so there must be a key to distinguish different image resources.

From the code that generates the key below, we can see that the way Glide generates the key is complicated. **There are many parameters that determine the cache key, including the model, signature, width, height, transformations, resourceClass, transcodeClass, options**

**We can assume that for almost any configuration change, it will cause multiple cache keys to be generated for the same picture.** For example: loading the same image into multiple ImageViews of different sizes will generate two cached images. 

<div class="row  justify-content-center">
  <img class="img-fluid text-center" src = "https://i.imgur.com/f26jN9z.png">
</div>

<br><br>

### Memory Cache Read

**Glide enables the memory cache by default.** The following memory caching method can only be used when the memory is turned on.

You can skip the memory cache by doing:

```java

skipMemoryCache(true)

```

**The memory cache is composed of weak references + LruCache.**

The memory cache code is implemented in **Engine#load()** class. This is the same class where Cache Key is generated.

<div class="row  justify-content-center">
  <img class="img-fluid text-center" src = "https://i.imgur.com/RSa6eCF.png">
</div>

<br><br>

**Glide divides the memory cache into two parts: one uses the LruCache algorithm mechanism, the other uses the weak reference mechanism.**

When the memory cache is obtained, it will be cached from the above two areas through two methods

**loadFromCache():** Get the cache from the memory cache using the LruCache algorithm mechanism

<div class="row  justify-content-center">
  <img class="img-fluid text-center" src = "https://i.imgur.com/RGRfC16.png">
</div>

<br><br>

**loadFromActiveResources():** Get the cache from the memory cache using the weak reference mechanism

<div class="row  justify-content-center">
  <img class="img-fluid text-center" src = "https://i.imgur.com/DDeggrZ.png">
</div>

<br><br>

**If the cached picture is not obtained by the above two methods (that is, there is no cache for the picture in the memory cache), a new thread is started to load the picture.**

<br><br>

### LRU Algorithm and Weak References

If you wondering what LRU Algorithm and Weak Reference is:

**The principle of the LruCache algorithm:** Store the most recently used objects in the LinkedHashMap by means of strong references; when the cache is full, remove the least recently used objects from the memory

**Weak references:** Weakly referenced objects have a shorter lifecycle because When the JVM performs garbage collection, once a weakly referenced object is found, it will be recycled (regardless of whether the memory is sufficient)

<br><br>

### Acquired Variable

The pictures that are in use are cached by weak references, and the pictures that are not in use are cached by LruCache. **This is achieved through the picture reference counter (acquired variable). EngineResource# acquire()**

**This acquired variable is used to record the number of times the picture is referenced. Calling the acquire() method will increase the variable by 1, and calling the release() method will decrease the variable by 1.**

<div class="row  justify-content-center">
  <img class="img-fluid text-center" src = "https://i.imgur.com/C5RyTxT.png">
</div>

<br><br>

### Memory Cache Write

The cache write is after the image is loaded. Taking a look at **Engine#load()** method again.

**There are two key objects here, EngineJob and DecodeJob.**

**EngineJob** which maintain a thread pool internally to manage resource loading and notify callbacks when resources are loaded

**DecodeJob** is a class responsible for decoding resources either from cached data or from the original source and applying transformations and transcodes.

<div class="row  justify-content-center">
  <img class="img-fluid text-center" src = "https://i.imgur.com/pEJW20A.png">
</div>

<br><br>

**The weakly referenced cache will be cleaned up when the memory is insufficient, while the memory cache based on LruCache is strongly referenced, so it will not be cleaned up due to memory reasons.**

**LruCache will clear out the least recently used cached data only when the cached data reaches the upper limit of the cache space.**

The implementation mechanisms of the two caches are based on **hash tables**, but LruCache maintains a linked list in addition to the hash table data structure.

The cache key of weak reference type is the same as LruCache, but the value is of weak reference type.

**In addition to being released when the memory is insufficient, the weak reference type cache will also be cleaned up when the engine's resources are released.**

<div class="row  justify-content-center">
  <img class="img-fluid text-center" src = "https://i.imgur.com/0CrM8fl.png">
</div>

<br><br>

The cache based on weak references always exists and cannot be disabled by the user, but the **user can turn off the cache based on LruCache.**

**In essence, weak reference-based caching and LruCache-based caching are for different application scenarios. Weak reference caching is a type of cache, but this kind of cache is more affected by available memory than LruCache.**

<br><br>

### Memory Cache Overview

**When reading the memory cache,** it first reads from the memory cache of the LruCache algorithm mechanism and then reads from the memory cache of the weak reference mechanism.

**When writing to the memory cache,** first write to the memory cache of the weak reference mechanism, and then write to the memory cache of the LruCache algorithm mechanism when the picture is no longer used

<br><br>

### Disk Cache

**Glide 5 disk cache strategy**

**DiskCacheStrategy.DATA -->** Only cache the original picture

**DiskCacheStrategy.RESOURCE -->** Only cache the converted pictures

**DiskCacheStrategy.ALL -->** cache both the original picture and the converted picture. For remote pictures, cache DATA and RESOURCE. For local pictures, only cache RESOURCE

**DiskCacheStrategy.AUTOMATIC -->** Default strategy, try to use the best strategy for local and remote pictures. When downloading network pictures, use DATA. For local pictures, use RESOURCE

**DiskCacheStrategy.NONE -->** Do not cache any content

**Glide cache is divided into weak reference + LruCache + DiskLruCache**

**The order of reading data is weak reference > LruCache > DiskLruCache > network.**

**The order of writing cache is network –> DiskLruCache–> LruCache–> weak reference**

Disk caching is implemented through DiskLruCache, and different types of cached pictures can be obtained according to different caching strategies. 

Its logic is: first fetch from the converted cache. if it is not available, then get the data from the original (unconverted) cache. if that is not possible then load the image data from the network.

<br><br>

### References

[https://github.com/bumptech/glide](https://github.com/bumptech/glide)  

**Relevant Video on "Awesome Dev Notes" YouTube**

<div class="embed-responsive embed-responsive-16by9"><iframe id="player" class="embed-responsive-item" src="https://www.youtube.com/embed/a9tqWZhx280" allowfullscreen="" allow="autoplay"></iframe>

### Contributions

**_For Open-source discussions, android dev questions, and content creation please join [Discord](https://discord.gg/vBnEhuC)_** 

**_Contributions and Pull requests are welcomed at [https://github.com/androiddevnotes](https://github.com/androiddevnotes) repositories!_**

🐣
