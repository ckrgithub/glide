# Glide
初始化
```java
  private static void initializeGlide(@NonNull Context context,@NonNull GlideBuilder buider){
    Context applicationContext=context.getApplicationContext();
    GeneratedAppGlideModule annotationGeneratedModule=getAnnotationGeneratedGlideModules();
    List<GlideModule> manifestModules=Collections.emptyList();
    if(annotationGeneratedModule==null|| annotationGeneratedModule.isManifestParsingEnabled()){
      manifestModules=new ManifestParser(applicationContext).parse();
    }
    if(annotationGeneratedModule!=null&&!annotationGeneratedModule.getExcludedModuleClasses().isEmpty()){
      Set<Class<?>> excludedModuleClasses=annotationGeneratedModule.getExcludeModuleClasses();
      Iterator<GlideModule> iterator=manifestModules.iterator();
      while(iterator.hasNext()){
        GlideModule current=iterator.next();
        if(!excludedModuleClasses.contains(current.getClass())){
          continue;
        }
        if(Log.isLoggable(TAG,Log.DEBUG)){
          Log.d(TAG,"AppGlideModule excludes manifest GlideModule: "+current);
        }
        iterator.remove();
      }
    }
    if(Log.isLoggable(TAG,Log.DEBUG)){
      for(GlideModule glideModule: manifestModules){
        Log.d(TAG,"Discovered GlideModule from manifest: "+glideModule.getClass());
      }
    }
    RequestManagerRetriever.RequestManagerFactory factory=annotationGeneratedModule!=null?annotationGeneratedModule.getRequestManagerFactory():null;
    builder.setRequestManagerFactory(factory);
    for(GlideModule module: manifestModules){
      module.applyOptions(applicationContext,builder);
    }
    if(annotationGeneratedModule!=null){
     annotationGeneratedModule.applyOptions(applicationContext,builder); 
    }
    Glide glide=builder.build(applicationContext);
    for(GlideModule module:manifestModules){
      module.registerComponents(applicationContext,glide,glide.registry);
    }
    if(annotationGeneratedModule!=null){
      annotationGeneratedModule.registerComponents(applicationContext,glide,glide.registry);
    }
    applicationContext.registerComponentCallbacks(glide);
    Glide.glide=glide;
  }
  
  private static GeneratedAppGlideModule getAnnotationGeneratedGlideModules(){
    GeneratedAppGlideModule result=null;
    try{
      
    }catch(ClassNotFoundException e){
      
    }catch(InstantiationException e){
      throwIncorrectGlideModule(e);
    }catch(IllegalAccessException e){
      throwIncorrectGlideModule(e);
    }catch(NoSuchMethodException e){
      throwIncorrectGlideModule(e);
    }catch(InvocationTargetException e){
      throwIncorrectGlideModule(e);
    }
    return result;
  }
  
  private static void throwIncorrectGlideModule(Exception e){
    throw new IllegalStateException("GeneratedAppGlideModuleImpl is implemented incorrectly."
      +"If you are manaully implemented this class,remove your implememtation.The Annotation"
      +"processor will generate a correct implementation",e);
  }
```
Manifest解析
```java
  public final class ManifestParser{
    private static final String TAG="ManifestParser";
    private static final String GLIDE_MODULE_VALUE="GlideModule";
    private final Context context;
    public ManifestParser(Context context){
      this.context=context;
    }
    public List<GlideModule> parse(){
      List<GlideModule> modules =new ArrayList<>();
      try{
        ApplicationInfo appInfo=context.getPackageManager().getApplicationInfo(context.getPackageName(),PackageManager.GET_META_DATA);
        if(appInfo.metaData==null){
          return modules;
        }
        for(String key:appInfo.metaData.keySet()){
          if(GLIDE_MODULE_VALUE.equals(appInfo.metaData.get(key))){
            modules.add(parseModule(key));
          }
        }
      }catch(PackageManager.NameNotFoundException e){
        throw new RuntimeException("Unable to find metadata to parse GlideModules", e);
      }
      return modules;
    }
    private static GlideModule parseModule(String className){
      Class<?> clazz;
      try{
        clazz=Class.forName(className);
      }catch(ClassNotFoundException e){
        throw new IllegalArgumentException("Unable to find GlideModule implementation",e);
      }
      Object module=null;
      try{
        module=clazz.getDeclaredConstructor().newInstance();
      }catch(InstantiationException e){
        throw new RuntimeException("Unable to instantiate GlideModule implementation for "+clazz,e);
      }catch(IllegalAccessException){
        throw new RuntimeException("Unable to instantiate GlideModule implementation for "+clazz,e);
      }catch(NoSuchMethodException e){
        throw new RuntimeException("Unable to instantiate GlideModule implementation for "+clazz,e);
      }catch(InvocationTargetException e){
        throw new RuntimeException("Unable to instantiate GlideModule implementation for "+clazz,e);
      }
      if(!(module instanceof GlideModule)){
        throw new RuntimeException("Expected instanceof GlideModule,but found "+module);
      }
      return (GlideModule)module;
    }
  }
```
## GlideBuilder
```java
  public final class GlideBuilder{
    private final Map<Class<?>,TransitionOptions<?,?>> defaultTransitionOptions=new ArrayMap<>();
    private Engine engine;
    private BitmapPool bitmapPool;
    private ArrayPool arrayPool;
    private MemoryCache memoryCache;
    private GlideExecutor sourceExecutor;
    private GlideExecutor diskCacheExecutor;
    private DiskCache.Factory diskCacheFactory;
    private MemorySizeCalculator memorySizeCalculator;
    private ConnectivityMonitorFactory connectivityMonitorFactory;
    private RequestOptions defaultRequestOptions=new RequestOptions();
    @Nullable
    private RequestManagerFactory requestManagerFactory;
    private GlideExecutor animationExecutor;
    private boolean isActiveResourceRetentionAllowed;
    @Nullable
    private List<RequestListener<Object>> defaultRequestListeners;
    
    @NonNull
    public GlideBuilder setBitmapPool(@Nullable BitmapPool bitmapPool){
      this.bitmapPool=bitmapPool;
      return this;
    }
    @NonNull
    public GlideBuilder setArrayPool(@Nullable ArrayPool arrayPool){
      this.arrayPool=arrayPool;
      return this;
    }
    @NonNull
    public GlideBuilder setMemoryCache(@Nullable MemoryCache memoryCache){
      this.memoryCache=memoryCache;
      return this;
    }
    @NonNull
    public GlideBuilder setDiskCache(@Nullable DiskCache.Factory diskCacheFactory){
      this.diskCacheFactory=diskCacheFactory;
      return this;
    }
    @NonNull
    public GlideBuilder setSourceExecutor(@Nullable GlideExecutor service){
      this.sourceExecutor=service;
      return this;
    }
    @NonNull
    public GlideBuilder setDiskCacheExecutor(@Nullable GlideExecutor service){
      this.diskCacheExecutor =service;
      return this;
    }
    @NonNull
    public GlideBuilder setAnimationExecutor(@Nullable GlideExecutor service){
      this.animationExecutor=service;
      return this;
    }
    @NonNull
    public GlideBuilder setDefaultRequestOptions(@Nullable RequestOptions requestOptions){
      this.defaultRequestOptions=requestOptions;
      return this;
    }
    @NonNull
    public <T> GlideBuilder setDefaultTransitionOptions(@NonNull Class<T> clazz,@Nullable TransitionOptions<?,T> options){
      defaultTransitionOptions.put(clazz,options);
      return this;
    }
    @NonNull
    public GlideBuilder setMemorySizeCalculator(@NonNull MemorySizeCalculator.Builder builder){
      return setMemorySizeCalculator(builder.build());
    }
    @NonNull
    public GlideBuilder setMemorySizeCalculator(@Nullable MemeorySizeCalculator calculator){
      this.memorySizeCalculator = calculator;
      return this;
    }
    @NonNull
    public GlideBuilder setConnectivityMonitorFactory(@Nullable ConnectivityMonitorFactory factory){
      this.connectivityMonitorFactory=factory;
      return this;
    }
    @NonNull
    public GlideBuilder setIsActiveResourceRetentionAllowed(boolean isActiveResourceRetentionAllowed){
      this.isActiveRetentionAllowed=isActiveResourceRetentionAllowed;
      return this;
    }
    @NonNull
    public GlideBuilder addGlobalRequestListener(@NonNull RequestListener<Object> listener){
      if(defaultRequestListeners==null){
        defaultRequestListeners=new ArrayList<>();
      }
      defaultRequestListeners.add(listener);
      return this;
    }
    void setRequestManagerFactory(@Nullable RequestManagerFactory factory){
      this.requestManagerFactory=factory;
    }
    GlideBuilder setEngine(Engine engine){
      this.engine=engine;
      return this;
    }
    @NonNull
    Glide build(){
      if(sourceExecutor==null){
        sourceExecutor=GlideExecutor.newSourceExecutor();
      }
      if(diskCacheExecutor==null){
        diskCacheExecutor=GlideExecutor.newDiskCacheExecutor();
      }
      if(animationExecutor==null){
        animationExecutor=GlideExecutor.newAnimationExecutor();
      }
      if(memorySizeCalculator==null){
        memorySzieCalculator=new MemorySizeCalculator.Builder(context).build();
      }
      if(connectivityMonitorFactory==null){
        connectivityMonitorFactory=new DefaultConnectivityMonitorFactory();
      }
      if(bitmapPool==null){
        int size=memorySizeCalculator.getBitmapPoolSize();
        if(size>0){
          bitmapPool=new LruBitmapPool(size);
        }else{
          bitmapPool=new BitmapPoolAdapter();
        }
      }
      if(arrayPool==null){
        arrayPool=new LruArrayPool(memorySizeCalculator.getArrayPoolSizeInBytes());
      }
      if(memoryCache==null){
        memoryCache=new LruResourceCache(memorySizeCalculator.getMemoryCacheSize());
      }
      if(diskCacheFactory==null){
        diskCacheFactory=new InternalCacheDiskCacheFactory(context);
      }
      if(engine==null){
        engine=new Engine(memoryCache,diskCacheFactory,diskCacheExecutor,sourceExecutor
          ,GlideExecutor.newUnlimitedSourceExecutor(),GlideExecutor.newAnimationExecutor(),isActiveResourceRetentionAllowed);
      }
      if(defaultRequestListeners==null){
        defaultRequestListeners=Collections.emptyList();
      }else{
        defaultRequestListeners=Collections.unmodifiableList(defaultRequestListeners);
      }
      RequestManagerRetriever requestMangerRetriever=new RequestManagerRetriever(requestMangerFactory);
      return new Glide(context,engine,memoryCache,bitmapPool,arrayPool,requestManagerRetriever,connectivityMonitorFactory,
        logLevel,defaultRequestOptions.lock(),defaultTransitionOptions,defaultRequestListeners);
    }
  }
```






























## 缓存策略
* DiskCacheStrategy.NONE: 表示不缓存任何内容
* DiskCacheStrategy.DATA: 表示只缓存原始图片
* DiskCacheStrategy.RESOURCE: 只缓存转换过后的图片
* DiskCacheStrategy.ALL: 即缓存原始图片，也缓存转换过后的图片
* DiskCacheStrategy.AUTOMATIC: 让glide根据图片资源智能选择使用哪种缓存策略



























