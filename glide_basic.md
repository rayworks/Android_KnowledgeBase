# Glide 源码初探（3.7.0)

## 1 缘起
近期新的项目中用到了广受好评的的图片加载库 [Glide](https://github.com/bumptech/glide)，着实提升了不少开发效率。以文档中最常见的代码为例
```
Glide.with(this).load("http://goo.gl/gEgYUd").into(imageView);
```
一行就能解决图片的自动加载，缓存，显示的问题。

## 2 核心类
* ```Glide``` : 以单例模式提供了一种简单的静态接口，采用 ```RequestBuilder``` 来构建相关的请求。
* ```GlideContext``` : ```Glide```中所有负载（all loads）的全局上下文，包含以及对外提供多样化注册机制，以及加载资源所需的类类型。
* ``` Engine ``` : 负载启动加载以及管理活动的，缓存的资源。
* ```RequestManager``` : 管理以及发起请求（request）。
* ```RequestTracker``` :  跟踪，取消，重启进行中的，已完成的，以及失败的请求。
* ```SingleRequest``` : 为一个```Target```加载一种```Resource```的请求(request)。

## 3 分析
### 3.1 Glide的初始化
加载图片有着不同的使用场景，Glide利用 [Facade](https://en.wikipedia.org/wiki/Facade_pattern) 的模式做出了良好的封装。
![Glide.with(X) 提供的统一接口](http://upload-images.jianshu.io/upload_images/5227029-d974d367934768f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

让我们随即进入任一实现的代码中，最后发现了Glide的初始化逻辑：
```
private static volatile Glide glide;
public static Glide get(Context context) {
    if (glide == null) {
      synchronized (Glide.class) {
        if (glide == null) {
          initGlide(context);
        }
      }
    }
    return glide;
}
```
是的，“未能免俗的” ```Glide```也采用了双重检查锁定(Double-Check Locking)实现了单例模式。显而易见的是，对于整个APP来说图片加载库有一个实例就够了。

### 3.2 加载相关的View dimension 的确定
如其[官方文档](https://github.com/bumptech/glide/wiki/Transformations)所述 :
> "Glide is smart enough to figure out the sizes of Views that use layout weights or match_parent as well as the sizes of those that have concrete dimensions."

它是如何做到的？让我们追踪下本文初始位置给出的**第一行示例代码**的具体调用过程：
```
Glide
.with(Fragment/Activity/View)
 + RequestManager -> LifecycleListener
   + load(model)
     RequestBuilder<Drawable>.load(model)
     + loadGeneric(model)
     + into(Y target) // Y extends Target<TranscodeType>
     + into(ImageView view)
         into (a DrawableImageViewTarget ) // Y target
           buildRequest(target)
             -buildRequestRecursive()
               -obtainRequest()
                 -SingleRequest.obtain
           target.setRequest(request); // set as Tag for ViewTarget
           requestManager.track(target, request);
             requestTracker.runRequest(request);
               request.begin();
                 #SingleRequest.begin()
                     target.getSize(this);
                       // view.addPreDrawListener
                       // callback callled onSizeReady async
                     target.onLoadStarted() -> apply your placeholder
```
原来正在发起图片请求前，有一个获取target size的异步调用。继续来看相关的```ViewTarget```的```getSize()```方法:
```
ViewTarget.getSize(cb)
    sizeDeterminer.getSize(cb);
        ViewTreeObserver observer = view.getViewTreeObserver();
        layoutListener = new SizeDeterminerLayoutListener(this);
        observer.addOnPreDrawListener(layoutListener);
            onPreDraw()
                sizeDeterminer.checkCurrentDimens();
                    // consider padding 
                    int currentWidth = getTargetWidth();
      		    int currentHeight = getTargetHeight();
                    notifyCbs(currentWidth, currentHeight);
cb.onSizeReady(width, height);
```
此处可以发现，由于View自身会经历不同的阶段（measure - layout - draw），在当次调用时，其大小可能有不确定性。加载过程如果在view大小未确定前，会对目标View的PreDraw事件加入监听，以得到最终确定的大小。而这也为后续支持的scale type： center_crop 和 fit_center 做好了铺垫。

### 3.3 对视图层生命周期的智能处理
由于Android View层组件Fragment，Activity 有自身的生命周期，Glide如何封装生命周期的变化，动态管理请求。我们可以从 ```RequestManager``` 创建之初的设置看出一二：

```
private RequestManager supportFragmentGet(Context context, FragmentManager fm,
      Fragment parentHint) {
    SupportRequestManagerFragment current = getSupportRequestManagerFragment(fm, parentHint);
    RequestManager requestManager = current.getRequestManager();
    if (requestManager == null) {
      // TODO(b/27524013): Factor out this Glide.get() call.
      Glide glide = Glide.get(context);
      requestManager =
          factory.build(glide, current.getLifecycle(), current.getRequestManagerTreeNode());
      current.setRequestManager(requestManager);
    }
    return requestManager;
}
```
其中 ```SupportRequestManagerFragment```为UI 无关的Fragment，它被创建并且添加到当前的Activity或是主Fragment之下。它与```RequestManager```建立关联关系并能安全的存储它。
``` requestManager = factory.build(glide, current.getLifecycle(),   current.getRequestManagerTreeNode());```
即将Non-UI Fragment的生命周期关联到了当前的requestManager上。所对应的是，View层上的变化onStart，onStop，onDestroy最终会映射到对request的启动，暂停以及清除。

### 3.4 对于组件扩展的支持
让我们重新关注到Glide初始化时，所做的具体事宜。
```
initGlide()
	// ...
    manifestModules = new ManifestParser(applicationContext).parse();

	GlideBuilder builder = new GlideBuilder()
	        .setRequestManagerFactory(factory);
			
	module.applyOptions(applicationContext, builder);
	annotationGeneratedModule.applyOptions(applicationContext, builder);
	
	glide = builder.build(applicationContext);
	    // initialize executors, memorySizeCalculator, bitmapPool, caches
		
		// Responsible for starting loads and managing active and cached resources
		engine = new Engine()

		requestManagerRetriever = new RequestManagerRetriever(
	        	requestManagerFactory);
		
	    return new Glide(...);
		   // Glide Constructor
		    registry = new Registry();
			registry.register(ByteBuffer.class, new ByteBufferEncoder())
					.register(InputStream.class, new StreamEncoder(arrayPool))
    				/* Bitmaps */
					.append(ByteBuffer.class, Bitmap.class,
						new ByteBufferBitmapDecoder(downsampler))
					.append(InputStream.class, Bitmap.class,
						new StreamBitmapDecoder(downsampler, arrayPool))
					.append(GlideUrl.class, InputStream.class, new HttpGlideUrlLoader.Factory())
					//...
			glideContext = new GlideContext(...)
	
    // eg. OKHttp integration
    // registry.replace(GlideUrl.class, InputStream.class, new OkHttpUrlLoader.Factory());
	for (GlideModule module : manifestModules) {
	      module.applyOptions(applicationContext, builder);
	}
	
	module.registerComponents(applicationContext, glide.registry);
	annotationGeneratedModule.registerComponents(applicationContext, glide.registry);
```
代码中的```manifestModules```包含了从manifest中解析得到的模块，对于模块非空的情况，这里也提供了替换的逻辑：
```
module.registerComponents(applicationContext, glide.registry);
```
以OKHttp为例，如果设置了对应的module，这将会替换Glide默认的网络模块，而采用配置的 OKHttp Client。

## 4 典型的调用栈
```
Glide
.with(Fragment/Activity/View)
 + RequestManager -> LifecycleListener
   + load(model)
     RequestBuilder<Drawable>.load(model)
     + loadGeneric(model)
     + into(Y target) // Y extends Target<TranscodeType>
     + into(ImageView view)
         into (a DrawableImageViewTarget ) // Y target
           buildRequest(target)
             -buildRequestRecursive()
               -obtainRequest()
                 -SingleRequest.obtain
           target.setRequest(request); // set as Tag for ViewTarget
           requestManager.track(target, request);
             requestTracker.runRequest(request);
               request.begin();
                 # SingleRequest.begin()
                     target.getSize(this);
                       // view.addPreDrawListener
                       // callback callled onSizeReady async
                     target.onLoadStarted() -> apply your placeholder
                   
                   onSizeReady()  // req.status = Status.RUNNING;
                     Engine.load()
                       // check the cache 
                       // check the active resources
                       EngineJob<R> engineJob = engineJobFactory.build(...)
                       DecodeJob<R> decodeJob = decodeJobFactory.build(..., engineJob)

                       engineJob.start(decodeJob);
                         decodeJob.run() // with runReason = RunReason.INITIALIZE;
                           runWrapped()
                             // state-machine 
                             // INIT- {RES_CACHE} -{DATA_CACHE}-{SOURCE}|FINISHED
                             getNextGenerator()
                             runGenerators()
                                reschedule() // with Stage.SOURCE // reason = RunReason.SWITCH_TO_SOURCE_SERVICE

                             SourceGenerator.startNext() 
                               loadData = helper.getLoadData().get() // ModelLoader registable ?
                                 modelLoader.buildLoadData
                               loadData.fetcher.loadData(cb)// async
                             SourceGenerator.onDataReady()
                         decodeJob.onDataFetcherReady()
                           decodeFromRetrievedData
                             -decodeFromData(currentFetcher, currentData, currentDataSource);
                                decodeFromFetcher(data, dataSource)
                                 runLoadPath()
                                   LoadPath#Resource<Transcode> load(..., new DecodeCallback(...))
                                     loadWithExceptionList()
                                       DecodePath#decode()
                                         decodeResource()
                                           decodeResourceWithList
                                             decoder.decode(data, width, height, options);
                                       callback.onResourceDecoded
                                 onResourceDecoded()
                                   appliedTransformation.transform()
                             -notifyEncodeAndRelease
                                notifyComplete
                                  callback.onResourceReady(resource, dataSource)
                         engineJob.onResourceReady()
                           MAIN_THREAD_HANDLER.obtainMessage(MSG_COMPLETE, this).sendToTarget();
                         handleResultOnMainThread()
                           cb.onResourceReady(engineResource, dataSource);

                 # SingleRequest.onResourceReady(Resource<?> resource, DataSource dataSource)
                     onResourceReady(Resource<R> resource, R result, DataSource dataSource)
                       if(!requestListener.onResourceReady())
                         target.onResourceReady(result, animation)
                           ImageViewTarget.onResourceReady()
                             setResourceInternal()
                               view.setImageDrawable(resource);
```

## 5 框架小结
* 封装组件的具体实现

* 支持组件的可配置化（registry）

* 对Application view layer的高效运用
