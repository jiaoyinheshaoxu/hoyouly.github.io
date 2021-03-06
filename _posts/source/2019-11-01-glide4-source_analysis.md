---
layout: post
title: 源码分析 - Glide4 之 with() 和 load()
category: 源码分析
tags: glide4
---
* content
{:toc}

# GlideAPP 是个啥玩意

Glide4 之前的版本，一般使用Glide 就是
`Glide.with(context).load("http:xxxx").into(target) ` 这种形式，可是突然发现，在Glide4后竟然多了一个GlideApp,这是个啥玩意啊，Glide框架中没有这个类啊，从哪里过来的呢，原来这个使用的是APT技术，通过GlideModule 这个注解动态生成的，本质上还是使用的Glide的接口。

## APT 技术
Annotation Processing Tool 可以在代码编译期间对注解进行处理，并且生成Java 文件，减少手动的代码输入。
注解分为
* 编译时注解  例如Dagger2，ButterKnife，EventBus3 ,
* 运行时注解  例如Retortfit ,通过动态代理生成网络请求。

## GlideModule 注解

```java
/**
 *在编译时，为AppGlideModules和LibraryGlideModules提供注入。
 *替换掉  AndroidManifest.xml 中value="GlideModule" 的 <meta-data /> 。
 *这里需要注意后续需要用到<meta-data />这个标签，先记住此处
 */
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.SOURCE)
public @interface GlideModule {
  /**
   *此处返回的name就是你在使用时的class name。
   *eg:将GlideApp改为GlideAppX
   *那么通过注解生成的类就是GlideAppX
   *那么你使用时候就会是 GlideAppX.with(this)
   */
  String glideName() default "GlideApp";
}
```
## 继承 AppGlideModule
主要包括两个方法 applyOptions()和 isMannifestParsingEnabled()
* isMannifestParsingEnabled()  是否检测AndroidManifest里面的GlideModule,默认返回true。注解和继承AppGlideModule生成自己的module时，官方要求我们实现这个方法，返回并且false，这样避免AndroidManifest加载两次。
* applyOptions()，可以在Glide被创建之前，对GlideBuilder进行设置，比如设置app 缓存的路径，缓存的大小等。

GlideApp.with()其实还是调用了Glide.with()
```Java
/**
 * @see Glide#with(Activity)
 */
@NonNull
public static GlideRequests with(@NonNull Activity activity) {
  return (GlideRequests) Glide.with(activity);
}
```

在 [ 源码分析 - Glide4 之 概况 ](../../../../2019/07/10/glide4/)中我们知道，Glide 加载流程是

```
model(数据源)-->data(转换数据)-->decode(解码)-->transformed(缩放)-->transcoded(转码)-->encoded(编码保存到本地)
```
那么我们就对照着 Glide.with(context).load("http:xxxx").into(target) 这行代码，看到是怎么加载的吧。


# Glide # with()
一组静态方法，有好几个重载方法，
例如
```
with(android.app.Activity)
with(android.app.Fragment)
with(android.support.v4.app.Fragment)
with(android.support.v4.app.FragmentActivity)
with(android.view)
```
这些方法最后都调用了`getRetriever(activity).get(activity)`,返回的是一个RequestManager对象


getRetriever(activity) 的源码如下，
```Java
private static RequestManagerRetriever getRetriever(@Nullable Context context) {
   return Glide.get(context).getRequestManagerRetriever();
 }
```
这里面主要干了两件事
1. 得到Glide对象，
2. 通过得到的Glide对象，得到 RequestManagerRetriever，

接下来我们就看第一步，得到 Glide 对象
## 创建Glide对象
### Glide # get()

```java
@NonNull
 public static Glide get(@NonNull Context context) {
   if (glide == null) {
     synchronized (Glide.class) {
       if (glide == null) {
         checkAndInitializeGlide(context);
       }
     }
   }
   return glide;
 }
```
这个简单了，双重校验的单例模式，没有Glide对象，肯定会在 checkAndInitializeGlide() 中创建一个，否则直接使用。

```java
private static void checkAndInitializeGlide(@NonNull Context context) {
    isInitializing = true;
    initializeGlide(context);
    isInitializing = false;
  }
private static void initializeGlide(@NonNull Context context) {
    initializeGlide(context, new GlideBuilder());
}
@SuppressWarnings("deprecation")
private static void initializeGlide(@NonNull Context context, @NonNull GlideBuilder builder) {
  Context applicationContext = context.getApplicationContext();
  GeneratedAppGlideModule annotationGeneratedModule = getAnnotationGeneratedGlideModules();
    ...
  Glide glide = builder.build(）applicationContext);
  for (com.bumptech.glide.module.GlideModule module : manifestModules) {
    module.registerComponents(applicationContext, glide, glide.registry);
  }
  if (annotationGeneratedModule != null) {
    annotationGeneratedModule.registerComponents(applicationContext, glide, glide.registry);
  }
  applicationContext.registerComponentCallbacks(glide);
  Glide.glide = glide;
}

```
创建Glide的过程，主要包括如下工作。
1. 使用APT技术动态加载 GeneratedAppGlideModule 对象，然后配置Glide
2. 使用构造者模式GlideBuilder创建Glide对象
3. 使用applicationContext.registerComponentCallbacks()，Glide 实现了ComponentCallbacks2接口，接口中void onTrimMemory(@TrimMemoryLevel int level);的方法会在操作系统内存不足的时候会调用

接下来主要看 得到Glide对象的操作，即 builder.build(）

### GlideBuilder # build()
```Java
Glide build(@NonNull Context context) {
    ...
    if (engine == null) {
      engine =new Engine(...);
    }
    ...
    RequestManagerRetriever requestManagerRetriever =new RequestManagerRetriever(requestManagerFactory);
    return new Glide(...);
  }
```

* 创建了一个RequestManagerRetriever对象，这个就是 Glide.get(context).getRequestManagerRetriever()得到的那个对象， 它职责就是创建RequestManager，
* 还有创建了一个Engine对象，这个后面会用到

Glide 是单例模式，只有一个，那么RequestManagerRetriever 是在builde中创建的，所以 RequestManagerRetriever 对象也是只有一个，同理 Engine 对象也只有一个。


然后看Glide的构造函数干了啥事情
```java
Glide(...) {
    ...
    registry
        //将byteBuffer转换为Flile
        .append(ByteBuffer.class, new ByteBufferEncoder())
        //将inputStream转换为File
        .append(InputStream.class, new StreamEncoder(arrayPool))
        /* Bitmaps */
        .append(Registry.BUCKET_BITMAP, ByteBuffer.class, Bitmap.class, byteBufferBitmapDecoder)
        .append(Registry.BUCKET_BITMAP, InputStream.class, Bitmap.class, streamBitmapDecoder)
        .append(Registry.BUCKET_BITMAP,ParcelFileDescriptor.class,Bitmap.class,parcelFileDescriptorVideoDecoder)
        .append(Registry.BUCKET_BITMAP,AssetFileDescriptor.class,Bitmap.class,VideoDecoder.asset(bitmapPool))
        .append(Bitmap.class, Bitmap.class, UnitModelLoader.Factory.<Bitmap>getInstance())
        .append(Registry.BUCKET_BITMAP, Bitmap.class, Bitmap.class, new UnitBitmapDecoder())
        .append(Bitmap.class, bitmapEncoder)
        /* BitmapDrawables */
        //ByteBuffer解码成Bitmap :decoderRegistry
        .append(Registry.BUCKET_BITMAP_DRAWABLE,ByteBuffer.class,BitmapDrawable.class,
            new BitmapDrawableDecoder<>(resources, byteBufferBitmapDecoder))
        ...
    glideContext =new GlideContext(...);
  }
```
注册了ModuleLoader,Encoder,Decoder,Transcoder这四个组件，使其具备转换模型数据。解析String，url等源，解析加载Bitmap，Drawable，file用的Decoder，
对资源进行转码的Transcoder,
将资源转File的Encoder

glide 创建后，RequestManagerRetriever 对象也有了，那就看看 RequestManagerRetriever中 get(...) 中干嘛了啊

### RequestManagerRetriever # get()
```Java
@NonNull
public RequestManager get(@NonNull FragmentActivity activity) {
 if (Util.isOnBackgroundThread()) {
   return get(activity.getApplicationContext());
 } else {
   assertNotDestroyed(activity);
   FragmentManager fm = activity.getSupportFragmentManager();
   return supportFragmentGet(activity, fm, /*parentHint=*/ null, isActivityVisible(activity));
 }
}

@NonNull
public RequestManager get(@NonNull Context context) {
  if (context == null) {
    throw new IllegalArgumentException("You cannot start a load on a null Context");
  } else if (Util.isOnMainThread() && !(context instanceof Application)) {
    //在主线程并且context 不属于 Application
    if (context instanceof FragmentActivity) {
      return get((FragmentActivity) context);
    } else if (context instanceof Activity) {
      return get((Activity) context);
    } else if (context instanceof ContextWrapper) {
      return get(((ContextWrapper) context).getBaseContext());
    }
  }
  return getApplicationManager(context);
}
```
将参数分为两种
1. 当传入的是 Application对象的时候，就会调用getApplicationManager(context)得到RequestManager对象
2. 当传入的不是 Application对象的时候，就会执行相应的方法，但是这三个方法都最终执行到了`fragmentGet(activity, fm,  null, isActivityVisible(activity));或者 supportFragmentGet(activity, fm,  null, isActivityVisible(activity))`
3. 如果是在子线程中，默认就当Application处理

![](../../../../images/glide_requestmanagerretriever_get.png)

如图，get()最终都会执行到fragmentGet（）和supportFragmentGet（）这两个方法中。
那么就看看这两个方法的区别啊
```java
private RequestManager fragmentGet(@NonNull Context context,@NonNull android.app.FragmentManager fm,
     @Nullable android.app.Fragment parentHint, boolean isParentVisible) {
 RequestManagerFragment current = getRequestManagerFragment(fm, parentHint, isParentVisible);
 RequestManager requestManager = current.getRequestManager();
 if (requestManager == null) {
   Glide glide = Glide.get(context);
   requestManager =factory.build(glide, current.getGlideLifecycle()
          , current.getRequestManagerTreeNode(), context);
   current.setRequestManager(requestManager);
 }
 return requestManager;
}

private RequestManager supportFragmentGet( @NonNull Context context,@NonNull FragmentManager fm,
     @Nullable Fragment parentHint, boolean isParentVisible) {
 SupportRequestManagerFragment current =getSupportRequestManagerFragment(fm, parentHint, isParentVisible);
 RequestManager requestManager = current.getRequestManager();
 if (requestManager == null) {
   Glide glide = Glide.get(context);
   requestManager =factory.build(glide, current.getGlideLifecycle()
            , current.getRequestManagerTreeNode(), context);
   current.setRequestManager(requestManager);
 }
 return requestManager;
}
```
1. 创建一个 SupportRequestManagerFragment 或者RequestManagerFragment ,这就是为啥我们在得到Fragment的数量的时候，会多出来一个Fragment的原因

添加这个隐藏的Fragment，主要是为了监听当前Activity的生命周期，如果Activity已经销毁，那么对应的Fragment也就没有了，可以在这个隐藏的Fragment的销毁的时候，停止加载图片。

2. 如果创建的Fragment得到的 RequestManger，那么通过factory build()一个，build的时候，会关联Fragment的 LifeCircle和RequestManagerTree,然后设置到Fragment中,这样，RequestManger 就和Fragment的生命周期绑定了


总结一下，with()方法就是根据Context 类型绑定一个生命周期的 RequestManager
# RequestManger # load()

```java
public RequestBuilder<Drawable> load(@Nullable String string) {
   return asDrawable().load(string);
 }

 public RequestBuilder<TranscodeType> load(@Nullable String string) {
   return loadGeneric(string);
 }

 private RequestBuilder<TranscodeType> loadGeneric(@Nullable Object model) {
     this.model = model;
     isModelSet = true;
     return this;
   }
```
这个就简单了，只是把load()过来的数据保存到一个model中，
isModelSet 设置为true ,这个变量在into()中还会用到，如果没有先执行load()，直接使用into(),那么就会报错。
load()有很多重载方法，但是最后都是执行到了loadGeneric()中

![](../../../../images/glide_load.png)

这就是Glide 流程的第一步，把 资源转换成一个model

load()返回的是 RequestBuilder 对象，所以第三步 就是查看 RequestBuilder的into()方法

详情查看  承接上文 [ 源码分析 - Glide4 之 into() ](../../../../2019/11/01/glide4-source_analysis_2/)


---
搬运地址：    

[Glide4.0源码全解析（一），GlideAPP和.with()方法背后的故事](https://blog.csdn.net/github_33304260/article/details/77869221)    

[Glide4.0源码全解析（二），load()背后的故事](https://blog.csdn.net/github_33304260/article/details/77992717)    

[Glide源码解析(三)](https://blog.csdn.net/pmx_121212/article/details/79085947)     

[Android图片加载库Glide 知其然知其所以然 开篇](https://juejin.im/post/5cbea88cf265da03555c7f58)    

[Android 图片加载库Glide知其然知其所以然之加载](https://juejin.im/post/5cd51235e51d456e831f69f6)    

[Glide架构设计艺术](https://www.jianshu.com/p/5c8ce241199e)     

[Glide源码导读](https://www.cnblogs.com/angeldevil/p/5841979.html)   

[Android源码系列-解密Glide](https://juejin.im/post/5e5c5e23f265da570c753621#heading-29)
