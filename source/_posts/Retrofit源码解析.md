---
title: Retrofit源码解析
date: 2017-03-16 15:19:35
tags: 源码解析
---

# Retrofit介绍
官方介绍:

>Type-safe HTTP client for Android and Java by Square, Inc.

<!-- more -->

大致意思就是说，Retrofit是Square公司开发的在Android和Java上都可以是用的类型安全的HTTP客户端。
Retrofit可以和 RxJava/RxAndroid ，Gson，OkHttp，等库结合使用，其内部默认网络实现是 OkHttp ，当然这里只做 Retrofit 的源码解析。

# Retrofit 基本用法

``` java

//获取服务列表的接口
public interface ServiceService {

    @FormUrlEncoded
    @POST("service/GetServiceList.do")
    Call<ServiceListBean> getServiceList(@Field("c") String c, @Field("status")String status, @Field("page")String page, @Field("limit") String limit);

}

//MainActivity.java
Retrofit retrofit = new Retrofit.Builder()
                .baseUrl("http://xinyan.4pole.cn:8031/client/")
                //添加解析器的具体实现，主要负责对服务器返回的 Json 转换成 JavaBean
                .addConverterFactory(GsonConverterFactory.create())
                .build();
ServiceService service = retrofit.create(ServiceService.class);
Call<ServiceListBean> list = service.getServiceList("221", "1", "1", "10");

//调用接口
list.enqueue(new Callback<ServiceListBean>() {
    @Override
    public void onResponse(Call<ServiceListBean> call, Response<ServiceListBean> response) {
        Toast.makeText(MainActivity.this, "ServiceListBean:" + String.valueOf(response.body().isStatus()), Toast.LENGTH_SHORT).show();
        }

    @Override
    public void onFailure(Call<ServiceListBean> call, Throwable t) {

    }
});

```

上面就是Retrofit 默认的使用，当然这里使用了Gson的解析器需要在 app 的 build.gradle 中添加 compile 'com.squareup.retrofit2:converter-gson:2.2.0' ，主要是方便把 Json 转换成 JavaBean ,那么我先从 Retrofit 的 Builder 开始分析。

# Retrofit 源码解析

Retrofit 最具魅力的地方就是他的可扩展性，内部使用了很多设计模式，对学习设计模式很有帮助，那就从上面的 Demo 中的创建过程开始，一步步的分析下 Retrofit 是怎么实现的吧~。

## Retrofit 的构建 「构建者模式」

``` java

  //Retrfit.Builder.java
  //从类的名字就能知道，Retrofit 内部使用了构建者模式，主要是将 Retrofit 的创建简单化.
  public static final class Builder {
    //
    //Platform 类主要是一些平台相关的信息。主要包含了Android 和 Java 在不同的平台调用不同的逻辑
    private final Platform platform;
    //OkHttp 中的类
    private okhttp3.Call.Factory callFactory;
    private HttpUrl baseUrl;
    private final List<Converter.Factory> converterFactories = new ArrayList<>();
    private final List<CallAdapter.Factory> adapterFactories = new ArrayList<>();
    private Executor callbackExecutor;
    private boolean validateEagerly;

    Builder(Platform platform) {
      this.platform = platform;
      // Add the built-in converter factory first. This prevents overriding its behavior but also
      // ensures correct behavior when using converters that consume all types.
      //这里，默认用到了一个叫 BuildtInConverters 的类作为默认的解析器。
      converterFactories.add(new BuiltInConverters());
    }

    public Builder() {
      this(Platform.get());
    }

    //主要是通过另外一个 Retrofit 构建一个新的 Retrofit。
    Builder(Retrofit retrofit) {
      platform = Platform.get();
      callFactory = retrofit.callFactory;
      baseUrl = retrofit.baseUrl;
      converterFactories.addAll(retrofit.converterFactories);
      adapterFactories.addAll(retrofit.adapterFactories);
      // Remove the default, platform-aware call adapter added by build().
      adapterFactories.remove(adapterFactories.size() - 1);
      callbackExecutor = retrofit.callbackExecutor;
      validateEagerly = retrofit.validateEagerly;
    }

    /**
     * The HTTP client used for requests.
     * <p>
     * This is a convenience method for calling {@link #callFactory}.
     */
    //设置callFactory
    public Builder client(OkHttpClient client) {
      return callFactory(checkNotNull(client, "client == null"));
    }

    /**
     * Specify a custom call factory for creating {@link Call} instances.
     * <p>
     * Note: Calling {@link #client} automatically sets this value.
     */
    //设置callFactory
    public Builder callFactory(okhttp3.Call.Factory factory) {
      this.callFactory = checkNotNull(factory, "factory == null");
      return this;
    }

    /**
     * Set the API base URL.
     *
     * @see #baseUrl(HttpUrl)
     */
    public Builder baseUrl(String baseUrl) {
      checkNotNull(baseUrl, "baseUrl == null");
      //封装HttpUrl对象，这个HttpUrl对象中记录了url中的很多信息,比如协议，端口号，用户名，密码，主机名称，等等，具体实现在OkHttp中。
      HttpUrl httpUrl = HttpUrl.parse(baseUrl);
      if (httpUrl == null) {
        throw new IllegalArgumentException("Illegal URL: " + baseUrl);
      }
      return baseUrl(httpUrl);
    }

    /**
     * Set the API base URL.
     * <p>
     * The specified endpoint values (such as with {@link GET @GET}) are resolved against this
     * value using {@link HttpUrl#resolve(String)}. The behavior of this matches that of an
     * {@code <a href="">} link on a website resolving on the current URL.
     * <p>
     * <b>Base URLs should always end in {@code /}.</b>
     * <p>
     * A trailing {@code /} ensures that endpoints values which are relative paths will correctly
     * append themselves to a base which has path components.
     * <p>
     * <b>Correct:</b><br>
     * Base URL: http://example.com/api/<br>
     * Endpoint: foo/bar/<br>
     * Result: http://example.com/api/foo/bar/
     * <p>
     * <b>Incorrect:</b><br>
     * Base URL: http://example.com/api<br>
     * Endpoint: foo/bar/<br>
     * Result: http://example.com/foo/bar/
     * <p>
     * This method enforces that {@code baseUrl} has a trailing {@code /}.
     * <p>
     * <b>Endpoint values which contain a leading {@code /} are absolute.</b>
     * <p>
     * Absolute values retain only the host from {@code baseUrl} and ignore any specified path
     * components.
     * <p>
     * Base URL: http://example.com/api/<br>
     * Endpoint: /foo/bar/<br>
     * Result: http://example.com/foo/bar/
     * <p>
     * Base URL: http://example.com/<br>
     * Endpoint: /foo/bar/<br>
     * Result: http://example.com/foo/bar/
     * <p>
     * <b>Endpoint values may be a full URL.</b>
     * <p>
     * Values which have a host replace the host of {@code baseUrl} and values also with a scheme
     * replace the scheme of {@code baseUrl}.
     * <p>
     * Base URL: http://example.com/<br>
     * Endpoint: https://github.com/square/retrofit/<br>
     * Result: https://github.com/square/retrofit/
     * <p>
     * Base URL: http://example.com<br>
     * Endpoint: //github.com/square/retrofit/<br>
     * Result: http://github.com/square/retrofit/ (note the scheme stays 'http')
     */
    public Builder baseUrl(HttpUrl baseUrl) {
      checkNotNull(baseUrl, "baseUrl == null");
      List<String> pathSegments = baseUrl.pathSegments();
      //这里强制要求baseUrl必须是 / 结尾，不然会抛出异常。
      if (!"".equals(pathSegments.get(pathSegments.size() - 1))) {
        throw new IllegalArgumentException("baseUrl must end in /: " + baseUrl);
      }
      this.baseUrl = baseUrl;
      return this;
    }

    /** Add converter factory for serialization and deserialization of objects. */
    //添加解析器对象，上面的 Demo 中使用了 GsonConverFactory。
    public Builder addConverterFactory(Converter.Factory factory) {
      converterFactories.add(checkNotNull(factory, "factory == null"));
      return this;
    }

    /**
     * Add a call adapter factory for supporting service method return types other than {@link
     * Call}.
     */
    // 中添加 调用适配器，在上面的 Demo 中并没有调用这个方法。默认的调用适配器应该是 OkHttp 的实现。这里可以设置成 RxJava的调用适配器，然后就可以通过 RxJava 来处理返回结果。
    public Builder addCallAdapterFactory(CallAdapter.Factory factory) {
      adapterFactories.add(checkNotNull(factory, "factory == null"));
      return this;
    }

    /**
     * The executor on which {@link Callback} methods are invoked when returning {@link Call} from
     * your service method.
     * <p>
     * Note: {@code executor} is not used for {@linkplain #addCallAdapterFactory custom method
     * return types}.
     */
    //设置线程池的实现 Android 和 Java 分别有不同的实现
    public Builder callbackExecutor(Executor executor) {
      this.callbackExecutor = checkNotNull(executor, "executor == null");
      return this;
    }

    /**
     * When calling {@link #create} on the resulting {@link Retrofit} instance, eagerly validate
     * the configuration of all methods in the supplied interface.
     */
    //暂时不知道是啥作用
    public Builder validateEagerly(boolean validateEagerly) {
      this.validateEagerly = validateEagerly;
      return this;
    }

    /**
     * Create the {@link Retrofit} instance using the configured values.
     * <p>
     * Note: If neither {@link #client} nor {@link #callFactory} is called a default {@link
     * OkHttpClient} will be created and used.
     */
    //构建 Retrofit 对象
    public Retrofit build() {
      //BaseUrl 是必须要设置的
      if (baseUrl == null) {
        throw new IllegalStateException("Base URL required.");
      }

      //默认使用 OkHttp3 的 CallFactory
      okhttp3.Call.Factory callFactory = this.callFactory;
      if (callFactory == null) {
        callFactory = new OkHttpClient();
      }

      //默认使用 MainThreadExecutor , 内部一个MainLooper 的 Handler。 
      Executor callbackExecutor = this.callbackExecutor;
      if (callbackExecutor == null) {
        callbackExecutor = platform.defaultCallbackExecutor();
      }

      // Make a defensive copy of the adapters and add the default Call adapter.
      //设置默认的callAdapter
      List<CallAdapter.Factory> adapterFactories = new ArrayList<>(this.adapterFactories);
      adapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

      // Make a defensive copy of the converters.
      List<Converter.Factory> converterFactories = new ArrayList<>(this.converterFactories);

      //创建 Retrofit 对象 并把上面的参数 传进去。
      return new Retrofit(callFactory, baseUrl, converterFactories, adapterFactories,
          callbackExecutor, validateEagerly);
    }
  }

```

上面的代码就是Retrofit的构建过程，其中主要包含了一些参数，比如 BaseUrl，Converter，CallAdapter，CallBackExecutor，等等...其中有部分参数都是有默认的实现，从这些参数的定义，得出其中很多地方都是可以自定义的，比如：解析器，调用适配器，都做成了可扩展的，可以使用不同的方式去实现，并且 Retrofit 自身也给出了默认的实现。

## Retrofit.create() 「代理模式」

``` java
  
  //create 方法，用来创建
  public <T> T create(final Class<T> service) {
    //检查接口是否合格
    Utils.validateServiceInterface(service);
    //默认为 false
    if (validateEagerly) {
      eagerlyValidateMethods(service);
    }
    //Prxoy 类是 Java 提供用来实现动态代理的类。可以动态的在某个
    return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
        new InvocationHandler() {
          private final Platform platform = Platform.get();

          @Override public Object invoke(Object proxy, Method method, Object[] args)
              throws Throwable {
            // If the method is a method from Object then defer to normal invocation.
            //上面这段注释的意思就是说如果是调用的Object里面的方法，就不执行网络相关的逻辑。
            if (method.getDeclaringClass() == Object.class) {
              return method.invoke(this, args);
            }
            //判断是否是默认方法( Defalut Method )，默认方法是 Java8 中的特性，就是接口里面可以声明默认方法。
            if (platform.isDefaultMethod(method)) {
              return platform.invokeDefaultMethod(method, service, proxy, args);
            }
            //下面就是重点了，Retrofit 真正的逻辑就是下面这几行代码。
            //这个方法暂时先分析到这，因为样分析下面的方法需要设涉及到 CallAdapter 这个类，所以要先分析下 CallAdapter 这个类
            ServiceMethod<Object, Object> serviceMethod =
                (ServiceMethod<Object, Object>) loadServiceMethod(method);
            OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
            return serviceMethod.callAdapter.adapt(okHttpCall);
          }
        });
  }

   //代码目录:Utils.java
   //检测接口定义是否合格
   //1. 必须要是结构 不能是类
   //2. 这个接口不能继承别的接口
   static <T> void validateServiceInterface(Class<T> service) {
    if (!service.isInterface()) {
      throw new IllegalArgumentException("API declarations must be interfaces.");
    }
    // Prevent API interfaces from extending other interfaces. This not only avoids a bug in
    // Android (http://b.android.com/58753) but it forces composition of API declarations which is
    // the recommended pattern.
    if (service.getInterfaces().length > 0) {
      throw new IllegalArgumentException("API interfaces must not extend other interfaces.");
    }
  }

   //加载ServiceMethod
   ServiceMethod<?, ?> loadServiceMethod(Method method) {
    //第三方库的老套路，缓存获取
    ServiceMethod<?, ?> result = serviceMethodCache.get(method);
    if (result != null) return result;

    //防止多线程重复创建
    synchronized (serviceMethodCache) {
      result = serviceMethodCache.get(method);
      if (result == null) {
        //没有缓存，创建一个新的并添加到缓存里面。
        //ServiceMethod 对应的就是我们在前面接口中声明的方法的封装对象。
        result = new ServiceMethod.Builder<>(this, method).build();
        serviceMethodCache.put(method, result);
      }
    }
    return result;
  }

```

Retrofit 的 create 方法中使用了 Java 中的动态代理类，熟悉 Spring 框架的应该知道里面有一个叫 AOP 的东西，作用就是可以在某个方法执行前和执行后执行某段代码，最主要是动态生成的一个代理类，可以在很多地方用到，比如记录日志，做一些公共的操作，等等之类的，在 Retrofit 中使用动态代理在我们调用接口的方法的时候为我们实现了网络方面的调用，并根据设置 CallAdapter 将数据封装返回。


## Converter.Factory 和 CallAdpater.Factory 「工厂模式」

在 Retrofit 构建的时候，涉及到两个可配置的类。一个叫CallAdapter，另一个叫Converter。
其中CallAdapter负责结构的返回类型，即我们上面在定义 Service 的时候，方法的返回值，上面我们用的是Call<ServiceListBean>，这里的Call是可以我们自定义的，如 RxJava2 实现的 RxJava2CallAdapterFactory ，具体的实现这里不做分析，但是这里无论是 RxJava2 RxJava2CallAdapterFactory 还是 Retrofit 默认的实现 ExecutorCallAdapterFactory ，都是继承至 CallAdapter.Factory，具体代码如下。

``` java

//CallAdapter interface 
public interface CallAdapter<R, T> {

  // 返回类型 Call ResponseBody Observable 等等....
  Type responseType();

  //这个方法会在 create 中调用 具体是
  T adapt(Call<R> call);
 
  abstract class Factory {

    //通过 return 返回 CallAdapter 的具体实现。
    public abstract CallAdapter<?, ?> get(Type returnType, Annotation[] annotations,
        Retrofit retrofit);


    protected static Type getParameterUpperBound(int index, ParameterizedType type) {
      return Utils.getParameterUpperBound(index, type);
    }


    protected static Class<?> getRawType(Type type) {
      return Utils.getRawType(type);
    }
  }
}


// Retrofit 默认实现的 ExecutorCallAdapterFactory
final class ExecutorCallAdapterFactory extends CallAdapter.Factory {
  //线程池
  final Executor callbackExecutor;

  ExecutorCallAdapterFactory(Executor callbackExecutor) {
    this.callbackExecutor = callbackExecutor;
  }

  @Override
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    //获取泛型中的类型，上面Demo中获取的就是ServiceListBean  
    final Type responseType = Utils.getCallResponseType(returnType);
    //new 一个CallAdapter 返回
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public Call<Object> adapt(Call<Object> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }

  //实现 Call接口 具体的调用走这个类，比如上面 Demo 中的 enqueue 方法。
  static final class ExecutorCallbackCall<T> implements Call<T> {
    final Executor callbackExecutor;
    final Call<T> delegate;

    //其实这个类知识一个空壳，具体的实现在 delegate 对象中。
    ExecutorCallbackCall(Executor callbackExecutor, Call<T> delegate) {
      this.callbackExecutor = callbackExecutor;
      this.delegate = delegate;
    }

    @Override public void enqueue(final Callback<T> callback) {
      if (callback == null) throw new NullPointerException("callback == null");

      delegate.enqueue(new Callback<T>() {
        @Override public void onResponse(Call<T> call, final Response<T> response) {
          //线程池调用 具体是在MainLooper的Handler 中 执行，也就是 Ui 线程，具体代码在 Platform.MainThraedExecutor.java
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              if (delegate.isCanceled()) {
                // Emulate OkHttp's behavior of throwing/delivering an IOException on cancellation.
                callback.onFailure(ExecutorCallbackCall.this, new IOException("Canceled"));
              } else {
                callback.onResponse(ExecutorCallbackCall.this, response);
              }
            }
          });
        }

        @Override public void onFailure(Call<T> call, final Throwable t) {
          callbackExecutor.execute(new Runnable() {
            @Override public void run() {
              //失败的回调
              callback.onFailure(ExecutorCallbackCall.this, t);
            }
          });
        }
      });
    }

    //是否执行
    @Override public boolean isExecuted() {
      return delegate.isExecuted();
    }

    //执行
    @Override public Response<T> execute() throws IOException {
      return delegate.execute();
    }

    //取消
    @Override public void cancel() {
      delegate.cancel();
    }

    //是否已经取消
    @Override public boolean isCanceled() {
      return delegate.isCanceled();
    }

    //Clone
    @SuppressWarnings("CloneDoesntCallSuperClone") // Performing deep clone.
    @Override public Call<T> clone() {
      return new ExecutorCallbackCall<>(callbackExecutor, delegate.clone());
    }

    //请求
    @Override public Request request() {
      return delegate.request();
    }
  }
}

```


>在上面的代码中，可以看出Retrofit 的CallAdapter 可以任意定制，只要实现对应的方法，其中比较重要的方法就是 get ,这个方法使用抽象工厂模式通过返回抽象的"产品"，将创建过程封装在内部，只需要传入 returnType 。并且在 Call 的具体实现中 也使用来 委派模式 真正的实现在 OkHttpCall.java 。

## ServiceMethod 构建过程

``` java

public ServiceMethod build() {  
      //创建callAdapter 主要是通过接口方法的返回类型来确定使用哪个CallAdapter
      callAdapter = createCallAdapter();
      //通过callAdapter 获取返回类型
      responseType = callAdapter.responseType();
      if (responseType == Response.class || responseType == okhttp3.Response.class) {
        throw methodError("'"
            + Utils.getRawType(responseType).getName()
            + "' is not a valid response body type. Did you mean ResponseBody?");
      }

      //创建responseConverter
      responseConverter = createResponseConverter();

      //解析注解
      for (Annotation annotation : methodAnnotations) {
        parseMethodAnnotation(annotation);
      }

      //请求方式不能为空
      if (httpMethod == null) {
        throw methodError("HTTP method annotation is required (e.g., @GET, @POST, etc.).");
      }

      //如果没有没有Body
      if (!hasBody) {
        if (isMultipart) {
          throw methodError(
              "Multipart can only be specified on HTTP methods with request body (e.g., @POST).");
        }
        if (isFormEncoded) {
          throw methodError("FormUrlEncoded can only be specified on HTTP methods with "
              + "request body (e.g., @POST).");
        }
      }

      //parameterAnnotationArray 存储的是 service 参数使用的注解
      //获取参数的个数 parameterAnnotationsArray 是一个二维数组
      int parameterCount = parameterAnnotationsArray.length;
      //初始化 parameterHandlers
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {

        //parameterTypes 存储的是 service 方法参数的类型
        Type parameterType = parameterTypes[p];
        //判断方法参数的类型是否正常
        if (Utils.hasUnresolvableType(parameterType)) {
          throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
              parameterType);
        }

        //如果发现没有使用 Retrofit 的注解，就抛出异常。
        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        if (parameterAnnotations == null) {
          throw parameterError(p, "No Retrofit annotation found.");
        }

        //解析parameter
        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }

      if (relativeUrl == null && !gotUrl) {
        throw methodError("Missing either @%s URL or @Url parameter.", httpMethod);
      }
      if (!isFormEncoded && !isMultipart && !hasBody && gotBody) {
        throw methodError("Non-body HTTP method cannot contain @Body.");
      }
      if (isFormEncoded && !gotField) {
        throw methodError("Form-encoded method must contain at least one @Field.");
      }
      if (isMultipart && !gotPart) {
        throw methodError("Multipart method must contain at least one @Part.");
      }

      return new ServiceMethod<>(this);
    }  

    //因为方法的参数是可以使用多个注解的，所以上面的 parameterAnnotationsArray 是一个二维数组
    //上面这个方法涉及到几个步骤，下面一个一个分析。

    //1. 解析参数
    private ParameterHandler<?> parseParameter(
        int p, Type parameterType, Annotation[] annotations) {
      ParameterHandler<?> result = null;
      for (Annotation annotation : annotations) {
        ParameterHandler<?> annotationAction = parseParameterAnnotation(
            p, parameterType, annotations, annotation);

        //如果解析失败直接跳过，解析下一个。
        if (annotationAction == null) {
          continue;
        }

        // Retrofit 只支持参数一个注解的情况
        if (result != null) {
          throw parameterError(p, "Multiple Retrofit annotations found, only one allowed.");
        }

        result = annotationAction;
      }

      if (result == null) {
        throw parameterError(p, "No Retrofit annotation found.");
      }

      //返回解析结果
      return result;
    }

    //
    private ParameterHandler<?> parseParameterAnnotation(
        int p, Type type, Annotation[] annotations, Annotation annotation) {

       //内容省略，主要是用来解析参数的注解 如 @Field @Path @Url 等等...
       //这里分析一下 @Field 
       //@Field 注解必须要配合 @FormUrlEncoded 注解一起使用，不然会报下面的异常
       if (!isFormEncoded) {
          throw parameterError(p, "@Field parameters can only be used with form encoding.");
        }
        Field field = (Field) annotation;
        String name = field.value();
        boolean encoded = field.encoded();

        gotField = true;

        Class<?> rawParameterType = Utils.getRawType(type);
        //判断参数是什么类型 迭代器类型 数组类型 或者 其他类型，分别都有不同的处理方式
        if (Iterable.class.isAssignableFrom(rawParameterType)) {
          if (!(type instanceof ParameterizedType)) {
            throw parameterError(p, rawParameterType.getSimpleName()
                + " must include generic type (e.g., "
                + rawParameterType.getSimpleName()
                + "<String>)");
          }
          ParameterizedType parameterizedType = (ParameterizedType) type;
          Type iterableType = Utils.getParameterUpperBound(0, parameterizedType);
          Converter<?, String> converter =
              retrofit.stringConverter(iterableType, annotations);
          return new ParameterHandler.Field<>(name, converter, encoded).iterable();
        } else if (rawParameterType.isArray()) {
          Class<?> arrayComponentType = boxIfPrimitive(rawParameterType.getComponentType());
          Converter<?, String> converter =
              retrofit.stringConverter(arrayComponentType, annotations);
          return new ParameterHandler.Field<>(name, converter, encoded).array();
        } else {
          //参数为String类型会执行下面的代码，ParameterHandler.Field 是继承至 ParameterHandler，这里默认会返回  BuiltInConverters.ToStringConverter 这个类。
          Converter<?, String> converter =
              retrofit.stringConverter(type, annotations);
          return new ParameterHandler.Field<>(name, converter, encoded);
        }
     }
```

>上面就是 ServiceMethod 的构建过程，首先解析方法的注解，然后在解析参数的注解，最后在封装成对应的对象，这里就没什么好说的了。


## adapt 方法

经过上面的分析 ServiceMethod 对象也创建完了，剩下的就是将CallAdapter 和 ServiceMethod 关联在一起，具体的实现如下:

``` java

  //默认的CallAdapter,ExecutorCallbackCallFactory.java
  public CallAdapter<?, ?> get(Type returnType, Annotation[] annotations, Retrofit retrofit) {
    if (getRawType(returnType) != Call.class) {
      return null;
    }
    final Type responseType = Utils.getCallResponseType(returnType);
    return new CallAdapter<Object, Call<?>>() {
      @Override public Type responseType() {
        return responseType;
      }

      @Override public Call<Object> adapt(Call<Object> call) {
        return new ExecutorCallbackCall<>(callbackExecutor, call);
      }
    };
  }


  //enqueue 上面Demo 中我们调用的方法，传入一个回调，结果会通过回调进行返回。
  @Override public void enqueue(final Callback<T> callback) {
    if (callback == null) throw new NullPointerException("callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          failure = creationFailure = t;
        }
      }
    }
  }

  //OkHttp的回调
  @Override public void enqueue(final Callback<T> callback) {
    if (callback == null) throw new NullPointerException("callback == null");

    okhttp3.Call call;
    Throwable failure;

    synchronized (this) {
      if (executed) throw new IllegalStateException("Already executed.");
      executed = true;

      call = rawCall;
      failure = creationFailure;
      if (call == null && failure == null) {
        try {
          call = rawCall = createRawCall();
        } catch (Throwable t) {
          failure = creationFailure = t;
        }
      }
    }

    if (failure != null) {
      callback.onFailure(this, failure);
      return;
    }

    if (canceled) {
      call.cancel();
    }

    call.enqueue(new okhttp3.Callback() {
      @Override public void onResponse(okhttp3.Call call, okhttp3.Response rawResponse)
          throws IOException {
        Response<T> response;
        try {
          //解析Response
          response = parseResponse(rawResponse);
        } catch (Throwable e) {
          callFailure(e);
          return;
        }
        //请求成功!
        callSuccess(response);
      }

      @Override public void onFailure(okhttp3.Call call, IOException e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callFailure(Throwable e) {
        try {
          callback.onFailure(OkHttpCall.this, e);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }

      private void callSuccess(Response<T> response) {
        try {
          //注意 这里的Response 已经有泛型了，说明结果已经转换完了
          //需要注意的是这里的Response 和 Response<T> 是不一样的，前者是 OkHttp 中的，后者是 Retrofit 中的。
          callback.onResponse(OkHttpCall.this, response);
        } catch (Throwable t) {
          t.printStackTrace();
        }
      }
    });
  }

  //解析 Response
  Response<T> parseResponse(okhttp3.Response rawResponse) throws IOException {
    ResponseBody rawBody = rawResponse.body();

    // Remove the body's source (the only stateful object) so we can pass the response along.
    rawResponse = rawResponse.newBuilder()
        .body(new NoContentResponseBody(rawBody.contentType(), rawBody.contentLength()))
        .build();

    int code = rawResponse.code();
    if (code < 200 || code >= 300) {
      try {
        // Buffer the entire body to avoid future I/O.
        ResponseBody bufferedBody = Utils.buffer(rawBody);
        return Response.error(bufferedBody, rawResponse);
      } finally {
        rawBody.close();
      }
    }

    if (code == 204 || code == 205) {
      rawBody.close();
      return Response.success(null, rawResponse);
    }

    ExceptionCatchingRequestBody catchingBody = new ExceptionCatchingRequestBody(rawBody);
    try {
      //解析结果，具体的解析器是用在构建的时候设定的 Converter
      T body = serviceMethod.toResponse(catchingBody);
      return Response.success(body, rawResponse);
    } catch (RuntimeException e) {
      // If the underlying source threw an exception, propagate that rather than indicating it was
      // a runtime exception.
      catchingBody.throwIfCaught();
      throw e;
    }
  }

  //代码目录：ServiceMethod.java
  R toResponse(ResponseBody body) throws IOException {
    return responseConverter.convert(body);
  }

```

# 总结

分析了这么多，上面大概就是 Retrofit 的整体执行逻辑，用到了很多设计模式，就是因为用了很多设计模式，所以 Retrofit 是一个扩展性很好的框架。
其中大概对CallAdapter 和 Converter 这两样东西做了扩展，具体的实现逻辑上面也分析了，总之 Retrofit 是一个使用了很多设计模式的框架，框架的本体并没有实现什么具体的逻辑，比如请求是用的OkHttp，结果的解析可以用Gson，返回结果的封装也可用使用RxJava等等...，自己只负责将这几个东西组合在一起，并使用接口 + 注解的方式来定义服务器的接口，将接口的定义和接口的调用进行了解耦，很推荐在项目中使用。

