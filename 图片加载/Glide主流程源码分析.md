# Glide 主流程源码分析（4.x）

Glide 是Android 端的图片加载框架，

使用的时候需要在代码中加入下面的依赖。一般添加第一个依赖就可以了，如果需要使用 Glide 的注解，那么还要加入第二个来在项目中启用注解处理：

    dependencies {
        compile 'com.github.bumptech.glide:glide:4.8.0'
        annotationProcessor 'com.github.bumptech.glide:compiler:4.8.0'
    }

然后，一个最基本的使用示例：

    Glide.with(fragment).load(myUrl).into(imageView);

接下来我们来分析这个过程究竟发生了什么。

### with()

当调用了 `Glide` 的 `with()` 方法的时候会得到一个 `RequestManager` 实例。`with()` 有多个重载方法，我们可以使用 `Activity` 或者 `Fragment` 等来获取 `Glide` 实例，它们最终都会在一个地方被处理。在后面的文章中，我们会分析这方面的内容。

    public static RequestManager with(Context context) {
        return getRetriever(context).get(context);
    }

在 `with()` 方法内部我们会先使用 `Glide` 的 `get()` 方法获取一个 Glide 实例，并从其中得到一个 `RequestManagerRetriever`，我们的 `RequestManager` 就是从 `RequestManagerRetriever` 中得到的：

    private static RequestManagerRetriever getRetriever(@Nullable Context context) {
        return Glide.get(context).getRequestManagerRetriever();
    }

这里调用了 Glide 的 `get()` 方法，它最终会调用下面的方法实例化一个**单例**的 `Glide` 实例。在下面的方法主要用来从 Manifest 和注解中解析出 `GlideModule` 对象。这个方法中需要出入一个 `GlideBuilder` 实例，它是一个构建者，用来构建 Glide 实例。我们会在调用下面的方法的时候直接使用 `new` 关键字创建一个 `GlideBuilder`。

    private static void initializeGlide(@NonNull Context context, @NonNull GlideBuilder builder) {
        Context applicationContext = context.getApplicationContext();

        // 获取注解生成的 GlideModule
        GeneratedAppGlideModule annotationGeneratedModule = getAnnotationGeneratedGlideModules();
        List<com.bumptech.glide.module.GlideModule> manifestModules = Collections.emptyList();
        // 没有使用注解或者使用了 Manifest 中声明 GlideModule 的话就从 Manifest 中解析
        if (annotationGeneratedModule == null || annotationGeneratedModule.isManifestParsingEnabled()) {
            manifestModules = new ManifestParser(applicationContext).parse();
        }

        // 对从注解中生成的 GlideModule 进行处理
        if (annotationGeneratedModule != null && !annotationGeneratedModule.getExcludedModuleClasses().isEmpty()) {
            Set<Class<?>> excludedModuleClasses = annotationGeneratedModule.getExcludedModuleClasses();
            Iterator<com.bumptech.glide.module.GlideModule> iterator = manifestModules.iterator();
            while (iterator.hasNext()) {
                com.bumptech.glide.module.GlideModule current = iterator.next();
                if (!excludedModuleClasses.contains(current.getClass())) {
                    continue;
                }
                iterator.remove();
            }
        }

        // 对 Manifest 中的 GlideModule 进行处理
        RequestManagerRetriever.RequestManagerFactory factory =
                annotationGeneratedModule != null ? annotationGeneratedModule.getRequestManagerFactory() : null;
        builder.setRequestManagerFactory(factory);
        for (com.bumptech.glide.module.GlideModule module : manifestModules) {
            module.applyOptions(applicationContext, builder);
        }

        if (annotationGeneratedModule != null) {
            annotationGeneratedModule.applyOptions(applicationContext, builder);
        }

        // 使用构建者最终构建出一个 Glide 实例
        Glide glide = builder.build(applicationContext);

        // 注册 GlideModule
        for (com.bumptech.glide.module.GlideModule module : manifestModules) {
            module.registerComponents(applicationContext, glide, glide.registry);
        }
        if (annotationGeneratedModule != null) {
            annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
        }
        applicationContext.registerComponentCallbacks(glide);

        // 赋值给静态的单例实例
        Glide.glide = glide;
    }

### load()

当我们拿到了 `RequestManager` 之后就可以使用它来调用 `load()` 方法了。在我们的实例中传入的是一个 url，所以会调用下面的这个方法：

    public RequestBuilder<Drawable> load(@Nullable String string) {
        return asDrawable().load(string);
    }

另外，`load()` 方法也是重载的，我们可以传入包括 Bitmap, Drawable, Uri 和 String 等在内的多种资源类型。

当我们调用完了上面的方法之后最终会调用下面的方法来获取一个 `RequestBuilder` 对象。在上面的 `asDrawable()` 方法中，我们传入的 `resourceClass` 是 `Drawable`，所以就最终得到了 `RequestBuilder<Drawable>`：

    public <ResourceType> RequestBuilder<ResourceType> as(@NonNull Class<ResourceType> resourceClass) {
        return new RequestBuilder<>(glide, this, resourceClass, context);
    }

然后，我们继续调用 `RequestBuilder<Drawable>` 的 `load(Object)` 方法来继续构建图片加载的请求：

    public RequestBuilder<TranscodeType> load(@Nullable Object model) {
        return loadGeneric(model);
    }

    @NonNull
    private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
        this.model = model;
        isModelSet = true;
        return this;
    }

`load()` 方法里又调用了 `loadGeneric()` 方法。它内部把要加载的图片对应的数据模型赋值给局部变量 `model` 之后就返回了。这样，我们就到达了 `into()` 方法。 

### into()

`into()` 方法也定义在 `RequestBuilder` 内部：

    public ViewTarget<ImageView, TranscodeType> into(@NonNull ImageView view) {
        Util.assertMainThread();
        Preconditions.checkNotNull(view);

        RequestOptions requestOptions = this.requestOptions;
        if (!requestOptions.isTransformationSet() && requestOptions.isTransformationAllowed() && view.getScaleType() != null) {
            switch (view.getScaleType()) {
                case CENTER_CROP:
                    requestOptions = requestOptions.clone().optionalCenterCrop();
                    break;
                // ... 根据图片的 scaleType 属性对请求做调整，类似与上面
            }
        }

        return into(glideContext.buildImageViewTarget(view, transcodeClass), null, requestOptions);
    }

RequestManager 中的 into() 具有多个重载方法，它们最终都会调用到下面的 into() 方法：

    private <Y extends Target<TranscodeType>> Y into(@NonNull Y target,
            RequestListener<TranscodeType> targetListener,
            BaseRequestOptions<?> options) {
        Util.assertMainThread();
        Preconditions.checkNotNull(target);
        if (!isModelSet) {
            throw new IllegalArgumentException("You must call #load() before calling #into()");
        }

        Request request = buildRequest(target, targetListener, options);

        Request previous = target.getRequest();
        if (request.isEquivalentTo(previous) && !isSkipMemoryCacheWithCompletePreviousRequest(options, previous)) {
            request.recycle();
            if (!Preconditions.checkNotNull(previous).isRunning()) {
                previous.begin();
            }
            return target;
        }

        requestManager.clear(target);
        target.setRequest(request);
        requestManager.track(target, request);

        return target;
    }

上面的 `buildRequest()` 最终会调用到下面的方法来构建一个请求对象，
	
    private Request buildRequestRecursive(
            Target<TranscodeType> target,
            @Nullable RequestListener<TranscodeType> targetListener,
            @Nullable RequestCoordinator parentCoordinator,
            TransitionOptions<?, ? super TranscodeType> transitionOptions,
            Priority priority,
            int overrideWidth,
            int overrideHeight,
            BaseRequestOptions<?> requestOptions) {

        // 根据我们是否设置了发生错误时的处理策略来构建一个错误请求
        ErrorRequestCoordinator errorRequestCoordinator = null;
        if (errorBuilder != null) {
            errorRequestCoordinator = new ErrorRequestCoordinator(parentCoordinator);
            parentCoordinator = errorRequestCoordinator;
        }

		// 构建主要的请求
        Request mainRequest = buildThumbnailRequestRecursive(/* 各种参数 */);

        if (errorRequestCoordinator == null) {
            return mainRequest;
        }

        int errorOverrideWidth = errorBuilder.getOverrideWidth();
        int errorOverrideHeight = errorBuilder.getOverrideHeight();
        if (Util.isValidDimensions(overrideWidth, overrideHeight)
                && !errorBuilder.isValidOverride()) {
            errorOverrideWidth = requestOptions.getOverrideWidth();
            errorOverrideHeight = requestOptions.getOverrideHeight();
        }

        Request errorRequest = errorBuilder.buildRequestRecursive(/* 各种参数 */);
        errorRequestCoordinator.setRequests(mainRequest, errorRequest);
        return errorRequestCoordinator;
    }

下面的方法是用来构建主请求的，虽然看上去比较长，但是内部并没有做太多的核心的逻辑，无非就是根据当前是否设置了 `Thumbnail` (缩略图) 来决定如何构建一个请求，但不论哪种方式都会调用到 `obtainRequest()` 方法来完成最终的请求的构建：

    private Request buildThumbnailRequestRecursive(
            Target<TranscodeType> target,
            RequestListener<TranscodeType> targetListener,
            @Nullable RequestCoordinator parentCoordinator,
            TransitionOptions<?, ? super TranscodeType> transitionOptions,
            Priority priority,
            int overrideWidth,
            int overrideHeight,
            BaseRequestOptions<?> requestOptions) {
		// 这里根据我们是否设置了 Thumbnail，分成两种情况来构建请求
        if (thumbnailBuilder != null) {
            TransitionOptions<?, ? super TranscodeType> thumbTransitionOptions = thumbnailBuilder.transitionOptions;

            if (thumbnailBuilder.isDefaultTransitionOptionsSet) {
                thumbTransitionOptions = transitionOptions;
            }

            Priority thumbPriority = thumbnailBuilder.isPrioritySet()
                    ? thumbnailBuilder.getPriority() : getThumbnailPriority(priority);

            int thumbOverrideWidth = thumbnailBuilder.getOverrideWidth();
            int thumbOverrideHeight = thumbnailBuilder.getOverrideHeight();
            if (Util.isValidDimensions(overrideWidth, overrideHeight)
                    && !thumbnailBuilder.isValidOverride()) {
                thumbOverrideWidth = requestOptions.getOverrideWidth();
                thumbOverrideHeight = requestOptions.getOverrideHeight();
            }

            ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
            Request fullRequest = obtainRequest(/* 各种参数 */);
            isThumbnailBuilt = true;

            Request thumbRequest = thumbnailBuilder.buildRequestRecursive(/* 各种参数 */);
            isThumbnailBuilt = false;
            coordinator.setRequests(fullRequest, thumbRequest);
            return coordinator;
        } else if (thumbSizeMultiplier != null) {
            ThumbnailRequestCoordinator coordinator = new ThumbnailRequestCoordinator(parentCoordinator);
            Request fullRequest = obtainRequest(/* 各种参数 */);
            BaseRequestOptions<?> thumbnailOptions = requestOptions.clone().sizeMultiplier(thumbSizeMultiplier);

            Request thumbnailRequest = obtainRequest(/* 各种参数 */);

            coordinator.setRequests(fullRequest, thumbnailRequest);
            return coordinator;
        } else {
		    // 当没有设置过 Thumbnail 的时候就会直接调用下面的这个方法了
            return obtainRequest(/* 各种参数 */);
        }
    }
	
显然下面的方法中使用了 `SingleRequest` 的静态 `obtain()` 方法来构建一个请求。

    private Request obtainRequest(
            Target<TranscodeType> target,
            RequestListener<TranscodeType> targetListener,
            BaseRequestOptions<?> requestOptions,
            RequestCoordinator requestCoordinator,
            TransitionOptions<?, ? super TranscodeType> transitionOptions,
            Priority priority,
            int overrideWidth,
            int overrideHeight) {
        return SingleRequest.obtain(/* 各种参数 */);
    }

而在 `SingleRequest` 的 `obtain()` 方法中会先尝试从请求的池中取出一个请求，当请求不存在的时候就会实例化一个 `SingleRequest`，然后调用它的 `init()` 方法把各种参数赋值进去：

    public static <R> SingleRequest<R> obtain(/* 各种参数 */) {
        SingleRequest<R> request = (SingleRequest<R>) POOL.acquire();
        if (request == null) {
            request = new SingleRequest<>();
        }
        request.init(/* 各种参数 */);
        return request;
    }

这里的 Pool 有两个，一个是用 simple() 静态方法创建的，在创建它的时候内部会 new 出一个 SimplePool. 而所谓的 SimplePool 本质上就是在内部维护了一个数组，这里的 simple() 静态方法返回的是 FactoryPool 实例。FactoryPool 只是对 SimplePool 进行了一层封装。所以，本质上这里的池 POOL 就是一个数组。

    private static final Pools.Pool<SingleRequest<?>> POOL = FactoryPools.simple(150,
            new FactoryPools.Factory<SingleRequest<?>>() {
                @Override
                public SingleRequest<?> create() {
                    return new SingleRequest<Object>();
                }
            });

这样我们分析了请求的构建的过程。那么我们再回到之前的位置来看一下，当请求被构建完成之后又是如何来使用请求来获取图片资源并展示的。

当得到了请求之后会调用 RequestManager 的 track() 方法。在该方法内部又会调用 RequestTracker 的 runRequest() 方法。它首先会把请求加入到一个 Set 中，然后判断Glide当前是不是处理暂停状态，如果没有处于暂停状态就会调用请求的 `begin()` 方法来开始请求：

    requestManager.track(target, request);
	
    void track(@NonNull Target<?> target, @NonNull Request request) {
        targetTracker.track(target);
        requestTracker.runRequest(request);
    }

	public void runRequest(@NonNull Request request) {
        requests.add(request);
        if (!isPaused) {
            request.begin();
        } else {
            request.clear();
            pendingRequests.add(request);
        }
    }

我们这里的请求最终是 SingleRequst，所以，我们可以得到其 begin() 方法如下：

    public void begin() {
        assertNotCallingCallbacks();
        stateVerifier.throwIfRecycled();
        if (model == null) {
            if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
                width = overrideWidth;
                height = overrideHeight;
            }
            int logLevel = getFallbackDrawable() == null ? Log.WARN : Log.DEBUG;
            onLoadFailed(new GlideException("Received null model"), logLevel);
            return;
        }

        if (status == Status.RUNNING) {
            throw new IllegalArgumentException("Cannot restart a running request");
        }

        // 如果我们在完成之后重新启动（通常通过诸如notifyDataSetChanged之类的东西，
        // 在相同的目标或视图中启动相同的请求），我们可以简单地使用我们上次检索的资源和大小
        // 并跳过获取新的大小。这意味着想要重新启动负载因为期望视图大小已更改的用户
        // 需要在开始新加载之前明确清除视图 (View) 或目标 (Target)。
        if (status == Status.COMPLETE) {
            onResourceReady(resource, DataSource.MEMORY_CACHE);
            return;
        }

        status = Status.WAITING_FOR_SIZE;
        if (Util.isValidDimensions(overrideWidth, overrideHeight)) {
            onSizeReady(overrideWidth, overrideHeight);
        } else {
            target.getSize(this);
        }

        if ((status == Status.RUNNING || status == Status.WAITING_FOR_SIZE)
                && canNotifyStatusChanged()) {
            // 使用占位图
            target.onLoadStarted(getPlaceholderDrawable());
        }
    }

当 model 为空时会调用 `onLoadFailed()` 方法，

#### Q：model 何时传入？

    private void onLoadFailed(GlideException e, int maxLogLevel) {
        loadStatus = null;
        status = Status.FAILED;

        isCallingCallbacks = true;
        try {
            boolean anyListenerHandledUpdatingTarget = false;
            if (requestListeners != null) {
                for (RequestListener<R> listener : requestListeners) {
                    anyListenerHandledUpdatingTarget |=
                            listener.onLoadFailed(e, model, target, isFirstReadyResource());
                }
            }
            anyListenerHandledUpdatingTarget |= targetListener != null
                && targetListener.onLoadFailed(e, model, target, isFirstReadyResource());

            if (!anyListenerHandledUpdatingTarget) {
                setErrorPlaceholder();
            }
        } finally {
            isCallingCallbacks = false;
        }

        notifyLoadFailed();
    }

    private void setErrorPlaceholder() {
        if (!canNotifyStatusChanged()) {
            return;
        }

        Drawable error = null;
        if (model == null) {
            error = getFallbackDrawable();
        }
        if (error == null) {
            error = getErrorDrawable();
        }
        if (error == null) {
            error = getPlaceholderDrawable();
        }
        target.onLoadFailed(error);
    }

    public void onLoadFailed(@Nullable Drawable errorDrawable) {
        super.onLoadFailed(errorDrawable);
        setResourceInternal(null);
        setDrawable(errorDrawable);
    }

    // 最终在 ImageViewTarget 中将错误图片的 Drawable 设置到 ImageView 上
    public void setDrawable(Drawable drawable) {
        view.setImageDrawable(drawable);
    }

#### onSizeReady(int, int)





