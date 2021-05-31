---
title: Retrofit2原理解析
date: 2020-04-13 21:53:06
catalog:    true
header-img: "post-bg-unix-linux.jpg"
tags:
    - Java
    - Android
---

## 如何使用

```js
public interface GitHubService {
    @GET("users/{user}/repos")
    Call<List<String>> listRepos(@Path("user") String user);
}


Retrofit retrofit = new Retrofit.Builder()
        .baseUrl("https://api.github.com/")
        .addConverterFactory(GsonConverterFactory.create())
        .build();


GitHubService service = retrofit.create(GitHubService.class);
Call<List<String>> repos = service.listRepos("octocat");
repos.enqueue(new Callback<List<String>>() {
    @Override
    public void onResponse(Call<List<String>> call, Response<List<String>> response) {
        
    }

    @Override
    public void onFailure(Call<List<String>> call, Throwable t) {

    }
});
```
上面使用中主要做了以下几件事：
* 通过Retrofit.Builder()创建Retrofit。 
* 调用Retrofit的create()方法将生成接口GitHubService的实例。 
* 调用GitHubService的listRepos()方法返Call<List<Repo>>。 
* 执行Call<List<Repo>>的enqueue方法在回调中处理返回的List<Repo>。

## 源码解析
**Retrofit初始化Builder**
```js
Builder(Platform platform) {
  this.platform = platform;
}

public Builder() {
  this(Platform.get());
}

private static final Platform PLATFORM = findPlatform();

static Platform get() {
  return PLATFORM;
}

private static Platform findPlatform() {
  try {
    Class.forName("android.os.Build");
    if (Build.VERSION.SDK_INT != 0) {
      return new Android();
    }
  } catch (ClassNotFoundException ignored) {
  }
  try {
    Class.forName("java.util.Optional");
    return new Java8();
  } catch (ClassNotFoundException ignored) {
  }
  return new Platform();
}
```
这里分析返回的是Android对象

**再看一下retrofit的build方法**
```js
public Retrofit build() {
  if (baseUrl == null) {
    throw new IllegalStateException("Base URL required.");
  }

  okhttp3.Call.Factory callFactory = this.callFactory;
  if (callFactory == null) {
    callFactory = new OkHttpClient();
  }

  Executor callbackExecutor = this.callbackExecutor;
  if (callbackExecutor == null) {
    callbackExecutor = platform.defaultCallbackExecutor();
  }

  // Make a defensive copy of the adapters and add the default Call adapter.
  List<CallAdapter.Factory> callAdapterFactories = new ArrayList<>(this.callAdapterFactories);
  callAdapterFactories.add(platform.defaultCallAdapterFactory(callbackExecutor));

  // Make a defensive copy of the converters.
  List<Converter.Factory> converterFactories =
      new ArrayList<>(1 + this.converterFactories.size());

  // Add the built-in converter factory first. This prevents overriding its behavior but also
  // ensures correct behavior when using converters that consume all types.
  converterFactories.add(new BuiltInConverters());
  converterFactories.addAll(this.converterFactories);

  return new Retrofit(callFactory, baseUrl, unmodifiableList(converterFactories),
      unmodifiableList(callAdapterFactories), callbackExecutor, validateEagerly);
}
```
1.如果没有callFactory则new一个OkHttpClient对象，由源码可知如果OkHttpClient没有什么特殊要求的话，在初始化retrofit的时候client()方法其实可以不需要调用。
2.对callbackExecutor进行赋值，因为platform是Android，所以callbackExecutor=new MainThreadExecutor()，该对象的做用就是进行主线程切换
```js
static class Android extends Platform {
  @Override public Executor defaultCallbackExecutor() {
    return new MainThreadExecutor();
  }

  @Override CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
    if (callbackExecutor == null) throw new AssertionError();
    return new ExecutorCallAdapterFactory(callbackExecutor);
  }

  static class MainThreadExecutor implements Executor {
    private final Handler handler = new Handler(Looper.getMainLooper());

    @Override public void execute(Runnable r) {
      handler.post(r);
    }
  }
}
```
3.往callAdapterFactories里面添加一个CallAdapter.Factory对象，该对象为ExecutorCallAdapterFactory
```js
CallAdapter.Factory defaultCallAdapterFactory(@Nullable Executor callbackExecutor) {
  if (callbackExecutor != null) {
    return new ExecutorCallAdapterFactory(callbackExecutor);
  }
  return DefaultCallAdapterFactory.INSTANCE;
}
```
4.往converterFactories中添加一个BuiltInConverters对象

**再看一下retrofit的create方法**
```js
public <T> T create(final Class<T> service) {
  Utils.validateServiceInterface(service);
  if (validateEagerly) {
    eagerlyValidateMethods(service);
  }
  return (T) Proxy.newProxyInstance(service.getClassLoader(), new Class<?>[] { service },
      new InvocationHandler() {
        private final Platform platform = Platform.get();

        @Override public Object invoke(Object proxy, Method method, @Nullable Object[] args)
            throws Throwable {
          Log.w("scj", "create invoke-->" );
          // If the method is a method from Object then defer to normal invocation.
          if (method.getDeclaringClass() == Object.class) {
            return method.invoke(this, args);
          }
          if (platform.isDefaultMethod(method)) {
            return platform.invokeDefaultMethod(method, service, proxy, args);
          }
          ServiceMethod<Object, Object> serviceMethod =
              (ServiceMethod<Object, Object>) loadServiceMethod(method);
          OkHttpCall<Object> okHttpCall = new OkHttpCall<>(serviceMethod, args);
          return serviceMethod.adapt(okHttpCall);
        }
      });
}
```
//create方法传入一个接口，并通过动态代码的方式来创建这个接口实例并返回。
以下面接口为例，当调用listRepos时，动态代理会进行拦截，然后掉到invoke方法中。
```js
@GET("users/{user}/repos")
Call<List<String>> listRepos(@Path("user") String user);
```
invoke中主要做了3件事
1.loadServiceMethod():判断下当前serviceMethodCache中有没有指定的method，如果没有就创建个ServiceMethod对象。ServiceMethod这个类可以理解成网络参数（如GET还是POST，请求参数之类）、执行的封装对象。
2.初始化OKHttpCall
3.调用serviceMethod.adapt()方法
下面主要看一下第1，3点的源码:

**loadServiceMethod()**
```js
ServiceMethod<?, ?> loadServiceMethod(Method method) {
  ServiceMethod<?, ?> result = serviceMethodCache.get(method);
  if (result != null) return result;

  synchronized (serviceMethodCache) {
    result = serviceMethodCache.get(method);
    if (result == null) {
      result = new ServiceMethod.Builder<>(this, method).build();
      serviceMethodCache.put(method, result);
    }
  }
  return result;
}
```
判断下当前serviceMethodCache中有没有指定的method，如果没有就创建个ServiceMethod对象。再看一下**如何创建ServiceMethod对象**
```js
Builder(Retrofit retrofit, Method method) {
      this.retrofit = retrofit;
      this.method = method;
      this.methodAnnotations = method.getAnnotations();
      this.parameterTypes = method.getGenericParameterTypes();
      this.parameterAnnotationsArray = method.getParameterAnnotations();
    }

    public ServiceMethod build() {
      callAdapter = createCallAdapter();//这里返回的就是ExecutorCallAdapterFactory
      ...省略...
      //通过之前传入的gson转换器工厂创建gson的转换器
      responseConverter = createResponseConverter();

      for (Annotation annotation : methodAnnotations) {
        //就是对每个注解进行解析，将注解转换成网络请求所需的各种配置,
        //一开始的@POST和@Body便是在这个方法里面进行解析的
        parseMethodAnnotation(annotation);
      }

      ...省略...

      //将所有的注解归类组成一个ParameterHandler的数组
      int parameterCount = parameterAnnotationsArray.length;
      parameterHandlers = new ParameterHandler<?>[parameterCount];
      for (int p = 0; p < parameterCount; p++) {
        Type parameterType = parameterTypes[p];
        if (Utils.hasUnresolvableType(parameterType)) {
          throw parameterError(p, "Parameter type must not include a type variable or wildcard: %s",
              parameterType);
        }

        Annotation[] parameterAnnotations = parameterAnnotationsArray[p];
        if (parameterAnnotations == null) {
          throw parameterError(p, "No Retrofit annotation found.");
        }

        parameterHandlers[p] = parseParameter(p, parameterType, parameterAnnotations);
      }
      ...省略...

      return new ServiceMethod<>(this);
}
```
总之这一步主要做的是完成网络参数的解析和配置，为后面的执行和处理做准备。

**调用serviceMethod.adapt()方法**
```js
T adapt(Call<R> call) {
  return callAdapter.adapt(call);
}
```
这里的callAapter就是ExecutorCallAdapterFactory，看一下ExecutorCallAdapterFactory的adapt做了什么
```js
@Override public Call<Object> adapt(Call<Object> call) {
  return new ExecutorCallbackCall<>(callbackExecutor, call);
}
```
创建一个ExecutorCallbackCall对象，参数分别对应前面的MainThreadExecutor，和动态代理的invoke方法中创建的okHttpCall由此可以知道，Call<List<String>> repos = service.listRepos("octocat");repos实际就是ExecutorCallbackCall对象。

**调用repos.enqueue()方法**
开始请求接口实际是调用到了ExecutorCallbackCall的enqueue()方法中代码如下:
```js
@Override public void enqueue(final Callback<T> callback) {
  checkNotNull(callback, "callback == null");

  delegate.enqueue(new Callback<T>() {
    @Override public void onResponse(Call<T> call, final Response<T> response) {
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
          callback.onFailure(ExecutorCallbackCall.this, t);
        }
      });
    }
  });
}
```
其中callbackExecutor为MainThreadExecutor，delegate为动态代理的invoke方法中创建的okHttpCall对象，所以是通过okHttpCall完成了网络请求并返回数据结果，并通过
callbackExecutor.execute（）将线程切换回主线程，并将数据结果回调给最外层的接口
             