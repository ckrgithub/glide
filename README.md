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
### GlideExecutor
* 磁盘缓存线程池：默认构建一个核心线程数和一个最大线程数
* 原数据线程池：默认构建最优核心线程数和最优最大线程数
* 无限制原数据线程池：默认构建0个核心线程数和Integer.MAX_VALUE个最大线程数
* 动画线程池：默认构建0个核心线程数和2个/1个最大线程数

```java
  public final class GlideExecutor implements ExecutorService{
    private static final String DEFAULT_SOURCE_EXECUTOR_NAME="source";
    private static final String DEFAULT_DISK_CACHE_EXECUTOR_NAME="disk-cache";
    private static final int DEFAULT_DISK_CACHE_EXECUTOR_THREADS=1;
    private static final String TAG="GlideExecutor";
    private staic final String SOURCE_UNLIMITED_EXECUTOR_NAME="source-unlimited";
    private static final String ANIMATION_EXECUTOR_NAME="animatioin";
    private static final long KEEP_ALIVE_TIME_MS=TimeUnit.SECONDS.toMillis(10);
    private static final int MAXIMUM_AUTOMATIC_THREAD_COUNT=4;
    private static volatile int bestThreadCount;
    private final ExecutorService delegate;
    
    public static GlideExecutor newDiskCacheExecutor(){
      return newDiskCacheExecutor(DEFAULT_DISK_CACHE_EXECUTOR_THREADS,DEFAULT_DISK_CACHE_EXECUTOR_NAME,UncaughtThrowableStrategy.DEFAULT);
    }
    public static GlideExecutor newDiskCacheExecutor(UncaughtThrowableStrategy uncaughtThrowableStrategy){
      return newDiskCacheExecutor(DEFAULT_DISK_CACHE_EXECUTOR_THREADS,DEFAULT_DISK_CAHCE_EXECUTOR_NAME,uncaughtThrowableStrategy);
    }
    public static GlideExecutor newDiskCacheExecutor(int threadCount,String name,UncaughtThrowableStrategy uncaughtThrowableStrategy){
      return new GlideExecutor(new TheadPoolExecutor(threadCount,threadCount,0,TimeUnit.MILLISECONDS
        ,new PriorityBlockingQueue<Runnable>(),new DefaultThreadFactory(name,uncaughtThrowableStrategy,true)));
    }
    public static GlideExecutor newSourceExecutor(){
      return newSourceExecutor(calculateBestThreadCount(),DEFAULT_SOURCE_EXECUTOR_NAME,UncaughtThrowableStrategy.DEFAULT);
    }
    public static GlideExecutor newSourceExecutor(UncaughtThrowableStrategy uncaughtThrowableStrategy){
      return newSourceExecutor(calculateBestThreadCount(),DEFAULT_SOURCE_EXECUTOR_NAME,uncaughtThrowableStrategy);
    }
    public static GlideExecutor newSourceExecutor(int threadCount,String name,UncaughtThrowableStrategy uncaughtThrowableStrategy){
      return new GlideExecutor(new ThreadPoolExecutor(threadCount,threadCount,0,TimeUnit.MILLISECONDS
        ,new PriorityBlockingQueue<Runnable>(),new DefaultThreadFactory(name,uncaughtThrowableStrategy,false)));
    }
    public static GlideExecutor newUnlimitedSourceExecutor(){
      return new GlideExecutor(new ThreadPoolExecutor(0,Integer.MAX_VALUE,KEEP_ALIVE_TIME_MS,TimeUnit.MILLISECONDS
        ,new SynchronousQueue<Runnable>(),new DefaultThreadFactory(SOURCE_UNLIMITED_EXECUTOR_NAME,UncaughtThrowableStrategy.DEFAULT,false)));
    }
    public static GlideExecutor newAnimationExecutor(){
      int bestThreadCount=calculateBestThreadCount();
      int maximumPoolSize=bestThreadCount>=4?2:1;
      return newAnimationExecutor(maximumPoolSize,UncaughtThrowableStrategy.DEFAULT);
    }
    public static GlideExecutor newAnimationExecutor(int threadCount,UncaughtThrowableStrategy uncaughtThrowableStrategy){
      return new GlideExecutor(new ThreadPoolExecutor(0,threadCount,KEEP_ALIVE_TIME_MS,TimeUnit.MILLISECONDS
        ,new PriorityBlockingQueue<Runnable>(),new DefaultThreadFactory(ANIMATION_EXECUTOR_NAME,uncaughtThrowableStrategy,true)));
    }
    GlideExecutor(ExecutorService delegate){
      this.delegate=delegate;
    }
    @Override
    public void execute(@NonNull Runnable command){
      delegate.execute(command);
    }
    @NonNull
    @Override
    public Future<?> submit(@NonNull Runnable task){
      delegate.submit(task);
    }
    @NonNull
    @Override
    public <T>List<Future<T>> invokeAll(@NonNull Collection<? extends Callable<T>> tasks) throws InterruptedException{
      delegate.invokeAll(tasks);
    }
    @NonNull
    @Override
    public <T> List<Future<T>> invokeAll(@NonNull Collection<? extends Callable<T>> tasks,long timeout,@NonNull TimeUnit unit) throws InterruptedException{
      return delegate.invokeAll(tasks,timeout,unit);
    }
    @NonNull
    @Override
    public <T> T invokeAny(@NonNull Collection<? extends Callable<T>> tasks) throws InterruptedException,ExecutionException{
      return delegate.invokeAny(tasks);
    }
    @Override
    public <T> T invokeAny(@NonNull Collection<? extends Callable<T>> tasks,long timeout,@NonNull TimeUnit unit) throws InterruptedException,ExecutionException,TimeoutException{
    return delegate.invokeAny(tasks,timeout,unit);
    }
    @NonNull
    @Override
    public <T> Fucture<T> submit(@NonNull Runnable task,T result){
      return delegate.submit(task,result);
    }
    @Override
    public <T> Future<T> submit(@NonNull Callable<T> task){
      return delegate.submit(task);
    }
    @Override
    public void shutdown(){
      delegate.shutdown();
    }
    @NonNull
    @Override
    public List<Runnable> shutdownNow(){
      return delegate.shutdownNow();
    }
    @Override
    public boolean isShutdown(){
      return delegate.isShutdown();
    }
    @Override
    public boolean isTerminated(){
      return delegate.isTerminated();
    }
    @Override
    public boolean awaitTermination(long timeout,@NonNull TimeUnit unit) throws InterruptedException{
      return delegate.awaitTermination(timeout,unit);
    }
    @Override
    public String toString(){
      return delegate.toString();
    }
    public static int calculateBestThreadCount(){
      if(bestThreadCount==0){
        bestThreadCount=Max.min(MAXIMUM_AUTOMATIC_THREAD_COUNT,RuntimeCompat.availableProcessors());
      }
      return bestThreadCount;
    }
    public interface UncaughtThrowableStrategy{
      UncaughtThrowableStartegy IGNORE=new UncaughtThrowableStrategy(){
        @Override
        public void handle(Throwable t){
          
        }
      };
      UncaughtThrowableStrategy LOG=new UncaughtThrowableStrategy(){
        @Override
        public void handle(Throwable t){
          if(t!=null&&Log.isLoggable(TAG,Log.ERROR)){
            Log.e(TAG,"Request threw uncaught throwble",t);
          }
        }
      };
      UncaughtThrowableStrategy THROW=new UncaughtThrowableStrategy(){
        @Override
        public void handle(Throwable t){
          if(t!=null){
            throw new RuntimeException("Request threw uncaught throwable",t);
          }
        }
      };
      UncaughtThrowableStrategy DEFAULT=LOG;
      void handle(Throwable t);
    }
    private static final class DefaultThreadFactory implements ThreadFactory{
      private static final int DEFAULT_PRIORITY=android.os.Process.THREAD_PRIORITY_BACKGROUND
        +android.os.Process.THREAD_PRIORITY_MODE_FAVORABLE;
      private final String name;
      @Synthetic final UncaughtThrowableStrategy uncaughtThrowableStrategy;
      @Synthetic final boolean preventNetWorkOpretions;
      private int threadNum;
      DefaultTheadFactory(String name,UncaughtThrowableStrategy uncaughtThrowableStrategy,boolean prevetNetworkOperations){
        this.name=name;
        this.uncaughtThrowableStrategy=uncaughtThrowableStrategy;
        this.preventNetworkOperations=preventNetworkOperations;
      }
      @Override
      public synchronized Thread newThead(@NonNull Runnable runnable){
        final Thread result=new Thead(runnable,"glide-"+name+"-thread-"+threadNum){
          @Override
          public void run(){
            android.os.Process.setThreadPriority(DEFAULT_PRIORITY);
            if(preventNetworkOperations){
              StrictMode.setThreadPolicy(new ThreadPolicy.Builder().detectNetwork().penaltyDeath().build());
            }
            try{
              super.run();
            }catch(){
              uncaughtThrowableStrategy.handle(t);
            }
          }
        };
        threadNum++;
        return result;
      }
    }
  }
```
RuntimeCompat：可用进程数
```java
  final class RuntimeCompat{
    private static final String TAG="GlideRuntimeCompat";
    private static final String CPU_NAME_REGEX="cpu[0-9]+";
    private static final String CPU_LOCATION="/sys/devices/system/cpu/";
    private RuntimeCompat(){
      
    }
    static int availableProcessors(){
      int cpus=Runtime.getRuntime().availableProcessors();
      if(Build.VERSION.SDK_INT <17){
        cpus=Math.max(getCoreCountPre17(),cpus);
      }
      return cpus;
    }
    private static int getCoreCountPre17(){
      File[] cpus=null;
      //重写线程策略，允许磁盘读操作
      //https://github.com/bumptech/glide/issues/1170
      ThreadPolicy originalPolicy = StrictMode.allowThreadDiskReads();
      try{
        File cpuInfo=new File(CPU_LOCATION);
        final Pattern cpuNamePattern = Pattern.compile(CPU_NAME_REGEX);
        cups=cupInfo.listFiles(new FilenameFilter(){
          @Override
          public boolean accept(File file,String s){
            return cpuNamePattern.match(s).matches();
          }
        });
      }catch(Throwable t){
        
      }finally{
        StrictMode.setThreadPolicy(originalPolicy);
      }
      return Math.max(1,cups!=null?cups.length:0);
    }
  }
```
### MemorySizeCalcultor
内存大小计算器
```java
  public final class MemorySizeCalculator{
    private static final String TAG="MemorySizeCalculator";
    static final int BYTES_PER_ARGB_8888_PIXEL=4;
    private static final int LOW_MEMORY_BYTE_ARRAY_POOL_DIVISOR=2;
    private final int bitmapPoolSize;
    private final int memoryCacheSize;
    private final Context context;
    private final int arrayPoolSize;
    interface ScreenDimensions{
      int getWidthPixels();
      int getHeightPixels();
    }
    MemorySizeCalculator(MemorySizeCalculator.Builder builder){
      this.context=builder.context;
      //低端设备，数组池内存分配等于高端设备一半
      arrayPoolSize=isLowMemoryDevice(builder.activityManager)?builder.arrayPoolSizeBytes/LOW_MEMORY_BYTE_ARRAY_POOL_DIVISOR:builder.arrayPoolSizeBytes;
      int maxSize=getMaxSize(builder.activityManager,builder.maxSizeMultiplier,builder.lowMemoryMaxSizeMultiplier);
      int widthPixels=builder.screenDimensions.getWidthPixels();
      int heightPixels=builder.screenDimensions.getHeightPixels();
      int screenSize=widthPixels*heightPixels*BYTES_PER_ARGB_8888_PIXEL;
      int targetBitmapPoolSize=Math.round(screenSize*builder.bitmapPoolScreens);
      int targetMemoryCacheSize=Math.round(screenSize*builder.memoryCacheScreens);
      int availableSize=maxSize-arrayPoolSize;
      if(targetMemoryCacheSize+targetBitmapPoolSize<=availableSize){
        memoryCacheSize=targetMemoryCahceSize;
        bitmapPoolSize=targetBitmapPoolSize;
      }else{
        float part=avialableSize/(builder.bitmapPoolScreens+builder.memoryCacheScreens);
        memoryCachSize=Math.round(part*builder.memoryCacheScreens);
        bitmapPoolSize=Math.round(part*builder.bitmapPoolScreens);
      }
    }
    @TargetApi(Builder.VERSION_CODES.KITKAT)
    static boolean isLowMemoryDevice(ActivityManager activityManager){
      if(Build.VERSION.SDK_INT>=Build.VERSION_CODES.KITKAT){
        return activityManager.isLowRamDevice();
      }else{
        return true;
      }
    }
    private static int getMaxSize(ActivityManager activityManager,float maxSizeMultiplier,float lowMemoryMaxSizeMultiplier){
      final int memroyClassBytes=activityManager.getMemoryClass()*1024*1024;
      final boolean isLowMemoryDevice=isLowMemoryDevice(activityManager);
      return Math.round(memoryClassBytes*(isLowMemoryDevice?lowMemoryMaxSizeMultiplier:maxSizeMultiplier));
    }
    public int getMemoryCacheSize(){
      return memoryCacheSize;
    }
    public int getBitmapPoolSize(){
      return bitmapPoolSize;
    }
    public int getArrayPoolSizeInBytes(){
      return arrayPoolSize;
    }
    privte String toMb(int bytes){
      return Formatter.formatFileSize(context,bytes);
    }
    
    public static final class Builder{
      static final int MEMORY_CACHE_TARGET_SCREENS=2;
      /**
      * On Android O+,we use {@link android.graphics.Bitmap.Config#HEADWARE} for all reasonably sized 
      */
    }
  }
```


























## 缓存策略
* DiskCacheStrategy.NONE: 表示不缓存任何内容
* DiskCacheStrategy.DATA: 表示只缓存原始图片
* DiskCacheStrategy.RESOURCE: 只缓存转换过后的图片
* DiskCacheStrategy.ALL: 即缓存原始图片，也缓存转换过后的图片
* DiskCacheStrategy.AUTOMATIC: 让glide根据图片资源智能选择使用哪种缓存策略




