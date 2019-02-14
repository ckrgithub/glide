# Glide
初始化流程：
* 得到注解生成的GeneratedAppGlideModule
* 解析Manifest配置的并对应GlideModule的meta-data
* 调用GlideModule中的applyOptions函数
* 创建glide对象
* 调用GlideModule中的registerComponents()
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
创建
* 原数据缓存GlideExecutor
* 磁盘缓存GlideExecutor
* 动画GlideExecutor
* 内存大小计算器memorySizeCalculator
* 网络状态监听器DefaultConnectivityMonitorFactory
* 位图池大小，大于0，则创建LruBitmapPool；否则，创建BitmapPoolAdapter

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
内存大小计算器:
* 位图池大小bitmapPoolSize: 4或1张全屏图片大小
* 内存缓存大小memoryCacheSize: 2张全屏图大小
* 数组池大小arrayPoolSize: 低端设备，数组池内存分配等于高端设备一半  2MB
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
      //低端设备，数组池内存分配等于高端设备一半  2MB
      arrayPoolSize=isLowMemoryDevice(builder.activityManager)?builder.arrayPoolSizeBytes/LOW_MEMORY_BYTE_ARRAY_POOL_DIVISOR:builder.arrayPoolSizeBytes;
      //0.4f  0.33f
      int maxSize=getMaxSize(builder.activityManager,builder.maxSizeMultiplier,builder.lowMemoryMaxSizeMultiplier);
      int widthPixels=builder.screenDimensions.getWidthPixels();
      int heightPixels=builder.screenDimensions.getHeightPixels();
      int screenSize=widthPixels*heightPixels*BYTES_PER_ARGB_8888_PIXEL;
      //4:1(4或1张全屏图片大小)
      int targetBitmapPoolSize=Math.round(screenSize*builder.bitmapPoolScreens);
      //2(2张全屏图大小)
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
      * On Android O+,we use {@link android.graphics.Bitmap.Config#HEADWARE} for all reasonably sized images unless we are creating thumbnails for the first time.As a result, the Bitmap pool is much less important on O than it was on previous versions.
      */
      static final int BITMAP_POOL_TARGET_SCREENS=Build.VERSION.SDK_INT<Build.VERSION_CODES.O?4:1;
      static final float MAX_SIZE_MULTIPLIER=0.4f;
      static final float LOW_MEMORY_MAX_SIZE_MULTIPLIER=0.33f;
      //4MB
      static final int ARRAY_POOL_SIZE_BYTES=4*1024*1024;
      final Context context;
      ActivityManager activityManager;
      ScreenDimensions screenDimensions;
      float memoryCacheScreens=MEMORY_CACHE_TARGET_SCREENS;
      float bitmapPoolScreens=BITMAP_POOL_TARGET_SCREENS;
      float maxSizeMultiplier=MAX_SIZE_MULTIPLIER;
      float lowMemoryMaxSizeMultiplier=LOW_MEMORY_MAX_SIZE_MULTIPLIER;
      int arrayPoolSizeBytes=ARRAY_POOL_SIZE_BYTES;
      
      public Builder(Context context){
        this.context=context;
        activityManager=(ActivityManager)context.getSystemService(Context.ACTIVITY_SERVICE);
        screenDimensions=new DisplayMetricsScreenDimensions(context.getResources().getDisplayMetrics());
        //On Android O+ Bitmaps are allocated natively,ART is much more efficient at managing garbage and we rely heavily on HEADWARE Bitmaps,making Bitmap re-use much less important.
        if(Build.VERSION.SDK_INT>=Build.VERSION_CODES.O && isLowMemoryDevice(activityManager)){
          bitmapPoolScreens=0;
        }
        public Builder setMemoryCacheScreens(float memoryCacheScreens){
          Preconditions.checkArgument(memroyCacheScreens>=0,"Memory cache screens must be greater than or equal to 0");
          this.memoryCacheScreens=memoryCacheScreens;
          return this;
        }
        public Builder setBitmapPoolScreens(float bitmapPoolScreens){
          Preconditions.checkArgument(bitmapPoolScreens>=0,"Bitmap pool screens must be greater than or equal to 0");
          this.bitmapPoolScreens=bitmapPoolScreens;
          return this;
        }
        public Builder setMaxSizeMultiplier(float maxSizeMultiplier){
          Preconditions.checkArgument(maxSizeMultiplier>=0&& maxSizeMultiplier<=1,"Size multiplier must be between 0 and 1");
          this.maxSizeMultiplier = maxSizeMultiplier;
          return this;
        }
        public Builder setLowMemoryMaxSizeMultiplier(float lowMemoryMaxSizeMultiplier){
          Preconditions.checkArgument(lowMemoryMaxSizeMultiplier>=0&&lowMemoryMaxSizeMultiplier<=1);
          this.lowMemoryMaxSizeMultiplier=lowMemoryMaxSizeMultiplier;
          return this;
        }
        public Builder setArrayPoolSize(int arrayPoolSizeBytes){
          this.arrayPoolSizeBytes=arrayPoolSizeBytes;
          return this;
        }
        Builder setActivityManager(ActivityManager activityManager){
          this.activityManager=activityManager;
          return this;
        }
        Builder setScreenDimensions(ScreenDimensions screenDimensions){
          this.screenDimensions=screenDimensions;
          return this;
        }
        public MemorySizeCalculator build(){
          return new MemorySizeCalculator(this);
        }
      }
    }
    private static final class DisplayMetricsScreenDimensions implements ScreenDimensions{
      private final DisplayMetrics displayMetrics;
      DisplayMetricsScreenDimensions(DisplayMetrics displayMetrics){
        this.displayMetrics=displayMetrics;
      }
      @Override
      public int getWidthPixels(){
        return displayMetrics.widthPixels;
      }
      @Override
      public int getHeightPixels(){
        return displayMetrics.heightPixels;
      }
    }
  }
```
### DefaultConnectivityMonitorFactory
```java
  public class DefaultConnectivityMonitorFactory implements ConnectivityMonitorFactory{
    private static final String TAG="ConectivityMonitor";
    private static final String METWORK_PERMISSION="android.permission.ACCESS_NETWORK_STATE";
    
    @NonNull
    @Override
    public ConnectivityMonitor build(@NonNull Context context,@NonNull ConnectivityMonitor.ConnectivityListener listener){
      int permissionResult =ContextCompat.checkSelfPermission(context,NETWORK_PERMISSION);
      boolean hasPermission=permissionResult==PackageManager.PERMISSION_GRANTED;
      return hasPermission?new DefaulConnectivityMonitor(context,listener):new NullConnectivityMonitor();
    }
  }
```
### LruBitmapPool
* LruPoolStrategy：默认创建位图大小与位图配置策略SizeConfigStrategy
```java
  public class LruBitmapPool implements BitmapPool{
    private static final String TAG="LruBitmapPool";
    private static final Bitmap.Config.DEFAULT_CONFIG=Bitmap.Config.ARAG_8888;
    private final LruPoolStrategy strategy;
    private final Set<Bitmap.Config> allowedConfigs;
    private final long initialMaxSize;
    private final BitmapTracker tracker;
    private long maxSize;
    private long currentSize;
    private int hits;
    private int misses;
    private int puts;
    privatte int evictions;
    LruBitmapPool(long maxSize,LruPoolStrategy strategy,Set<Bitmap.Config> allowConfigs){
      this.initialMaxSize=maxSize;
      this.maxSize=maxSize;
      this.strategy=strategy;
      this.allowedConfigs=allowedConfigs;
      this.tracker=new NullBitmapTracker();
    }
    public LruBitmapPool(long maxSize){
      this(maxSize,getDefaultStrategy(),getDefaultAllowedConfigs());
    }
    public LruBitmapPool(long maxSize,Set<Bitmap.Config> allowedConfigs){
      this(maxSize,getDefaultStrategy(),allowedConfigs);
    }
    @Override
    public long getMaxSize(){
      return maxSize;
    }
    @Override
    public synchronized void setSizeMultiplier(float sizeMultiplier){
      maxSize=Math.round(initialMaxSize*sizeMultiplier);
      evict();
    }
    @Override
    public synchronized void put(Bitmap bitmap){
      if(bitmap==null){
        throw new NullPointerException("Bitmap must not be null");
      }
      if(bitmap.isRecycled()){
        throw new IllegalStateException("Cannot pool recycled bitmap");
      }
      if(!bitmap.isMutable()||strategy.getSize(bitmap)>maxSize||!allowedConfigs.contains(bitmap.getConfig())){
        bitmap.recycle();
        return;
      }
      final int size=strategy.getSize(bitmap);
      strategy.put(bitmap);
      tracker.add(bitmap);
      puts++;
      currentSize+=size;
      dump();
      evict();
    }
    private void evict(){
      trimToSize(maxSize);
    }
    @Override
    @NonNull
    public Bitmap get(int width,int height,Bitmap.Config config){
      Bitmap result=getDirtyOrNull(width,height,config);
      if(result!=null){
        result.eraseColor(Color.TRANSPARENT);
      }else{
        result=createBitmap(width,height,config);
      }
      return result;
    }
    @NonNull
    @Override
    public Bitmap getDirty(int width,int height,Bitmap.Config config){
      Bitmap result=getDirtyOrNull(width,height,config);
      if(result==null){
        result=createBitmap(width,height,config);
      }
      return result;
    }
    @NonNull
    private static Bitmap createBitmap(int width,int height,@Nullable Bitmap.Config config){
      return Bitmap.createBitmap(width,height,config!=null?config:DEFAULT_CONFIG);
    }
    @TargetApi(Build.VERSION_CODES.O)
    private static void assertNotHardwareConfig(Bitmap.Config config){
      if(Build.VERISON.SDK_INT<Build.VERSION_CODES.O){
        return;
      }
      if(config==Bitmap.Config.HARDWARE){
        throw new IllegalArgumentException("Connot create a mutalbe Bitmap with config: "+config+". Consider setting Downsample#ALLOW_HARDWARE_CONFIG to false in your RequestOptions and/or in GlideBuilder.setDefaultRequestOptions");
      }
    }
    @Nullable
    private synchronized Bitmap getDirtyOrNull(int width,int height,@Nullable Bitmap.Config config){
      assertNotHardwareConfig(config);
      final Bitmap result=strategy.get(width,height,config!=null?config:DEFAULT_CONFIG);
      if(result==null){
        misses++;
      }else{
        hits++;
        currentSize-=strategy.getSize(result);
        tracker.remove(result);
        normalize(result);
      }
      dump();
      return result;
    }
    private static void normalize(Bitmap bitmap){
      bitmap.setHasAlpha(true);
      maybeSetPreMultiplied(bitmap);
    }
    @TargetApi(Build.VERSION_CODES.KITKAT)
    private static void maybeSetPreMultiplied(Bitmap bitmap){
      if(Build.VERSION_CODES.SDK_INT>=Build.VERSION_CODES.KITKAT){
        bitmap.setPremultiplied(true);
      }
    }
    @Override
    public void clearMemory(){
      trimToSize(0);
    }
    @Override
    public void trimMemory(int level){
      if(level>=android.content.ComponentCallbacks2.TRIM_MEMORY_BACKGROUND){
        clearMemory();
      }else if(level>=android.content.ComponentCallback2.TRIM_MEMORY_UI_HIDDEN||level==android.content.ComponentCallback2.TRIM_MEMORY_RUNNING_CRITICAL){
        trimToSize(getMaxSize/2);
      }
    }
    private synchronized void trimToSize(long size){
      while(currentSize>size){
        final Bitmap removed=strategy.removeLast();
        if(removed==null){
          currentSize=0;
          return
        }
        tracker.remove(removed);
        currentSize-=strategy.getSize(removed);
        evictions++;
        dump();
        removed.recycle();
      }
    }
    private void dump(){
      if(Log.isLoggable(TAG,Log.VERBOSE)){
        dumpUnchecked();
      }
    }
    private void dumpUnchecked(){
      Log.v(TAG,"Hits="+hits+",misses="+misses+",puts="+puts+",evictions="+evictions+",currentSize="+currentSize+",maxSize="+masSize+"\nStrategy="+strategy);
    }
    
    private static LurPoolStrategy getDefaultStrategy(){
      final LruPoolStrategy strategy;
      if(Build.VERSION.SDK_INT>=Build.VERSION_CODES.KITKAT){
        strategy=new SizeConfigStrategy();
      }else{
        strategy=new AttributeStrategy();
      }
      return strategy;
    }
    @TargetApi(Build.VERSION_CODES.O)
    private static Set<Bitmap.Config> getDefaultAllowedConfigs(){
      Set<Bitmap.Config> configs=new HashSet<>(Arrays.asList(Bitmap.Config.values()));
      if(Build.VERSION.SDK_INT>=Build.VERSION_CODES.KITKAT){
        //GIFs 占坑
        configs.add(null);
      }
      if(Build.VERSION.SDK_INT>=Build.VERSION_CODES.O){
        configs.remove(Bitmap.Config.HARDWARE);
      }
      return Collections.unmodifiableSet(configs);
    }
    private interface BitmapTracker{
      void add(Bitmap bitmap);
      void remove(Bitmap bitmap);
    }
    private static class ThrowingBitmapTracker implements BitmapTracker{
      private final Set<Bitmap> bitmaps=Collections.synchronizedSet(new HashSet<Bitmap>());
      @Override
      public void add(Bitmap bitmap){
        if(bitmaps.contains(bitmap)){
          throw new IllegalStateException("Cannot add already added bitmap:"+bitmap+" ["+bitmap.getWidth()+"x"+bitmap.getHeight()+"]");
        }
        bitmaps.add(bitmap);
      }
      @Override
      public void remove(Bitmap bitmap){
        if(!bitmaps.contains(bitmap)){
          throw new IllegalStateException("Connot remove bitmap not in tracker");
        }
        bitmaps.remove(bitmap);
      }
    }
    private static final class NullBitmapTracker implements BitmapTracker{
      NullBitmapTracker(){}
      @Override
      public void add(Bitmap bitmap){}
      @Override
      public void remove(Bitmap bitmap){}
    }
  }
```
SizeConfigStrategy:重用bitmap字节大小的位图，提高了位图池命中率
* RGBA_F16_IN_CONFIGS数组:{Bitmap.Config.ARGB_8888,null,Bitmap.Config.RGBA_F16}
* ARGB_8888_IN_CONFIGS数组:{Bitmap.Config.ARGB_8888,null}
* RGB_565_IN_CONFIGS数组:{Bitmap.Config.RGB_565}
* ARGB_4444_IN_CONFIG数组:{Bitmap.Config.ARGB_4444}
* ALPHA_8_IN_CONFIG数组:{Bitmap.Config.ALPHA_8}  
> Bitmap.Config:描述像素如何存储，将影响质量(颜色深度)和透明/半透明颜色。
> ALPHA_8:每个像素都存储为一个半透明通道，不存储颜色信息。在这种配置下，每个像素需要一个字节的内存
> RGB_565:每个像素存储为2字节，只有rgb通道被编码：红色存储为5位精度(32个可能值)，绿色存储为6位精度(64个可能值)，蓝色存储为5位精度。当使用不需要高颜色保真度的不透明位图时，这种配置可能很有用
> ARGB_4444:因为此配置质量较差，推荐使用ARGB_8888
> ARGB_8888:每个像素被存储为4个字节。每个通道存储为8位(256个可能值)
> RGBA_F16:每个像素被存储为8个字节。每个通道存储为高精度的浮点值。这个配置适合宽色域和HDR内容
> HEADWARE:位图仅存储在图形内存中的特殊配置。这个配置的位图不可变，在屏幕上绘制是最优的
* 查找最优Key:
```java
  @RequiresApi(Build.VERSION_CODES.KITKAT)
  public class SizeConfigStrategy implements LruPoolStrategy{
    private static final int MAX_SIZE_MULTIPLE=8;
    private static fianl Bitmap.Config[] ARGB_8888_IN_CONFIGS;
    static{
      Bitmap.Config[] result=new Bitmap.Config[]{
        Bitmap.Config.ARGB_8888,
        null,
      };
      if(Build.VERSION.SDK_INT>=Build.VERSION_CODES.O){
        result=Arrays.copyOf(result,result.length+1);
      }
      ARGB_8888_IN_CONFIGS=result;
    }
    private static final Bitmap.Config[] RGBA_F16_IN_CONFIGS=ARGB_8888_IN_CONFIGS;
    private static final Bitmap.COnfig[] RGB_565_IN_CONFIGS=new Bitmap.Config[]{Bitmap.Config.RGB_565};
    private static final Bitmap.Config[] ARGB_4444_IN_CONFIGS=new Bitmap.Config[]{Bitmap.Config.ARGB_4444};
    private static final Bitmap.Config[] ALPHA_8_IN_CONFIGS=new Bitmap.Config[]{Bitmap.Config.ALPHA_8};
    
    private final KeyPool keyPool=new KeyPool();
    //类似：LinkedHashMap,
    private final GroupedLinkedMap<Key,Bitmap> groupedMap=new GroupedLinkedMap<>();
    private final Map<Bitmap.Config,NavigableMap<Integer,Integer>> sortedSizes=new HashMap<>();
    
    @Override
    public void put(Bitmap bitmap){
      int size=Util.getBitmapByteSize(bitmap);
      Key key=keyPool.get(size,bitmap.getConfig());
      groupedMap.put(key,bitmap);
      NavigalbeMap<Integer,Integer> sizes=getSizesForConfig(bitmap.getConfig());
      Integer current=sizes.get(key.size);
      sizes.put(key.size,current==null?1:current+1);
    }
    @Override
    @Nullable
    public Bitmap get(int width,int height,Bitmap.Config config){
      int size=Util.getBitmapByteSize(width,height,config);
      Key bestKey=findBestKey(size,config);
      Bitmap result=groupedMap.get(bestKey);
      if(result!=null){
        decrementBitmapOfSize(bestKey.size,result);
        result.reconfigure(width,height,config);
      }
      return result;
    }
    private Key findBestKey(int size,Bitmap.Config config){
      Key result=keyPool.get(size,config);
      for(Bitmap.Config possibleConfig:getInConfigs(config)){
        NavigableMap<Integer,Integer> sizesForPossibleConfig=getSizesForConfig(possibleConfig);
        Integer possibleSize=sizesForPossibleConfig.ceilingKey(size);
        if(possibleSize!=null&&possibleSize<=size*MAX_SIZE_MULTIPLE){
          if(possibleSize!=size||(possibleConfig==null?config!=null:!possibleConfig.equals(config))){
            keyPool.offer(result);
            result=keyPool.get(possibleSize,possibleConfig);
          }
          break;
        }
      }
      return result;
    }
    @Override
    @Nullable
    public Bitmap removeLast(){
      Bitmap removed=groupedMap.removeLast();
      if(removed!=null){
        int removedSize=Util.getBitmapByteSize(removed);
        decrementBitmapOfSize(removedSize,removed);
      }
      return removed;
    }
    private void decrementBitmapOfSize(Integer size,Bitmap removed){
      Bitmap.Config config=removed.getConfig();
      NavigableMap<Integer,Integer> sizes=getSizesForConfig(config);
      Integer current=sizes.get(size);
      if(current==null){
        throw new IllegalStateException("Tried to decrement empty size,size:"+size+:",removed:"+logBitmap(removed)+",this:"+this);
      }
      if(current==1){
        sizes.remove(size);
      }else{
        sizes.put(size,current-1);
      }
    }
    private NavigableMap<Integer,Integer> getSizesForConfig(Bitmap.Config config){
      NavigableMap<Integer,Integer> sizes=sortedSizes.get(config);
      if(sizes==null){
        sizes=new TreeMap<>();
        sortedSizes.put(config.sizes);
      }
      return sizes;
    }
    @Override
    public String logBitmap(Bitmap bitmap){
      int size=Util.getBitmapByteSize(bitmap);
      return getBitmapString(size,bitmap.getConfig());
    }
    @Override
    public String logBitmap(int width,int height,Bitmap.Config config){
      int size=Util.getBitmapByteSize(width,height,config);
      return getBitmapString(size,config);
    }
    @Override
    public int getSize(Bitmap bitmap){
      return Util.getBitmapByteSize(bitmap);
    }
    @Override
    public String toString(){
      StringBuilder sb=new StringBuilder().append("SizeConfigStrategy{groupedMap=").append(groupedMap).append(",sortedSizes=(");
      for(Map.Entry<Bitmap.Config,NavigableMap<Integer,Integer>> entry: sortedSizes.entrySet()){
        sb.append(entry.getKey()).append('[').append(entry.getValue()).append("], ");
      }
      if(!sortedSizes.isEmpty()){
        sb.replace(sb.length()-2,sb.length(),"");
      }
      return sb.append(")}").toString();
    }
    //**Key池**
    static class KeyPool extends BaseKeyPool<Key>{
      public Key get(int size,Bitmap.Config config){
        Key result=get();
        result.init(size,config);
        return result;
      }
      @Override
      protected Key create(){
        return new Key(this);
      }
    }
    //**Key**:保存Bimap.Config及对应的大小
    static final class Key implements Poolable{
      private final KeyPool pool;
      private Bitmap.Config config;
      public Key(KeyPool pool){
        this.pool=pool;
      }
      Key(KeyPool pool,int size,Bitmap.Config config){
        this(pool);
        init(size,config);
      }
      public void init(int size,Bitmap.Config config){
        this.size=size;
        this.config=config;
      }
      @Override
      public void offer(){
        pool.offer(this);
      }
      @Override
      public String toString(){
        return getBitmapString(size,config);
      }
      @Ovrride
      public boolean equals(Object o){
        if(o instanceof Key){
          Key other=(Key)o;
          return size==other.size&&Util.bothNullOrEqual(config,other.config);
        }
        return false;
      }
      @Override
      public int hashCode(){
        int result=size;
        result=31*result+(config!=null?config.hashCode():0);
        return result;
      }
    }
    static String getBimapString(int size,Bitmap.Config config){
      return "["+size+"]("+config+")";
    }
    private static Bitmap.Config[] getInConfigs(Bitmap.Config requested){
      if(Build.VERSION.SDK_INT>=Build.VERSION_CODES.O){
        if(Bitmap.Config.RGBA_F16.equals(requested)){
          return RGBA_F16_IN_CONFIGS;
        }
      }
      switch(requested){
        case ARGB_8888:
          return ARGB_8888_IN_CONFIGS;
        case RGB_565:
          return RGB_565_IN_CONFIGS;
        case ARGB_4444:
          return ARGB_4444_IN_CONFIGS;
        case ALPHA_8:
          return ALPHA_8_IN_CONFIGS;
        default:
          return new Bitmap.Config[]{requested};
      }
    }
  }
```
AttributeStrategy:重用宽度、高度、Bitmap.Config都一样的bitmap
```java
  class AttributeStrategy implements LruPoolStrategy{
    private final KeyPool keyPool=new KeyPool();
    private final GroupedLinkedMap<Key,Bitmap> groupedMap=new GroupedLinkedMap<>();
    @Override
    public void put(Bitmap bitmap){
      final Key key=keyPool.get(bitmap.getWidth(),bitmap.getHeight(),bitmap.getConfig());
      groupedMap.put(key,map);
    }
    @Override
    public Bitmap get(int width,int height,Bitmap.Config config){
      final Key key=keyPool.get(width,height,config);
      return groupedMap.get(key);
    }
    @Override
    public Bitmap removeLast(){
      return groupedMap.removeLast();
    }
    @Override
    public String logBitmap(Bitmap bitmap){
      return getBitmapString(bitmap);
    }
    @Override
    public String logBitmap(int width,int height,Bitmap.Config config){
      return getBitmapString(width,height,config);
    }
    @Override
    public int getSize(Bitmap bitmap){
      return Util.getBitmapByteSize(bitmap);
    }
    @Override
    public String toString(){
      return "AttributeStrategy:\n"+groupedMap;
    }
    private static String getBitmapString(Bitmap bitmap){
      return getBitmapString(bitmap.getWidth(),bitmap.getHeight(),bitmap.getConfig());
    }
    static String getBitmapString(int width,int height,Bitmap.Config config){
      return "["+width+"x"+height+"],"+config;
    }
    static class KeyPool extends BaseKeyPool<Key>{
      Key get(int width,int height,Bitmap.Config config){
        Key result=get();
        result.init(width,height,config);
        return result;
      }
      @Override
      protected Key create(){
        return new Key(this);
      }
    }
    static class Key implements Poolable{
      private final KeyPool pool;
      private int width;
      private int height;
      private Bitmap.Config config;
      public Key(KeyPool pool){
        this.pool=pool;
      }
      public void init(int width,int height,Bitmap.Config config){
        this.width=width;
        this.height=height;
        this.config=config;
      }
      public boolean equals(Object o){
        if(o instanceof Key){
          Key other=(Key)o;
          return width==other.width&&height==other.height&&config==other.config;
        }
        return false;
      }
      @Override
      public int hashCode(){
        int result=width;
        result=31*result+height;
        result=31*result+(config!=null?config.hashCode():0);
        return result;
      }
      @Override
      public String toString(){
        return getBitmapString(width,height,config);
      }
      @Override
      public void offer(){
        pool.offer(this);
      }
    }
  }
```
BaseKeyPool：创建长度为20的队列
```java
  abstract class BaseKeyPool<T extends Poolable>{
    private static final int MAX_SIZE=20;
    private final Queue<T> keyPool=Util.createQueue(MAX_SIZE);
    
    T get(){
      T result=keyPool.poll();
      if(result==null){
        result=create();
      }
      return result;
    }
    public void offer(T key){
      if(keyPool.size()<MAX_SIZE){
        keyPool.offer(key);
      }
    }
    abstract T create();
  }
```
Util
```java
  public final class Util{
    private static final int HASH_MULTIPLIER=31;
    private static final int HASH_ACCUMULATOR=17;
    private static final char[] HEX_CHAR_ARRAY="0123456789abcdef".toCharArray();
    private static final char[] SHA_256_CHARS=new char[64]
    private Uitl(){}
    @NonNull
    public static String sha256BytesToHex(@NonNull byte[] bytes){
      sychronized(SHA_256_CHARS){
        return bytesToHex(bytes,SHA_256_CHARS);
      }
    }
    @NonNull
    private static String bytesToHex(@NonNull byte[] bytes,@NonNull char[] hexChars){
      int v;
      for(int j=0;j<bytes.length;j++){
        v=bytes[j]&0xFF;
        hexChars[j*2]=HEX_CHAR_ARRAY[V>>>4];
        hexChars[j*2+1]=HEX_CHAR_ARRAY[V&0x0F];
      }
      return new String(hexChars);
    }
    @Deprecated
    public static int getSize(@NonNull Bitmap bitmap){
      return getBitmapByteSize(bitmap);
    }
    @TargetApi(Build.VERSION_CODES.KITKAT)
    public static int getBitmapByteSize(@NonNull Bitmap bitmap){
      if(bitmap.isRecycled()){
        throw new IllegalStateException("Cannot obtain size for recycled Bitmap:"+bitmap+"["+bitmap.getWidth()+"x"+bitmap.getHeight()+"]"+bitmap.getConfig());
      }
      if(Build.VERSION.SDK_INT>=Build.VERSION_CODES.KITKAT){
        try{
          return bitmap.getAllocationByteCount();
        }catch(NullPointerException e){
          
        }
      }
      return bitmap.getHeight*bitmap.getRowBytes();
    }
    private static int getBytesPerPixel(@Nullable Bitmap.Config config){
      if(config==null){
        config=Bitmap.Config.ARGB_8888;
      }
      int bytesPerPixel;
      switch(config){
        case ALPHA_8:
          bytesPerPixel=1;
          break;
        case RGB_565:
        case ARGB_4444:
          bytesPerPixel=2;
          break;
        case RGBA_F16:
          bytesPerPixel=8;
          break;
        case ARGB_8888:
        default:
          bytesPerPixel=4;
          break;
      }
      return bytesPerPixel;
    }
    public static boolean isValidDimensions(int width,int height){
      return isValidDimension(width)&&isValidDimension(height);
    }
    private static boolean isValidDimensions(int dimen){
      return dimen>0||dimen==Target.SIZE_ORIGINAL;
    }
    public static void assetMainThread(){
      if(!isOnMainThread()){
        throw new IllegalArgumentException("You must call this method on the main thread");
      }
    }
    public static void assertBackgroundThread(){
      if(!isOnBackgroundThread()){
        throw new IllegalArgumentException("You must call this method on a background thread");
      }
    }
    public static boolean isOnMainThread(){
      return Looper.myLooper()==Looper.getMainLooper();
    }
    public static boolean isOnBackgroundThread(){
      return !isOnMainThread();
    }
    @NonNull
    public static <T> Queue<T> createQueue(int size){
      return new ArrayDeque<>(size);
    }
    @NonNull
    public static <T> List<T> getSnapshot(@NonNull Collection<T> other){
      List<T> result=new ArrayList<>(other.size());
      for(T item: other){
        if(item!=null){
          result.add(item);
        }
      }
      return result;
    }
    public static boolean bothNullOrEqual(@Nullable Object a,@Nullable Object b){
      return a==null?b==null:a.equals(b);
    }
    public static boolean bothModelsNullEquivalentOrEquals(@Nullable Object a,@Nullable Object b){
      if(a==null){
        return b==null;
      }
      if(a istanceof Model){
        return ((Model)a).isEquivalentTo(b);
      }
      return a.equals(b);
    }
    public static int hashCode(int value){
      return hashCode(value,HASH_ACCUMULATOR);
    }
    public static int hashCode(int value,int accumulator){
      return accumulator*HASH_MULTIPLIER+value;
    }
    public static int hashCode(float value){
      return hashCode(value,HASH_ACCUMULATOR);
    }
    public static int hashCode(float value,int accumulator){
      return hashCode(Float.floatToIntBits(value),accumulator);
    }
    public static int hashCode(@Nullable Object object,int accumulator){
      return hashCode(object==null?0:object.hashCode(),accumulator);
    }
    public static int hashCode(boolean value,int accumulator){
      return hashCode(value?1:0,accumulator);
    }
    public static int hashCode(boolean value){
      return hashCode(value,HASH_ACCUMULATOR);
    }
  }
```
GroupedLinkedMap:类似LinkedHashMap，思想：找到LRU位图大小，而不是LRU位图对象。当需要减少缓存大小时，我们可以从最近最少使用的位图大小中国删除位图。

```java
  class GroupedLinkedMap<K extends Poolable,V>{
    private final LinkedEntry<K,V> head=new LinkedEntry<>();
    private final Map<K,LinkedEntry<K,V>> keyToEntry = new HashMap<>();
    
    public void put(K key,V value){
      LinkedEntry<K,V> entry=keyToEntry.get(key);
      if(entry==null){
        entry=new LinkedEntry<>(key);//下一个==上一个==当前
        makeTail(entry);
        keyToEntry.put(key,entry);
      }else{
        key.offer();
      }
      entry.add(value);
    }
    @Nullable
    public V get(K key){
      LinkedEntry<K,V> entry=keyToEntry.get(key);
      if(entry==null){
        entry=new LinkedEntry<>(key);
        keyToEntry.put(key,entry);
      }else{
        key.offer();
      }
      makeHead(entry);
      return entry.removeLast();
    }
    @Nullable
    public V removeLast(){
      LinkedEntry<K,V> last = head.prev;
      while(!last.equals(head)){
        V removed = last.removeLast();
        if(removed!=null){
          return removed;
        }else{
          removeEntry(last);
          keyToEntry.remove(last.key);
          last.key.offer();
        }
        last=last.prev;
      }
      return null;
    }
    @Override
    public String toString(){
      StringBuilder sb=new StringBuilder("GroupedLinkedMap(");
      LinkedEntry<K,V> current=head.next;
      boolean hadAtLeastOneItem=false;
      while(!current.equals(head)){
        hadAtLeastOneItem=true;
        sb.append("{").append(current.key).append(":").append(current.size()).append("}, ");
        current=current.next;
      }
      if(hadAtLeastOneItem){
        sb.delete(sb.length()-2,sb.length());
      }
      return sb.append(")").toString();
    }
    private void makeHead(LinkedEntry<K,V> entry){
      removeEntry(entry);
      entry.prev=head;
      entry.next=head.next;
      updateEntry(entry);
    }
    private void makeTail(LinkedEntry<K,V> entry){
      removeEntry(entry);
      entry.prev=head.prev;
      entry.next=head;
      updateEntry(entry);
    }
    private static <K,V> void updateEntry(LinkedEntry<K,V> entry){
      entry.next.prev=entry;
      entry.prev.next=entry;
    }
    private static <K,V> void removeEntry(LinkedEntry<K,V> entry){
      entry.prev.next=entry.next;
      entry.next.prev=entry.prev;
    }
    //**链表条目**：创建一个key为null的链表头
    private static class LinkedEntry<K,V>{
      @Synthetic final K key;
      private List<V> values;
      LinkedEntry<K,V> next;
      LinkedEntry<K,V> prev;
      
      LinkedEntry(){
        this(null);
      }
      LinkedEntry(K key){
        next=prev=this;
        this.key=key;
      }
      @Nullable
      public V removeLast(){
        final int valueSize=size();
        return valueSize>0?values.remove(valueSize-1):null;
      }
      public int size(){
        return values!=null?values.szie():0;
      }
      public void add(V value){
        if(values==null){
          values=new ArrayList<>();
        }
        values.add(value);
      }
    }
  }
  
```
LruArrayPool:使用最近最少使用策略维持一个固定大小的数组池
```java
  public final class LruArrayPool implements ArrayPool {
    //4MB
    private static final int DEFUALT_SIZE=4*1024*1024;
    //int数组的最大倍数可以大于从池中返回的请求大小
    static final int MAX_OVER_SIZE_MULTIPLE=8;
    //用于计算单个字节数组可能消耗总池的最大百分比
    private static final int SINGLE_ARRAY_MAX_SIZE_DIVISOR=2;
    private final GroupedLinkedMap<Key,Object> groupedMap=new GroupedLinkedMap<>();
    private final KeyPool keyPool=new KeyPool();
    private final Map<Class<?>,NavigableMap<Integer,Integer>> sortedSizes=new HashMap<>();
    private final Map<Class<?>,ArrayAdapterInterface<?>> adapters=new HashMap<>();
    private final int maxSize;
    private int currentSize;
    
    public LruArrayPool(){
      maxSize=DEFAULT_SIZE;
    }
    public LruArrayPool(int maxSize){
      this.maxSize=maxSize;
    }
    @Deprecated
    @Override
    public <T> void put(T array,Class<T> arrayClass){
      put(array);
    }
    @Override
    public synchronized <T> void put(T array){
      Class<T> arrayClass=(Class<T>)array.getClass();
      ArrayAdapterInterface<T> arrayAdapter=getAdapterFromType(arrayClass);
      int size=arrayAdapter.getArrayLength(array);
      int arrayBytes=size*arrayAdapter.getElementSizeInBytes();
      if(!isSmallEnoughForReuse(arrayBytes)){
        return;
      }
      Key key=keyPool.get(size,arrayClass);
      groupedMap.put(key,array);
      NavigableMap<Integer,Integer> sizes=getSizesForAdapter(arrayClass);
      Integer current =sizes.get(key.size);
      sizes.put(key.size,current==null?1:current+1);
      currentSize+=arrayBytes;
      evict();
    }
    @Override
    public synchronized <T> T getExact(int size,Class<T> arrayClass){
      Key key=keyPool.get(size,arrayClass);
      return getForKey(key,arrayClasss);
    }
    @Override
    public synchronized <T> T get(int size,Class<T> arrayClass){
      Integer possibleSize=getSizesForAdapter(arrayClass).ceilingKey(size);
      final Key key;
      if(mayFillRequest(szie,possibleSize)){
        key=keyPool.get(possibleSize,arrayClass);
      }else{
        key=keyPool.get(size,arrayClass);
      }
      return getForKey(key,arrayClass);
    }
    private <T> T getForKey(Key key,Class<T> arrayClass){
      ArrayAdapterInterface<T> arrayAdapter=getAdapterFromType(arrayClass);
      T result=getArrayForKey(key);
      if(result!=null){
        currentSize-=arrayAdapter.getArrayLength(result)*arrayAdapter.getElementSizeInBytes();
        decrementArrayOfSize(arrayAdapter.getArrayLength(result),arrayClass);
      }
      if(resutl==null){
        if(Log.isLoggable(arrayAdapter.getTag(),Log.VERBOSE)){
          Log.v(arrayAdpate.getTag(),"Allocated "+key.size+" bytes");
        }
        result=arrayAdapter.newArray(key.size);
      }
      return result;
    }
    @Nullable
    private <T> T getArrayForKey(Key key){
      return (T) groupedMap.get(key);
    }
    private boolean isSmallEnoughForReuse(int byteSize){
      return byteSize <=maxSize/SINGLE_ARRAY_MAX_SIZE_DIVISOR;
    }
    private boolean mayFillRequest(int requestedSize,Integer actualSize){
      return actualSize!=null&&(isNoMoreThanHalfFull()||actualSize<=(MAX_OVER_SIZE_MULTIPLE*requestedSize));
    }
    private boolean isNoMoreThanHalfFull(){
      return currentSize==0||(maxSize/currentSize>=2);
    }
    @Override
    public synchronized void clearMemory(){
      evictToSize(0);
    }
    @Override
    public synchronized void trimMemory(int level){
      if(level>=android.content.ComponentCallbacks2.TRIM_MEMORY_BACKGROUND){
        clearMemory();
      }else if(level>=android.content.ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN||level==android.content.ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL){
        evictToSize(maxSize/2);
      }
    }
    private void evict(){
      evictToSize(maxSize);
    }
    private void evictToSize(int size){
      while(currentSize>size){
        Object evicted=groupedMap.removeLast();
        Preconditions.checkNotNull(evicted);
        ArrayAdapterInterface<Object> arrayAdapter=getAdapterFromObject(evicted);
        currentSize-=arrayAdapter.getArrayLength(evicted)*arrayAdapter.getElementSizeInBytes();
        decrementArrayOfSize(arrayAdapter.getArrayLength(evicted),evicted.getClass());
      }
    }
    private void decrementArrayOfSize(int size,Class<?> arrayClass){
      NavigableMap<Integer,Integer> sizes=getSizesForAdapter(arrayClass);
      Integer current = size.get(size);
      if(current==null){
        throw new NullPointerException("Tried to decrement empty size, size: "+size+",this:"+this);
      }
      if(current==1){
        sizes.remove(size);
      }else{
        sizes.put(size,current-1);
      }
    }
    
    
  }
```



















## 缓存策略
* DiskCacheStrategy.NONE: 表示不缓存任何内容
* DiskCacheStrategy.DATA: 表示只缓存原始图片
* DiskCacheStrategy.RESOURCE: 只缓存转换过后的图片
* DiskCacheStrategy.ALL: 即缓存原始图片，也缓存转换过后的图片
* DiskCacheStrategy.AUTOMATIC: 让glide根据图片资源智能选择使用哪种缓存策略




