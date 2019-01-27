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
```



## 缓存策略
* DiskCacheStrategy.NONE: 表示不缓存任何内容
* DiskCacheStrategy.DATA: 表示只缓存原始图片
* DiskCacheStrategy.RESOURCE: 只缓存转换过后的图片
* DiskCacheStrategy.ALL: 即缓存原始图片，也缓存转换过后的图片
* DiskCacheStrategy.AUTOMATIC: 让glide根据图片资源智能选择使用哪种缓存策略



























