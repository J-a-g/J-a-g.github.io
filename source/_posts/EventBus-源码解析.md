---
title: EventBus 源码解析
date: 2020-09-01 00:50:57
catalog:    true
header-img: "post-bg-unix-linux.jpg"
tags:
    - Java
    - Android
---

## 介绍

>[EventBus](https://github.com/greenrobot/EventBus)是Android上的以发布\订阅事件为核心的库。事件 (event) 通过 post() 发送到总线，然后再分发到匹配事件类型的订阅者 (subscribers) 。订阅者只有在总线中注册 (register) 了才能收到事件，注销 (unrigister) 之后就收不到任何事件了。事件方法必须带有 Subscribe 的注解，必须是 public ，没有返回类型 void 并且只能有一个参数。

## 使用

添加依赖到 Gradle：
```js
implementation 'org.greenrobot:eventbus:3.2.0'
```
1.定义事件
```js
public static class MessageEvent { /* Additional fields if needed */ }
```
2.准备订阅者
声明并用`Subscribe`注释修饰订阅方法
```js
@Subscribe(threadMode = ThreadMode.MAIN)  
public void onMessageEvent(MessageEvent event) {/* Do something */};
```
订阅者同时需要在总线上注册和注销自己。只有当订阅者注册了才能接收到事件。在Android中，通常与`Activity`和`Fragment`的生命周期绑定在一起。
```js
 @Override
 public void onStart() {
     super.onStart();
     EventBus.getDefault().register(this);
 }

 @Override
 public void onStop() {
     super.onStop();
     EventBus.getDefault().unregister(this);
 }
```
3.发送事件
```js
 EventBus.getDefault().post(new MessageEvent());
```

## 源码解析

源码解析会从创建对象、准备订阅者、注册、反注册、发送消息几个步骤进行解析

## 创建EventBus对象
先看看`getDefault()`
```js
    static volatile EventBus defaultInstance;
    /** Convenience singleton for apps using a process-wide EventBus instance. */
    public static EventBus getDefault() {
        EventBus instance = defaultInstance;
        if (instance == null) {
            synchronized (EventBus.class) {
                instance = EventBus.defaultInstance;
                if (instance == null) {
                    instance = EventBus.defaultInstance = new EventBus();
                }
            }
        }
        return instance;
    }
```
单例设计模式，用到了double check。再看看`EventBus`的构造方法：
```js
   /**
     * Creates a new EventBus instance; each instance is a separate scope in which events are delivered. To use a
     * central bus, consider {@link #getDefault()}.
     */
    public EventBus() {
        this(DEFAULT_BUILDER);
    }
```
虽然采用了单例设计模式，但是构造函数还是`public`，这样的设计是因为不仅仅可以只有一条总线，还可以有其他的线 (bus) ，订阅者可以注册到不同的线上的`EventBus`，通过不同的`EventBus`实例来发送数据，不同的`EventBus`是相互隔离开的，订阅者都只会收到注册到该线上事件。

再看看这个 this(DEFAULT_BUILDER) :
```js
    EventBus(EventBusBuilder builder) {
        logger = builder.getLogger();

        //维护3个重要的Map
        //key为event，value为subscriber列表，这个map就是这个事件有多少的订阅者，也就是事件对应的订阅者
        //key为订阅方法的参数类型，value为Subscription列表，这个map就是这个事件有多少的订阅者，也就是事件对应的订阅者
        subscriptionsByEventType = new HashMap<>();
        //key为subscriber，value为event列表，这个map就是这个订阅者有多少的事件，也就是订阅者订阅的事件列表
        //key为订阅类，value为订阅类中订阅方法参数类型集合，这个map就是这个订阅者有多少的事件，也就是订阅者订阅的事件列表
        typesBySubscriber = new HashMap<>();
        //粘性事件
        stickyEvents = new ConcurrentHashMap<>();

        mainThreadSupport = builder.getMainThreadSupport();
        //MainThread的poster
        mainThreadPoster = mainThreadSupport != null ? mainThreadSupport.createPoster(this) : null;
        //Backgroud的poster
        backgroundPoster = new BackgroundPoster(this);
        //Async的poster
        //订阅者方法寻找类，默认情况下参数是(null, false, false)
        asyncPoster = new AsyncPoster(this);
        indexCount = builder.subscriberInfoIndexes != null ? builder.subscriberInfoIndexes.size() : 0;
        subscriberMethodFinder = new SubscriberMethodFinder(builder.subscriberInfoIndexes,
                builder.strictMethodVerification, builder.ignoreGeneratedIndex);
        logSubscriberExceptions = builder.logSubscriberExceptions;
        logNoSubscriberMessages = builder.logNoSubscriberMessages;
        sendSubscriberExceptionEvent = builder.sendSubscriberExceptionEvent;
        sendNoSubscriberEvent = builder.sendNoSubscriberEvent;
        throwSubscriberException = builder.throwSubscriberException;
        eventInheritance = builder.eventInheritance;
        executorService = builder.executorService;
    }
```
说明一下上面的3个Map:
**subscriptionsByEventType：**是以`event`为key，`subscriber`列表为value，当发送`event`的时候，都是去这里找对应的订阅者。
**typesBySubscriber：**是以`subscriber`为key，`event`列表为`value`，当`register()`和`unregister()`的时候都是操作这个`map`，同时对`subscriptionsByEventType`进行对用操作。
**stickyEvents：**维护的是粘性事件，粘性事件也就是当`event`发送出去之后再注册粘性事件的话，该粘性事件也能收到之前发送出去的`event`。

这里运用到了`builder`设计模式，那么来看看这个`Builder`中有哪些参数：
```js
public class EventBusBuilder {
    //线程池
    private final static ExecutorService DEFAULT_EXECUTOR_SERVICE = Executors.newCachedThreadPool();
    //当调用事件处理函数异常时是否打印异常信息
    boolean logSubscriberExceptions = true;
    //当没有订阅者订阅该事件时是否打印日志
    boolean logNoSubscriberMessages = true;
    //当调用事件处理函数异常时是否发送 SubscriberExceptionEvent 事件
    boolean sendSubscriberExceptionEvent = true;
    //当没有事件处理函数对事件处理时是否发送 NoSubscriberEvent 事件
    boolean sendNoSubscriberEvent = true;
    //是否要抛出异常，建议debug开启
    boolean throwSubscriberException;
    //与event有继承关系的是否需要发送
    boolean eventInheritance = true;
    ////是否忽略生成的索引(SubscriberInfoIndex)
    boolean ignoreGeneratedIndex;
    //是否严格的方法名校验
    boolean strictMethodVerification;
    //线程池，async 和 background 的事件会用到
    ExecutorService executorService = DEFAULT_EXECUTOR_SERVICE;
    //当注册的时候会进行方法名的校验(EventBus3之前方法名必须以onEvent开头)，而这个列表是不参加校验的类的列表(EventBus3之后就没用这个参数了)
    List<Class<?>> skipMethodVerificationForClasses;
    //维护着由EventBus生成的索引(SubscriberInfoIndex)
    List<SubscriberInfoIndex> subscriberInfoIndexes;
    Logger logger;
    MainThreadSupport mainThreadSupport;

    EventBusBuilder() {
    }
```

## 准备订阅者
先看一下`Subscribe`注释
```js
@Documented
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.METHOD})
public @interface Subscribe {
    ThreadMode threadMode() default ThreadMode.POSTING;

    /**
     * If true, delivers the most recent sticky event (posted with
     * {@link EventBus#postSticky(Object)}) to this subscriber (if event available).
     */
    boolean sticky() default false;

    /** Subscriber priority to influence the order of event delivery.
     * Within the same delivery thread ({@link ThreadMode}), higher priority subscribers will receive events before
     * others with a lower priority. The default priority is 0. Note: the priority does *NOT* affect the order of
     * delivery among subscribers with different {@link ThreadMode}s! */
    int priority() default 0;
}
```
`ThreadMode`是枚举：
```js
public enum ThreadMode {
    POSTING,//post的时候是哪个线程订阅者就在哪个线程接收到事件
    MAIN,//订阅者在主线程接收到事件
    BACKGROUND,//订阅者在主线程接收到消息，如果post的时候不是在主线程的话，那么订阅者会在post的时候那个线程接收到事件。适合密集或者耗时少的事件。
    ASYNC//订阅者会在不同的子线程中收到事件。适合操作耗时的事件。
}
```

## 注册订阅者
先看代码，这部分代码比较少,但是分析时也分开两个步骤进行分析
```js
public void register(Object subscriber) {
        //拿到订阅者的class
        Class<?> subscriberClass = subscriber.getClass();
        //通过class去找到订阅方法
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        //blabla...
    }
```
这里主要是subscriberMethodFinder.findSubscriberMethods(subscriberClass)，看一下代码：
```js
List<SubscriberMethod> findSubscriberMethods(Class<?> subscriberClass) {
        //先从METHOD_CACHE中查看是否已经有这个订阅者了
        List<SubscriberMethod> subscriberMethods = METHOD_CACHE.get(subscriberClass);
        //有了的话直接把订阅者的方法返回
        if (subscriberMethods != null) {
            return subscriberMethods;
        }
        //ignoreGeneratedIndex默认是false
        if (ignoreGeneratedIndex) {
            subscriberMethods = findUsingReflection(subscriberClass);
        } else {
            subscriberMethods = findUsingInfo(subscriberClass);
        }
        if (subscriberMethods.isEmpty()) {
            throw new EventBusException("Subscriber " + subscriberClass
                    + " and its super classes have no public methods with the @Subscribe annotation");
        } else {
            //放到缓存当中
            METHOD_CACHE.put(subscriberClass, subscriberMethods);
            return subscriberMethods;
        }
    }
```
在使用`EventBus.getDefault()`时用的是默认的`builder`，而当使用`EventBusBuilder.installDefaultEventBus()`时是设置自己可配置的`builder`。这里我们先讨论默认情况下的。所以应该走到`findUsingInfo(subscriberClass)`方法来。
```js
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        //1.得到一个FindState对象
        FindState findState = prepareFindState();
        //blabla..
    }
```
看一下`FindState`是什么，`prepareFindState()`做了什么 
```js
static class FindState {
        //订阅者的方法的列表
        final List<SubscriberMethod> subscriberMethods = new ArrayList<>();
        //以EventType为key，method为value
        final Map<Class, Object> anyMethodByEventType = new HashMap<>();
        //以method的名字生成一个methodKey为key，该method的类(订阅者)为value
        final Map<String, Class> subscriberClassByMethodKey = new HashMap<>();
        //构建methodKey的StringBuilder
        final StringBuilder methodKeyBuilder = new StringBuilder(128);
        //订阅者
        Class<?> subscriberClass;
        //当前类
        Class<?> clazz;
        //是否跳过父类
        boolean skipSuperClasses;
        SubscriberInfo subscriberInfo;

        void initForSubscriber(Class<?> subscriberClass) {
            this.subscriberClass = clazz = subscriberClass;
            skipSuperClasses = false;
            subscriberInfo = null;
        }
```
```js
 //FindState复用池大小
    private static final int POOL_SIZE = 4;
    //FindState复用池
    private static final FindState[] FIND_STATE_POOL = new FindState[POOL_SIZE];

    private FindState prepareFindState() {
        synchronized (FIND_STATE_POOL) {
            //遍历复用池
            for (int i = 0; i < POOL_SIZE; i++) {
                FindState state = FIND_STATE_POOL[i];
                if (state != null) {
                    FIND_STATE_POOL[i] = null;
                    return state;
                }
            }
        }
        return new FindState();
    }
```
就是从一个复用池中取`FindState`对象并返回，继续`findUsingInfo(subscriberClass)`的流程：
```js
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        //1.得到一个FindState对象
        FindState findState = prepareFindState();
        //2.subscriberClass赋值给findState
        findState.initForSubscriber(subscriberClass);
        //2.findState的当前class不为null
        while (findState.clazz != null) {
            //2.默认情况下，getSubscriberInfo()返回的是null
            findState.subscriberInfo = getSubscriberInfo(findState);
            //2.那么这个if判断就跳过了
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                //2.来到了这里,这个方法重要，在这个方法中找到了哪些是订阅者订阅的方法和事件：
                findUsingReflectionInSingleClass(findState);
            }
            //...
           
        }
        //...
    }
```
`findUsingReflectionInSingleClass()`很重要，在这个方法中找到了哪些是订阅者订阅的方法和事件：
```js
    //在这个方法中找到了哪些是订阅者订阅的方法和事件
    private void findUsingReflectionInSingleClass(FindState findState) {
        Method[] methods;
        //通过反射，获取到订阅者的所有方法
        try {
            // This is faster than getMethods, especially when subscribers are fat classes like Activities
            methods = findState.clazz.getDeclaredMethods();
        } catch (Throwable th) {
            // Workaround for java.lang.NoClassDefFoundError, see https://github.com/greenrobot/EventBus/issues/149
            try {
                methods = findState.clazz.getMethods();
            } catch (LinkageError error) { // super class of NoClassDefFoundError to be a bit more broad...
                String msg = "Could not inspect methods of " + findState.clazz.getName();
                if (ignoreGeneratedIndex) {
                    msg += ". Please consider using EventBus annotation processor to avoid reflection.";
                } else {
                    msg += ". Please make this class visible to EventBus annotation processor to avoid reflection.";
                }
                throw new EventBusException(msg, error);
            }
            findState.skipSuperClasses = true;
        }
        for (Method method : methods) {
            //拿到修饰符(public 、private等)
            int modifiers = method.getModifiers();
            //判断是否是public，是否有需要忽略修饰符
            if ((modifiers & Modifier.PUBLIC) != 0 && (modifiers & MODIFIERS_IGNORE) == 0) {
                //获得方法的参数
                Class<?>[] parameterTypes = method.getParameterTypes();
                //EventBus只允许订阅方法后面的订阅事件是一个
                if (parameterTypes.length == 1) {
                    //判断该方法是不是被Subcribe的注解修饰着的
                    Subscribe subscribeAnnotation = method.getAnnotation(Subscribe.class);
                    //确定这是一个订阅方法
                    if (subscribeAnnotation != null) {
                        Class<?> eventType = parameterTypes[0];
                        if (findState.checkAdd(method, eventType)) {
                            //通过Annotation去拿一些数据
                            ThreadMode threadMode = subscribeAnnotation.threadMode();
                            //添加到subscriberMethods中
                            findState.subscriberMethods.add(new SubscriberMethod(method, eventType, threadMode,
                                    subscribeAnnotation.priority(), subscribeAnnotation.sticky()));
                        }
                    }
                } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                    String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                    throw new EventBusException("@Subscribe method " + methodName +
                            "must have exactly 1 parameter but has " + parameterTypes.length);
                }
            } else if (strictMethodVerification && method.isAnnotationPresent(Subscribe.class)) {
                String methodName = method.getDeclaringClass().getName() + "." + method.getName();
                throw new EventBusException(methodName +
                        " is a illegal @Subscribe method: must be public, non-static, and non-abstract");
            }
        }
    }

```
该方法流程是：
1.拿到当前`class`的所有方法
2.过滤掉不是`public`和是`abstract、static、bridge、synthetic`的方法
3.过滤出方法参数只有一个的方法
4.过滤出被`Subscribe注解`修饰的方法
5.将`method方法`和`event事件`添加到FindState中
6.将EventBus关心的`method方法、event事件、threadMode、priority、sticky`封装成`SubscriberMethod`对象添加到findState.subscriberMethods列表中

那么再来看看`findState.checkAdd(method, eventType)`
```js
/**
 *
 * @param method 订阅方法的Method对象
 * @param eventType 订阅事件的类型
 * @return true:该订阅方法允许添加至findState.subscriberMethods中   
 *             false:该订阅方法不需添加
 */
boolean checkAdd(Method method, Class<?> eventType) {
    // Map<Class, Object> anyMethodByEventType，它是FindState的成员变量
    // 这里以订阅的事件类型为key,订阅方法的Method对象为value put进此Map
    Object existing = anyMethodByEventType.put(eventType, method);
    if (existing == null) {
        // 若existing为null表示从未订阅过此类型的事件,当然直接返回true
        // 一级检查
        return true;
    } else {
        // 到这里说明anyMethodByEventType中存在了key为当前订阅事件(eventType)的值
        // existing = Map中以前eventType对应的value
        // 二级检查
        if (existing instanceof Method) {
            // 只会在第一次遇到相同事件类型时先调用checkAddWithMethodSignature
            if (!checkAddWithMethodSignature((Method) existing, eventType)) {
                throw new IllegalStateException();
            }
            // 将该事件类型对应的value置位非Method类型
            anyMethodByEventType.put(eventType, this);
        }
        //检查订阅方法是否需要解析为SubscriberMethod对象并添加至集合
        return checkAddWithMethodSignature(method, eventType);
    }
}
```
此方法分为了两级检查，首先会检查订阅事件类型在`anyMethodByEventType`是否存在了，如果不存在，则直接允许将该订阅方法添加至`findState.subscriberMethods`中，如果存在，则调用`checkAddWithMethodSignature()`进行方法签名的检查。为什么需要两级呢？因为大多数注册类中的订阅方法，一般都不会同时订阅相同类型的事件，此时便无需调用`checkAddWithMethodSignature()`了。只有当注册类中的多个订阅方法同时订阅了同一事件类型时，才需要进一步验证。一级检查很简单，所以我们重点来看看二级检查：`checkAddWithMethodSignature()`方法：
```js
private boolean checkAddWithMethodSignature(Method method, Class<?> eventType) {
    // 清空methodKeyBuilder中的内容
    methodKeyBuilder.setLength(0);
    // 将方法的名字添加到methodKeyBuilder中
    methodKeyBuilder.append(method.getName());
    // 将 > 符号 和 订阅事件类型全限定名点击到methodKeyBuilder中
    methodKeyBuilder.append('>').append(eventType.getName());
    // 假设订阅方法:public void methodName(String name);
    // 此时methodKey是这样的格式:methodName>java.lang.String
    String methodKey = methodKeyBuilder.toString();
    // 获取订阅方法所在类的类名 例如:class MainActivity
    Class<?> methodClass = method.getDeclaringClass();
    // Map<String, Class> subscriberClassByMethodKey，它是FindState的成员
    // key是methodKey(方法名+ > + 参数类型),value是订阅方法所在类的类名
    // methodClassOld是以前在subscriberClassByMethodKey中key为methodKey的value,
    // 如果没有则为null
    Class<?> methodClassOld = subscriberClassByMethodKey.put(methodKey, methodClass);
    // methodClassOld == null 这个好理解,表示此方法签名从来没存在过,
    // 那当然是返回true
    // 后面一个条件是什么鬼?它判断methodClassOld是否是methodClass自己或它的父类
    if (methodClassOld == null || methodClassOld.isAssignableFrom(methodClass)) {
        return true;
    } else {
        subscriberClassByMethodKey.put(methodKey, methodClassOld);
        return false;
    }
}
```
这两级检查主要就是为了不能出现一个订阅者有多个方法订阅的是同一事件的情况。现在回过头来看一下`SubscriberMethod`这个类：
```js
public class SubscriberMethod {
    final Method method;
    final ThreadMode threadMode;
    final Class<?> eventType;
    final int priority;
    final boolean sticky;
    /** Used for efficient comparison */
    String methodString;

    public SubscriberMethod(Method method, Class<?> eventType, ThreadMode threadMode, int priority, boolean sticky) {
        this.method = method;
        this.threadMode = threadMode;
        this.eventType = eventType;
        this.priority = priority;
        this.sticky = sticky;
    }
  	//blablabla......
}
```
`SubscriberMethod`类包括订阅类、事件类型、优先级、是否是粘性事件、ThreadMode，将EventBus所需要的全封装起来了。再回到`findUsingInfo(subscriberClass)`:
```js
private List<SubscriberMethod> findUsingInfo(Class<?> subscriberClass) {
        //1.得到一个FindState对象
        FindState findState = prepareFindState();
        //2.subscriberClass赋值给findState
        findState.initForSubscriber(subscriberClass);
        //2.findState的当前class不为null
        while (findState.clazz != null) {
            //2.默认情况下，getSubscriberInfo()返回的是null
            findState.subscriberInfo = getSubscriberInfo(findState);
            //2.那么这个if判断就跳过了
            if (findState.subscriberInfo != null) {
                SubscriberMethod[] array = findState.subscriberInfo.getSubscriberMethods();
                for (SubscriberMethod subscriberMethod : array) {
                    if (findState.checkAdd(subscriberMethod.method, subscriberMethod.eventType)) {
                        findState.subscriberMethods.add(subscriberMethod);
                    }
                }
            } else {
                //2.来到了这里,在这个方法中找到了哪些是订阅者订阅的方法和事件：
                findUsingReflectionInSingleClass(findState);
            }
            //3.将当前clazz变为该类的父类，然后再进行while循环的判断
            findState.moveToSuperclass();
        }
        //3.将findState释放资源，放回复用池中，返回封装好的SubscriberMethod列表
        return getMethodsAndRelease(findState);
    }
```
总结一下`subscriberClass`的流程：
&emsp;&emsp;1.从复用池中或者new一个，得到`findState`
&emsp;&emsp;2.将`subscriberClass`复制给`findState`
&emsp;&emsp;&emsp;&emsp;2.1.进入循环，判断当前`clazz`为不为`null`
&emsp;&emsp;&emsp;&emsp;2.2.不为`null`的话调用`findUsingReflectionInSingleClass()`方法得到该类的所有的`SubscriberMethod`
&emsp;&emsp;&emsp;&emsp;2.3.将`clazz`变为`clazz`的父类，再次进行循环的判断，再去检查父类中是否有订阅方法
&emsp;&emsp;3.返回所有的`SubscriberMethod`
现在再返回到了`findSubscriberMethods()`方法中，将所有的`SubscriberMethod`存储到`METHOD_CACHE`当中
```js
METHOD_CACHE.put(subscriberClass, subscriberMethods);
```
再回到`EventBus.register()`中：
```js
public void register(Object subscriber) {
        //拿到订阅者的class
        Class<?> subscriberClass = subscriber.getClass();
        //通过class去找到订阅方法
        List<SubscriberMethod> subscriberMethods = subscriberMethodFinder.findSubscriberMethods(subscriberClass);
        synchronized (this) {
            //遍历
            for (SubscriberMethod subscriberMethod : subscriberMethods) {
                //订阅
                subscribe(subscriber, subscriberMethod);
            }
        }
    }
```
`subscriberMethods`对象就是一个包含订阅类、事件类型、优先级、是否是粘性事件的集合，接下来遍历这个集合完成订阅。`subscribe(subscriber, subscriberMethod)`：
```js
   /**
     *
     * @param subscriber 注册类
     * @param subscriberMethod 订阅类、事件类型、优先级、是否是粘性事件、ThreadMode的封装对象
     */
private void subscribe(Object subscriber, SubscriberMethod subscriberMethod) {
    // 得到事件类型
    Class<?> eventType = subscriberMethod.eventType;
    // 创建Subscription对象,并将 注册类对象 和 订阅方法信息对象(subscriberMethod)保存进去
    Subscription newSubscription = new Subscription(subscriber, subscriberMethod);
    // subscriptionsByEventType:
    // key为事件类型
    // value为该事件类型对应的Subscription(注册类+订阅方法信息)集合
    // 取出eventType对应的List<Subscription>
    CopyOnWriteArrayList<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions == null) {
        // 如果eventType对应的List<Subscription>为null,则创建一个新集合
        subscriptions = new CopyOnWriteArrayList<>();
        // 以eventType为key,将新集合put到subscriptionsByEventType中
        subscriptionsByEventType.put(eventType, subscriptions);
    } else {
        // 若eventType对应的subscriptions(List<Subscription>)不为null
        // 判断subscriptions中是否存在了newSubscription
        // subscribe分析1
        if (subscriptions.contains(newSubscription)) {
            // 如果存在,抛出异常
            throw new EventBusException("Subscriber " + subscriber.getClass() + 
                                        " already registered to event " + eventType);
        }
    }
    int size = subscriptions.size();
    for (int i = 0; i <= size; i++) {
        if (i == size || subscriberMethod.priority >
            subscriptions.get(i).subscriberMethod.priority) {
            // 根据priority优先级,将newSubscription插入到subscriptions中合适的位置
            // (subscriptions中始终保持priority降序)
            subscriptions.add(i, newSubscription);
            break;
        }
    }
    // typesBySubscriber:key为注册类对象;value为该注册类中的事件类型集合
    // 根据注册类获取注册类中所有的事件类型
    List<Class<?>> subscribedEvents = typesBySubscriber.get(subscriber);
    if (subscribedEvents == null) {
        // 如果获取到的事件类型为null,则创建一个新集合
        subscribedEvents = new ArrayList<>();
        // 以注册类对象为key,新建的事件类型集合为value放进typesBySubscriber中
        typesBySubscriber.put(subscriber, subscribedEvents);
    }
    // 将eventType(当前订阅方法的订阅事件类型)加入到刚刚新建的集合中
    subscribedEvents.add(eventType);
    //如果是粘性事件
    if (subscriberMethod.sticky) {
            // eventInheritance还熟悉吧
            // false表示仅将事件对象发送给订阅了自己的订阅者
            // true表示还需要考虑发送给订阅了事件对象父类或接口的订阅者
            // 在postSingleEvent()中也用到了此变量判断
            if (eventInheritance) {
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    //是否有继承关系
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        // 如果eventType(订阅类型的Class对象)是粘性Map中取出的事件Class对象的父类
                        // 则也需要接收此事件
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                // 只关心事件对象本身类型就好
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }
}
```

## 注销订阅者

解除当前类中所有订阅方法对事件的订阅。现在我们来看看`unregister()`方法的具体实现：
```js
public synchronized void unregister(Object subscriber) {
    // 从typesBySubscriber根据注册类对象获取所有的订阅事件类型
    List<Class<?>> subscribedTypes = typesBySubscriber.get(subscriber);
    if (subscribedTypes != null) {
        // 如果订阅类型集合不为空,则遍历得到每一个订阅事件类型
        for (Class<?> eventType : subscribedTypes) {
            // 根据订阅事件类型去subscriptionsByEventType中移除
            // Subscription.subscriber与取消注册的subscriber相同的Subscription
            unsubscribeByEventType(subscriber, eventType);
        }
        // 从typesBySubscriber移除注册类中对应的所有订阅事件类型
        typesBySubscriber.remove(subscriber);
    } else {
        // 如果要解除注册的类中并不存在任何订阅,则打印此log
        logger.log(Level.WARNING, "Subscriber to unregister was not 
        registered before: " + subscriber.getClass());
    }
}
```
再看一下`unsubscribeByEventType(subscriber, eventType)`：
```js
private void unsubscribeByEventType(Object subscriber, Class<?> eventType) {
    // 根据订阅事件类型获取到所有的Subscription对象
    List<Subscription> subscriptions = subscriptionsByEventType.get(eventType);
    if (subscriptions != null) {
        int size = subscriptions.size();
        for (int i = 0; i < size; i++) {
            Subscription subscription = subscriptions.get(i);
            // 即订阅方法所在类的对象 与 解除注册类对象 相同
            if (subscription.subscriber == subscriber) {
                // 设为非活跃状态
                subscription.active = false;
                // 从List<Subscription>中移除
                subscriptions.remove(i);
                i--;
                size--;
            }
        }
    }
}
```
这个方法做的事情就是根据订阅事件类型获取到`List<Subscription>`，这个集合中的`Subscription`只保证订阅方法所订阅的事件类型是传入的`eventType`，但不能就这样把它干掉，还需要判断`Subscription`是否属于需要解除注册的类，`subscription.subscriber == subscriber`就是这个作用。一旦满足了这些条件，就可以放心的将`Subscription`从集合中移除了。前面那段代码则是从`typesBySubscriber`移除注册类中对应的所有订阅事件类型。

## 发送事件

通过`EventBus.post(event)`来发送事件
```js
public void post(Object event) {
  	//1.得到PostingThreadState
	PostingThreadState postingState = currentPostingThreadState.get();
	//blablabla......
}
```
`currentPostingThreadState`是一个`ThreadLocal`对象，而`ThreadLocal`是线程独有，不会与其他线程共享的。其实现是返回一个`PostingThreadState`对象，而`PostingThreadState`类的结构是：
```js
 /** For ThreadLocal, much faster to set (and get multiple values). */
    //PostingThreadState 封装的是当前线程的 post 信息，包括事件队列、是否正在分发中、是否在主线程、订阅者信息、事件实例、是否取消
    final static class PostingThreadState {
        final List<Object> eventQueue = new ArrayList<>();
        boolean isPosting;
        boolean isMainThread;
        Subscription subscription;
        Object event;
        boolean canceled;
    }
```
`PostingThreadState`封装的是当前线程的`post`信息，包括事件队列、是否正在分发中、是否在主线程、订阅者信息、事件实例、是否取消,回到`post()`方法：
```js
public void post(Object event) {
        //1.得到PostingThreadState
        PostingThreadState postingState = currentPostingThreadState.get();
        //2.获取其中的队列
        List<Object> eventQueue = postingState.eventQueue;
        //2.将该事件添加到队列中
        eventQueue.add(event);
        //2.如果postingState没有进行发送
        if (!postingState.isPosting) {
            //2. 判断当前线程是否是主线程
            postingState.isMainThread = isMainThread();
            //2.将isPosting状态改为true，表明正在发送中
            postingState.isPosting = true;
            //2.如果取消掉了，抛出异常
            if (postingState.canceled) {
                throw new EventBusException("Internal error. Abort state was not reset");
            }
            try {
                //2.循环，直至队列为空
                while (!eventQueue.isEmpty()) {
                    //2.发送事件
                    postSingleEvent(eventQueue.remove(0), postingState);
                }
            } finally {
                postingState.isPosting = false;
                postingState.isMainThread = false;
            }
        }
    }
```
最后走到一个`while`循环，判断事件队列是否为空了，如果不为空，继续循环，进行`postSingleEvent`操作，从事件队列中取出一个事件进行发送。
```js
private void postSingleEvent(Object event, PostingThreadState postingState) throws Error {
        Class<?> eventClass = event.getClass();
        boolean subscriptionFound = false;
        if (eventInheritance) {//是否查看事件的继承关系
            //找到事件的所以继承关系的事件类型
            List<Class<?>> eventTypes = lookupAllEventTypes(eventClass);// 就是查找该事件的所有父类
            int countTypes = eventTypes.size();
            for (int h = 0; h < countTypes; h++) {
                Class<?> clazz = eventTypes.get(h);
                //发送事件
                subscriptionFound |= postSingleEventForEventType(event, postingState, clazz);
            }
        } else {
            //直接发送事件
            subscriptionFound = postSingleEventForEventType(event, postingState, eventClass);
        }
        if (!subscriptionFound) {//如果没有任何事件
            if (logNoSubscriberMessages) {
                logger.log(Level.FINE, "No subscribers registered for event " + eventClass);
            }
            if (sendNoSubscriberEvent && eventClass != NoSubscriberEvent.class &&
                    eventClass != SubscriberExceptionEvent.class) {
                //发送一个NoSubscriberEvent的事件出去
                post(new NoSubscriberEvent(this, event));
            }
        }
    }
```
`lookupAllEventTypes()`就是查找该事件的所有父类，返回所有的该事件的父类的`class`。它通过循环和递归一起用，将一个类的父类（接口）全部添加到全局静态变量`eventTypes`集合中。再看一下`postSingleEventForEventType`方法：
```js
 private boolean postSingleEventForEventType(Object event, PostingThreadState postingState, Class<?> eventClass) {
        CopyOnWriteArrayList<Subscription> subscriptions;
        synchronized (this) {
            //所有订阅了event的事件集合
            subscriptions = subscriptionsByEventType.get(eventClass);
        }
        if (subscriptions != null && !subscriptions.isEmpty()) {
            for (Subscription subscription : subscriptions) {
                postingState.event = event;
                postingState.subscription = subscription;
                boolean aborted;
                try {
                    //这里调用的postToSubscription方法
                    postToSubscription(subscription, event, postingState.isMainThread);
                    aborted = postingState.canceled;
                } finally {
                    postingState.event = null;
                    postingState.subscription = null;
                    postingState.canceled = false;
                }
                if (aborted) {
                    break;
                }
            }
            return true;
        }
        return false;
    }
```
接下来看一下`postToSubscription`:
```js
private void postToSubscription(Subscription subscription, 
                                    Object event, boolean isMainThread) {
    // 获取订阅方法注解信息中的threadMode
    switch (subscription.subscriberMethod.threadMode) {
        // POSTING即发送事件在哪个线程,接收事件就在哪个线程
        case POSTING:
            // 因此,直接反射调用
            invokeSubscriber(subscription, event);
            break;
        // MAIN即发送事件时如果在主线程,则直接反射执行订阅方法
        // 否则交给mainThreadPoster处理
        case MAIN:
            if (isMainThread) {
                // 发送事件时处于主线程
                invokeSubscriber(subscription, event);
            } else {
                // 发送事件时不处于主线程
                mainThreadPoster.enqueue(subscription, event);
            }
            break;
        // MAIN_ORDERED即无论发送事件时处于哪个线程,都交由mainThreadPoster处理
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        // BACKGROUND即如果发送事件时处于主线程,则交由backgroundPoster处理
        // 否则发送事件时所处的线程,就是订阅方法接收事件时所处的线程
        case BACKGROUND:
            if (isMainThread) {
                backgroundPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
        // ASYNC即无论发送事件时处于哪个线程,都交由asyncPoster处理
        case ASYNC:
            asyncPoster.enqueue(subscription, event);
            break;
        default:
            throw new IllegalStateException("Unknown thread mode: " + 
                            subscription.subscriberMethod.threadMode);
    }
}
```
## 深入threadMode
**POSTING**
```js
case POSTING://当前线程直接调用
     invokeSubscriber(subscription, event);
break;


void invokeSubscriber(Subscription subscription, Object event) {
        try {
        	//通过反射机制实现方法调用
            subscription.subscriberMethod.method.invoke(subscription.subscriber, event);
        } catch (InvocationTargetException e) {
            handleSubscriberException(subscription, event, e.getCause());
        } catch (IllegalAccessException e) {
            throw new IllegalStateException("Unexpected exception", e);
        }
    }
```
看到这里，我们就可以得出这样一个结论：**POSTING不会做任何线程切换，接收事件时的线程 = 发送事件时所在的线程。**

**MAIN**
```js
// MAIN即发送事件时如果在主线程,则直接反射执行订阅方法
// 否则交给mainThreadPoster处理
case MAIN:
    if (isMainThread) {
        // 发送事件时处于主线程
        invokeSubscriber(subscription, event);
    } else {
        // 发送事件时不处于主线程
        mainThreadPoster.enqueue(subscription, event);
    }
break;
```
首先它判断了发送事件时的线程时否为主线程，如果是则直接反射调用订阅方法，如果不是，则执行了`mainThreadPoster.enqueue(subscription, event);`。
```js
public class HandlerPoster extends Handler implements Poster {
    // 维护着PendingPost(待发送事件与订阅者信息)的队列
    private final PendingPostQueue queue;
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
    // handleMessage是否正在处理PendingPost
    private boolean handlerActive;
    // HandlerPoster在EventBus创建时便会创建
    protected HandlerPoster(EventBus eventBus, Looper looper, 
                                    int maxMillisInsideHandleMessage) {
        // 调用了public Handler(Looper looper)
        super(looper);
        this.eventBus = eventBus;
        this.maxMillisInsideHandleMessage = maxMillisInsideHandleMessage;
        queue = new PendingPostQueue();
    }
    public void enqueue(Subscription subscription, Object event) {
        // 得到PendingPost对象
        PendingPost pendingPost = PendingPost.obtainPendingPost(subscription, event);
        synchronized (this) {
            // 将PendingPost对象加入到PendingPostQueue队列中
            queue.enqueue(pendingPost);
            if (!handlerActive) {
                // 将标记位设为true,表示handleMessage正在处理PendingPost
                handlerActive = true;
                // 发送一个Message至MessageQueue
                // 通过looper(MainLooper)使handleMessage被回调在主线程
                if (!sendMessage(obtainMessage())) {
                    throw new EventBusException("Could not send handler message");
                }
            }
        }
    }
    @Override
    public void handleMessage(Message msg) {
       //...
    }
}
```
在`enqueue()`方法中，首先通过`obtainPendingPost()`方法得到了`PendingPost`对象，下面我们先来看看`PendingPost`：
```js
final class PendingPost {
    // PendingPost对象缓存池
    private final static List<PendingPost> pendingPostPool = new ArrayList<PendingPost>();
    Object event;
    Subscription subscription;
    PendingPost next;
    private PendingPost(Object event, Subscription subscription) {
        this.event = event;
        this.subscription = subscription;
    }
    static PendingPost obtainPendingPost(Subscription subscription, Object event) {
        synchronized (pendingPostPool) {
            int size = pendingPostPool.size();
            if (size > 0) {
                // 若缓存池中有数据,则直接复用缓存池中的对象
                PendingPost pendingPost = pendingPostPool.remove(size - 1);
                pendingPost.event = event;
                pendingPost.subscription = subscription;
                pendingPost.next = null;
                return pendingPost;
            }
        }
        // 否则直接创建一个新的PendingPost对象
        return new PendingPost(event, subscription);
    }
    static void releasePendingPost(PendingPost pendingPost) {
        // 在使用完成后,对PendingPost对象字段置空
        pendingPost.event = null;
        pendingPost.subscription = null;
        pendingPost.next = null;
        synchronized (pendingPostPool) {
            // 防止缓存池无限制增长,因此最大不能超过10000
            if (pendingPostPool.size() < 10000) {
                // 将PendingPost对象放入缓存池
                pendingPostPool.add(pendingPost);
            }
        }
    }
}
```
简单吧，到这里我们发现`EventBus`中经常使用对象缓存池，之前的`FindState`，这里的`PendingPost`都是如此。
继续回到`HandlerPoster`的`enqueue()`方法中，得到了`PendingPost`对象后，会将`PendingPost`对象加入到`PendingPostQueue`队列中，下面我们一起来看一看`PendingPostQueue`的实现：
```js
package org.greenrobot.eventbus;
final class PendingPostQueue {
    // 队列头部引用
    private PendingPost head;
    // 队列尾部引用
    private PendingPost tail;
    synchronized void enqueue(PendingPost pendingPost) {
        if (pendingPost == null) {
            throw new NullPointerException("null cannot be enqueued");
        }
        if (tail != null) {
            // 若队列存在尾部,则将尾部的next引用指向当前pendingPost
            tail.next = pendingPost;
            // 将tail更新为当前pendingPost
            tail = pendingPost;
        } else if (head == null) {
            // 若不存在尾部,也不存在头部,则将当前pendingPost当做头部也当做尾部
            head = tail = pendingPost;
        } else {
            throw new IllegalStateException("Head present, but no tail");
        }
        notifyAll();
    }
    synchronized PendingPost poll() {
        // 队列先进先出,取出头部PendingPost对象
        PendingPost pendingPost = head;
        if (head != null) {
            head = head.next;
            if (head == null) {
                tail = null;
            }
        }
        return pendingPost;
    }
    synchronized PendingPost poll(int maxMillisToWait) throws InterruptedException {
        if (head == null) {
            wait(maxMillisToWait);
        }
        return poll();
    }
}
```
`PendingPostQueue`中包含了入队与出队的方法，都很简单。将`PendingPost`加入到队列中后，执行了`sendMessage(obtainMessage())`，这个时候，`HandlerPoster`中的`handleMessage()`方法就会被回调，又由于`looper`是`MainLooper`，因此它是被回调在主线程的，这样也就完成了线程切换：
```js
public class HandlerPoster extends Handler implements Poster {
    // 维护着PendingPost(待发送事件与订阅者信息)的队列
    private final PendingPostQueue queue;
    private final int maxMillisInsideHandleMessage;
    private final EventBus eventBus;
    // handleMessage是否正在处理PendingPost
    private boolean handlerActive;
    // HandlerPoster在EventBus创建时便会创建
    protected HandlerPoster(EventBus eventBus, Looper looper, 
                                int maxMillisInsideHandleMessage) {
        // ...
    }
    public void enqueue(Subscription subscription, Object event) {
        // ...
    }
    @Override
    public void handleMessage(Message msg) {
        boolean rescheduled = false;
        try {
            long started = SystemClock.uptimeMillis();
            // 不断从PendingPostQueue中取出PendingPost
            while (true) {
                PendingPost pendingPost = queue.poll();
                if (pendingPost == null) {
                    synchronized (this) {
                        // 再次尝试出列
                        pendingPost = queue.poll();
                        if (pendingPost == null) {
                            handlerActive = false;
                            return;
                        }
                    }
                }
                // 调用eventBus中的invokeSubscriber
                // 作用是回收pendingPost对象并反射调用订阅方法
                eventBus.invokeSubscriber(pendingPost);
                long timeInMethod = SystemClock.uptimeMillis() - started;
                // 当时间超出了maxMillisInsideHandleMessage(默认是10)
                // 会重新发一条消息
                // 至于有什么用,目前不太清楚.
                if (timeInMethod >= maxMillisInsideHandleMessage) {
                    if (!sendMessage(obtainMessage())) {
                        throw new EventBusException("Could not send handler message");
                    }
                    rescheduled = true;
                    return;
                }
            }
        } finally {
            handlerActive = rescheduled;
        }
    }
}
```
可以看到，在`handleMessage()`中会不停地从`PendingPostQueue`中取出`PendingPost`，交给`EventBus`的`invokeSubscriber`执行订阅方法的调用：
```js
void invokeSubscriber(PendingPost pendingPost) {
    // 事件对象
    Object event = pendingPost.event;
    // 订阅者信息对象
    Subscription subscription = pendingPost.subscription;
    // PendingPost成员置空
    PendingPost.releasePendingPost(pendingPost);
    if (subscription.active) {
        // 反射调用订阅方法
        invokeSubscriber(subscription, event);
    }
}
```
到这里，一切都结束了。这就是`EventBus`为何能够切换线程的奥秘所在。

**MAIN_ORDERED**
MAIN_ORDERED也就没什么可说的了，它只不过是无论发送事件时处于哪个线程，都会统一交由`mainThreadPoster`处理：
```js
// MAIN_ORDERED即无论发送事件时处于哪个线程,都交由mainThreadPoster处理
        case MAIN_ORDERED:
            if (mainThreadPoster != null) {
                mainThreadPoster.enqueue(subscription, event);
            } else {
                invokeSubscriber(subscription, event);
            }
            break;
```

**BACKGROUND与ASYNC**
有了上面的基础以后，大家可以自行查看BackgroundPoster.java和AsyncPoster.java，在这里简单说明一下：`BackgroundPoster`会开启一个新的线程来处理队列中的事件；`AsyncPoster`会为每个事件单独开启一个新的子线程来处理。

## EventBus的postSticky

```js
    public void postSticky(Object event) {
        synchronized (stickyEvents) {
            // 以事件Class对象为key,事件对象本身为value保存
            stickyEvents.put(event.getClass(), event);
        }
        // Should be posted after it is putted, in case the subscriber wants to remove immediately
        post(event);
    }
```
只不过在执行正常`post`流程之前，先将事件对象保存到了`stickyEvents`这个`Map`中。现在我们要回到`register()`方法看一下相关部分的代码：
```js
if (subscriberMethod.sticky) {
            // eventInheritance还熟悉吧
            // false表示仅将事件对象发送给订阅了自己的订阅者
            // true表示还需要考虑发送给订阅了事件对象父类或接口的订阅者
            // 在postSingleEvent()中也用到了此变量判断
            if (eventInheritance) {
                Set<Map.Entry<Class<?>, Object>> entries = stickyEvents.entrySet();
                for (Map.Entry<Class<?>, Object> entry : entries) {
                    Class<?> candidateEventType = entry.getKey();
                    //是否有继承关系
                    if (eventType.isAssignableFrom(candidateEventType)) {
                        Object stickyEvent = entry.getValue();
                        // 如果eventType(订阅类型的Class对象)是粘性Map中取出的事件Class对象的父类
                        // 则也需要接收此事件
                        checkPostStickyEventToSubscription(newSubscription, stickyEvent);
                    }
                }
            } else {
                // 只关心事件对象本身类型就好
                Object stickyEvent = stickyEvents.get(eventType);
                checkPostStickyEventToSubscription(newSubscription, stickyEvent);
            }
        }


private void checkPostStickyEventToSubscription(Subscription newSubscription, Object stickyEvent) {
        if (stickyEvent != null) {
            // If the subscriber is trying to abort the event, it will fail (event is not tracked in posting state)
            // --> Strange corner case, which we don't take care of here.
            postToSubscription(newSubscription, stickyEvent, isMainThread());
        }
    }
```

## 总结整体流程

**1. getDefault():** 
&emsp;&emsp;通过单例模式获取到EventBus对象，且在创建对象时完成初始化工作。其中两个重要的Map要注意：`subscriptionsByEventType、typesBySubscriber`。
&emsp;&emsp;**typesBySubscriber**:`key`值是订阅类，`value`值是该订阅类中所有订阅事件类型的`List`。
&emsp;&emsp;**subscriptionsByEventType**:`key`值是订阅事件类型(即订阅方法参数类型)，`value`值是`List<Subscription>`，其中`Subscription`是将订阅类和`SubscriberMethod`订阅方法封装类(包括Method、eventType、threadMode、sticky等)。
**2. regist():** 
&emsp;&emsp;**2.1.** 先从缓存中判断该注册类有该注册类的信息，有则返回`List<SubscriberMethod>`；若没有，则通过反射获取该类所有方法，并通过判断修饰符、参数个数（1个）、注释等条件确定订阅方法，然后解析订阅方法信息并封装成`SubscriberMethod`,如果有父类也对父类信息做如上处理，最后返回一个订阅类及订阅方法信息集合`List<SubscriberMethod>`（重要）。
&emsp;&emsp;**2.2.** 遍历上面的`List<SubscriberMethod>`，将订阅类、订阅方法、事件类型、即关键分装数据分别放入`subscriptionsByEventType、typesBySubscriber`中。
**3. post():** 
&emsp;&emsp;**3.1.** 先将发送的事件保存到队列中，再循环取出调用postSingleEvent
&emsp;&emsp;**3.2.** 调用postSingleEvent，开始发送事件，根据事件类型找出所有相关类（父类），开始遍历返回的事件类型集合调用postSingleEventForEventType，通过事件类型，找到
&emsp;&emsp;**3.3.** 调用postSingleEventForEventType，通过事件类型，从`subscriptionsByEventType`中找到`List<Subscription>`,然后遍历集合调用postToSubscription
&emsp;&emsp;**3.3.** 调用postToSubscription，先判断threadMode类型，再进行处理，如POSTING：则直接调用通过放射invoke调用到订阅方法；如MAIN，则先判断是否在主线程，如在主线程则直接调用通过放射invoke调用到订阅方法，如不在主线程，则先通过Handler完成线程切换，在调用invoke。
**4. unregist():**
&emsp;&emsp;先通过订阅类从`typesBySubscriber`中获取该订阅类中所有订阅事件类型的`List`，并遍历这个集合通过订阅事件类型来获取订阅类和`SubscriberMethod`订阅方法封装类集合B，再遍历集合B,判断是否和当前需要注销的订阅类相同，若相同则移除item，最后移除`typesBySubscriber`中的item。

## 思考

**如MainActivity extend BaseActivity，为什么EventBus.getDefault().register(this)方法无论写在父类还是子类的onCreate都可以生效？**
`EventBus.getDefault().register(this),无论写在父类还是子类的onCreate中其实是一样的，this实例都是MainActivity，所以两个都可以生效`

**如果父类中存在与子类相同的订阅方法，方法名、参数类型都相同，为什么父类中的订阅方法失效？**
`仔细阅读上文注册订阅者一栏，boolean checkAdd(Method method, Class<?> eventType)方法`

**两个订阅方法如onMessageEvent(Man event)和onMessageEvent(Preson event),其中Man是Person的子类，post（new Man()）,为什么两个订阅方法都会被调用到？**
`仔细阅读上午发送事件一栏， List<Class<?>> eventTypes = lookupAllEventTypes(eventClass)部分的代码`