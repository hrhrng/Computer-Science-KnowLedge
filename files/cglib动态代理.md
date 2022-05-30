### cglib动态代理
给出一个例子，再做分析
目标类
```java
public class AliSmsService {  
    public String send(String message) {  
        System.out.println("send message:" + message);  
        return message;  
    }  
}
```
实现一个 MothodInterceptor ，实现intercept方法，用做代理
```Java
public class DebugMethodInterceptor implements MethodInterceptor {  
  
    /**  
     * @param o           代理对象（增强的对象）  
     * @param method      被拦截的方法（需要增强的方法）  
     * @param args        方法入参  
     * @param methodProxy 用于调用原始方法 保存方法的一些信息  
     */  
    @Override  
    public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {  
        //调用方法之前，我们可以添加自己的操作  
        System.out.println("before method " + method.getName());  
        Object object = methodProxy.invokeSuper(o, args);  
        //调用方法之后，我们同样可以添加自己的操作  
        System.out.println("after method " + method.getName());  
        return object;  
    }  
  
}
```
工厂类获取代理对象
```Java
public class CglibProxyFactory {  
    public static Object getProxy(Class<?> clazz) {  
        // 创建动态代理增强类  
        Enhancer enhancer = new Enhancer();  
        // 设置类加载器  
        enhancer.setClassLoader(clazz.getClassLoader());  
        // 设置被代理类  
        enhancer.setSuperclass(clazz);  
        // 设置方法拦截器  
        enhancer.setCallback(new DebugMethodInterceptor());  
        // 创建代理类  
        return enhancer.create();  
    }  
}
```
在main中
```Java
public static void main(String[] args) {  
    // MethodProxy是怎么生成的  
    System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "D:\\cglibProxyClass");  
    AliSmsService aliSmsService = (AliSmsService) 
    CglibProxyFactory.getProxy(AliSmsService.class);  
    aliSmsService.send("java");  
}
```
首先看代理对象是怎么调用send的，通过
`System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, "D:\\cglibProxyClass");`
可以保存动态代理类，通过反编译工具打开代理类，可以看到代理类的send方法
```Java
public final String send(String var1) {  
    MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;  
    if (var10000 == null) {  
        CGLIB$BIND_CALLBACKS(this);  
        var10000 = this.CGLIB$CALLBACK_0;  
    }  
  
    return var10000 != null ? (String)var10000.intercept(this, CGLIB$send$0$Method, new Object[]{var1}, 
                                                     CGLIB$send$0$Proxy) : super.send(var1);  
}
```
可以看到，代理类中的send方法通过调用我们之前实现的MethodInterceptor的intercept方法来完成整个过程
在intercept中，会调用send方法的MethodProxy的invokeSuper方法来完成原始的方法调用，invokeSuper方法如下
```Java
public Object invokeSuper(Object obj, Object[] args) throws Throwable {  
    try {  
        this.init();  
        MethodProxy.FastClassInfo fci = this.fastClassInfo;  
        return fci.f2.invoke(fci.i2, obj, args);  
    } catch (InvocationTargetException var4) {  
        throw var4.getTargetException();  
    }  
}
```
cglib不同于JDK的动态代理使用反射的方式，它使用了一种叫FastClass的方式来执行原始方法，MothodProxy中保存了一些FastClass的信息和方法的index信息
```Java
private static class FastClassInfo {  
    FastClass f1;  
    FastClass f2;  
    int i1;  
    int i2;  
  
    private FastClassInfo() {  
    }  
}
```
f1是代理类的FastClass，f2代表目标类的FastClass，i1代表代理方法的index信息，i2代表原始方法的index信息
进入`fci.f2.invoke(fci.i2, obj, args)`
```Java
public Object invoke(int var1, Object var2, Object[] var3) throws InvocationTargetException {  
    9f3eb0c8 var10000 = (9f3eb0c8)var2;  
    int var10001 = var1;  
  
    try {  
        switch(var10001) {  
        case 0:  
            return new Boolean(var10000.equals(var3[0]));  
        //............省略一些选项
        case 18:  
            return var10000.CGLIB$clone$4();  
        case 19:  
            9f3eb0c8.CGLIB$STATICHOOK1();  
            return null;        
        case 20:  
            return var10000.CGLIB$send$0((String)var3[0]);  
        }  
    } catch (Throwable var4) {  
        throw new InvocationTargetException(var4);  
    }  
  
    throw new IllegalArgumentException("Cannot find matching method/constructor");  
}
```
invoke方法中通过i2选择执行相应的方法，send的i2为20，那么将执行
`var10000.CGLIB$send$0((String)var3[0]);` ,var10000为代理类，让我们看看代理类的`CGLIB$send$0`方法
```Java
final String CGLIB$send$0(String var1) {  
    return super.send(var1);  
}
```
此处调用父类的send方法，而代理类的父类即为目标类，就此调用原始方法成功。
***
再让我们来看看代理类是如何创建的，首先是`Enhancer enhancer = new Enhancer();`
```Java
public Enhancer() {  
    super(SOURCE);  
}
// Enhancer继承了AbstractClassGenerator
protected AbstractClassGenerator(AbstractClassGenerator.Source source) {  
    this.strategy = DefaultGeneratorStrategy.INSTANCE;  
    this.namingPolicy = DefaultNamingPolicy.INSTANCE;  
    this.useCache = DEFAULT_USE_CACHE;  
    this.source = source;  
}
```
构造函数主要是设置source，source就是this的ClassName，另外设置缓存等策略
***
构造完ClassGenerator（此处为Enhancer）后，执行`enhancer.setClassLoader(clazz.getClassLoader());`设置类加载器，之后再执行`enhancer.setSuperclass(clazz);`设置目标类，`enhancer.setCallback(new DebugMethodInterceptor());`设置拦截器，拦截器可以用`setCallbacks`方法来设置多个。
```Java
public void setClassLoader(ClassLoader classLoader) {  
    this.classLoader = classLoader;  
}
public void setSuperclass(Class superclass) {  
    if (superclass != null && superclass.isInterface()) {  
        this.setInterfaces(new Class[]{superclass});  
    } else if (superclass != null && superclass.equals(Object.class)) {  
        this.superclass = null;  
    } else {  
        this.superclass = superclass;  
    }  
  
}
public void setCallback(Callback callback) {  
    this.setCallbacks(new Callback[]{callback});  
}  
  
public void setCallbacks(Callback[] callbacks) {  
    if (callbacks != null && callbacks.length == 0) {  
        throw new IllegalArgumentException("Array cannot be empty");  
    } else {  
        this.callbacks = callbacks;  
    }  
}
```
在cglib中，ClassGenerator是可以复用的，每次生成代理类需要重新设置以上这些内容。
***
最后调用create方法创建对象，在Enhancer中，调用createHelper方法
```Java
public Object create() {  
    this.classOnly = false;  
    this.argumentTypes = null;  
    return this.createHelper();  
}
private Object createHelper() {  
    this.preValidate();  
    Object key = KEY_FACTORY.newInstance(this.superclass != null ? this.superclass.getName() : 
                null, ReflectUtils.getNames(this.interfaces), this.filter == ALL_ZERO ? null : 
                new WeakCacheKey(this.filter), this.callbackTypes, this.useFactory, 
                this.interceptDuringConstruction, this.serialVersionUID);  
    this.currentKey = key;  
    Object result = super.create(key);  
    return result;  
}
```
createHelper方法中,首先调用perValidate，检查我们定义的CallBack是否为已定义的Type，如果不属于任何一种，那么抛出异常（定义的拦截器必须实现给定的接口，这些接口都实现了Callback），不能只实现了Callback）
```Java
private void preValidate() {  
    if (this.callbackTypes == null) {  
        this.callbackTypes = CallbackInfo.determineTypes(this.callbacks, false);  
        this.validateCallbackTypes = true;  
    }  
  
    if (this.filter == null) {  
        if (this.callbackTypes.length > 1) {  
            throw new IllegalStateException("Multiple callback types possible but no filter 
                                             specified");  
        }  
  
        this.filter = ALL_ZERO;  
    }  
  
}
private static Type determineType(Class callbackType, boolean checkAll) {  
    Class cur = null;  
    Type type = null;  
  
    for(int i = 0; i < CALLBACKS.length; ++i) {  
        CallbackInfo info = CALLBACKS[i];  
        if (info.cls.isAssignableFrom(callbackType)) {  //存在相等或继承或继承
            if (cur != null) {  
                throw new IllegalStateException("Callback implements both " + cur + " and " + 
                                                 info.cls);  
            }  
  
            cur = info.cls;  
            type = info.type;  
            if (!checkAll) {  
                break;  
            }  
        }  
    }  
  
    if (cur == null) {  
        throw new IllegalStateException("Unknown callback type " + callbackType);  
    } else {  
        return type;  
    }  
}
```
```Java
static {  
    CALLBACKS = new CallbackInfo[]{
	    new CallbackInfo(NoOp.class, NoOpGenerator.INSTANCE),
	    new CallbackInfo(MethodInterceptor.class, MethodInterceptorGenerator.INSTANCE), 
	    new CallbackInfo(InvocationHandler.class, InvocationHandlerGenerator.INSTANCE), 
	    new CallbackInfo(LazyLoader.class, LazyLoaderGenerator.INSTANCE), 
	    new CallbackInfo(Dispatcher.class, DispatcherGenerator.INSTANCE), 
	    new CallbackInfo(FixedValue.class, FixedValueGenerator.INSTANCE), 
	    new CallbackInfo(ProxyRefDispatcher.class, DispatcherGenerator.PROXY_REF_INSTANCE)};  
}
```
通过调试，我们定义的拦截器和第二个CallbackInfo定义的MethodInterceptor存在相等或继承或实现关系，而事实也是如此，所以将返回MethodInterceptor的type，Type是ASM的一个工具类，返回后保存在Enhancer中，在此不深入了解。
***
接下类创建一个key，这个key用来找到目标类关联的一些信息，信息存放在一个生成的代理类中
```java
private final String FIELD_0;  
private final String[] FIELD_1;  
private final WeakCacheKey FIELD_2;  
private final Type[] FIELD_3;  
private final boolean FIELD_4;  
private final boolean FIELD_5;  
private final Long FIELD_6;
// 对应
KEY_FACTORY.newInstance(
this.superclass != null ? this.superclass.getName() : null, ReflectUtils.getNames(this.interfaces), 
this.filter == ALL_ZERO ? null : new WeakCacheKey(this.filter), this.callbackTypes, 
this.useFactory, 
this.interceptDuringConstruction, 
this.serialVersionUID);
```
***
之后将生成的key作为参数执行create方法。
``` Java
protected Object create(Object key) {  
    try {  
        ClassLoader loader = this.getClassLoader();  
        Map<ClassLoader, AbstractClassGenerator.ClassLoaderData> cache = CACHE;  
        AbstractClassGenerator.ClassLoaderData data = (AbstractClassGenerator.ClassLoaderData)cache.get(loader);  
        if (data == null) {  
            Class var5 = AbstractClassGenerator.class;  
            synchronized(AbstractClassGenerator.class) {  
                cache = CACHE;  
                data = (AbstractClassGenerator.ClassLoaderData)cache.get(loader);  
                if (data == null) {  
                    Map<ClassLoader, AbstractClassGenerator.ClassLoaderData> newCache = new WeakHashMap(cache);  
                    data = new AbstractClassGenerator.ClassLoaderData(loader);  
                    newCache.put(loader, data);  
                    CACHE = newCache;  
                }  
            }  
        }  
  
        this.key = key;  
        Object obj = data.get(this, this.getUseCache());  
        return obj instanceof Class ? this.firstInstance((Class)obj) : this.nextInstance(obj);  
    } catch (RuntimeException var9) {  
        throw var9;  
    } catch (Error var10) {  
        throw var10;  
    } catch (Exception var11) {  
        throw new CodeGenerationException(var11);  
    }  
}
```
在create中，首先利用缓存（用ClassLoader为key，存放信息）获取data，调用data的get方法。
```Java
// AbstractClassGenerator.ClassLoaderData
private final LoadingCache<AbstractClassGenerator, Object, Object> generatedClasses;
public Object get(AbstractClassGenerator gen, boolean useCache) {  
    if (!useCache) {  
        return gen.generate(this);  
    } else {  
        Object cachedValue = this.generatedClasses.get(gen);  
        return gen.unwrapCachedValue(cachedValue);  
    }  
}
```
get方法中，如果使用缓存，调用`generatedClasses.get(gen)`
```Java
// ClassLoaderData的构造方法中定义了LoadingCache的apply方法
public ClassLoaderData(ClassLoader classLoader) {  
    if (classLoader == null) {  
        throw new IllegalArgumentException("classLoader == null is not yet supported");  
    } else {  
        this.classLoader = new WeakReference(classLoader);  
	        Function<AbstractClassGenerator, Object> load = new Function<AbstractClassGenerator, Object>() 
            {  
            public Object apply(AbstractClassGenerator gen) {  
                Class klass = gen.generate(ClassLoaderData.this);  
                return gen.wrapCachedClass(klass); //   
            }  
        };  
        this.generatedClasses = new LoadingCache(GET_KEY, load);  
    }  
}
// ClassLoaderData中定义了keyMapper的apply方法，此处的key，即为create中生成的key，相当于目标类的一些信息
private static final Function<AbstractClassGenerator, Object> GET_KEY = new Function<AbstractClassGenerator, Object>() {  
    public Object apply(AbstractClassGenerator gen) {  
        return gen.key;  
    }  
};
// LoadingCache
protected final ConcurrentMap<KK, Object> map;
// 此处的key是指gen
public V get(K key) {  
    KK cacheKey = this.keyMapper.apply(key);  
    Object v = this.map.get(cacheKey);  
    return v != null && !(v instanceof FutureTask) ? v : this.createEntry(key, cacheKey, v);  
}
protected V createEntry(final K key, KK cacheKey, Object v) {  
    boolean creator = false;  
    FutureTask task;  
    Object result;  
    if (v != null) {  
        task = (FutureTask)v;  
    } else {  
        task = new FutureTask(new Callable<V>() {  
            public V call() throws Exception {  
                return LoadingCache.this.loader.apply(key);  
            }  
        });  
    //........
    // 主要是执行task，value就是task的返回，task的逻辑如下，可见类是在这里生成的
	public Object apply(AbstractClassGenerator gen) {  
		Class klass = gen.generate(ClassLoaderData.this); //此处的逻辑和不使用缓存一样 
		return gen.wrapCachedClass(klass);   
	}  
}
// 生成EnhancerFactoryData，存放类、构造器、构造器参数等信息
protected Object wrapCachedClass(Class klass) {  
    Class[] argumentTypes = this.argumentTypes;  
    if (argumentTypes == null) {  
        argumentTypes = Constants.EMPTY_CLASS_ARRAY;  
    }  
	  
    Enhancer.EnhancerFactoryData factoryData = new Enhancer.EnhancerFactoryData(klass, argumentTypes, this.classOnly);  
    Field factoryDataField = null;  
  
    try {  
        factoryDataField = klass.getField("CGLIB$FACTORY_DATA");  
        factoryDataField.set((Object)null, factoryData);  
        Field callbackFilterField = klass.getDeclaredField("CGLIB$CALLBACK_FILTER");  
        callbackFilterField.setAccessible(true);  
        callbackFilterField.set((Object)null, this.filter);  
    } catch (NoSuchFieldException var6) {  
        throw new CodeGenerationException(var6);  
    } catch (IllegalAccessException var7) {  
        throw new CodeGenerationException(var7);  
    }  
  
    return new WeakReference(factoryData);  
}
// 将EnhancerFactoryData转换为强引用
protected Object unwrapCachedValue(Object cached) {  
    if (this.currentKey instanceof Enhancer.EnhancerKey) {  
        Enhancer.EnhancerFactoryData data = (Enhancer.EnhancerFactoryData)((WeakReference)cached).get();  
        return data;  
    } else {  
        return super.unwrapCachedValue(cached);  
    }  
}
// super
protected Object unwrapCachedValue(T cached) {  
    return ((WeakReference)cached).get();  
}
// 结构Class对象
protected Object nextInstance(Object instance) {  
    Enhancer.EnhancerFactoryData data = (Enhancer.EnhancerFactoryData)instance;  
    if (this.classOnly) {  
        return data.generatedClass;  
    } else {  
        Class[] argumentTypes = this.argumentTypes;  
        Object[] arguments = this.arguments;  
        if (argumentTypes == null) {  
            argumentTypes = Constants.EMPTY_CLASS_ARRAY;  
            arguments = null;  
        }  
  
        return data.newInstance(argumentTypes, arguments, this.callbacks);  
    }  
}
public Object newInstance(Class[] argumentTypes, Object[] arguments, Callback[] callbacks) {  
    this.setThreadCallbacks(callbacks);  
  
    Object var4;  
    try {  
        if (this.primaryConstructorArgTypes != argumentTypes && !Arrays.equals(this.primaryConstructorArgTypes, argumentTypes)) {  
            var4 = ReflectUtils.newInstance(this.generatedClass, argumentTypes, arguments);  
            return var4;  
        }  
  
        var4 = ReflectUtils.newInstance(this.primaryConstructor, arguments);  
    } finally {  
        this.setThreadCallbacks((Callback[])null);  
    }  
  
    return var4;  
}
// 以上可以看出，主要是用构造器和参数来反射构造对象
// 再来看看如果没有缓存会怎么做
private Object createUsingReflection(Class type) {  
    setThreadCallbacks(type, this.callbacks);  
  
    Object var2;  
    try {  
        if (this.argumentTypes != null) {  
            var2 = ReflectUtils.newInstance(type, this.argumentTypes, this.arguments);  
            return var2;  
        }  
  
        var2 = ReflectUtils.newInstance(type);  
    } finally {  
        setThreadCallbacks(type, (Callback[])null);  
    }  
  
    return var2;  
}
// 直接使用class构造，而这个函数的底层还是会用到public static Object newInstance(Constructor cstruct, Object[] args)，缓存的作用相当提前获取了Constructor对象。
```
其中，获取cacheKey（目标类的信息），data内存放了一个ConcurrentHashMap，如果get(cacheKey)不为空，那么意味着已经生成过这个代理类，直接返回，否则就创建，创建过程省略（大概是生成一个代理类，两个FastClass类）。
值得注意的是，**LoadingCache中的map的value是一个弱引用**
### 一些启发
1. cglib中对WeakHashMap的应用：
	三层缓存结构,`WeakHashMap<K,KK>`,`ConcurrentHashMap<K, V>`
	K的类型是ClassLoader
	KK的类型是ClassLoaderData，在ClassLoaderData中，对ClassLoader的引用是弱引用。
	ClassLoaderData中的第二级缓存时LoadingCache，
	K的类型是一个动态生成的类，存放一些目标类的信息，没有对ClassLoader的引用
	V的值是EnhanceFactoryData，有对ClassLoader的引用，所以设置为弱引用。
	如果着两个地方对ClassLoader的引用不为弱引用的话，那么只要Enhancer存在，这个ClassLoader就得不到gc。
2. createEntry中的多线程调度
	1. 一级缓存	
	这里是用了一个双重检验锁的机制对临界进行保护，以cache是否被改变作为检验条件，而在单例模式中，检验的条件是对象是否被创建
	```java
	ClassLoader loader = this.getClassLoader();  
Map<ClassLoader, AbstractClassGenerator.ClassLoaderData> cache = CACHE;  
AbstractClassGenerator.ClassLoaderData data = (AbstractClassGenerator.ClassLoaderData)cache.get(loader);  
if (data == null) {  
    Class var5 = AbstractClassGenerator.class;  
    synchronized(AbstractClassGenerator.class) {  
        cache = CACHE;  
        data = (AbstractClassGenerator.ClassLoaderData)cache.get(loader);  
        if (data == null) {  
            Map<ClassLoader, AbstractClassGenerator.ClassLoaderData> newCache = new WeakHashMap(cache);  
            data = new AbstractClassGenerator.ClassLoaderData(loader);  
            newCache.put(loader, data);  
            CACHE = newCache;  
        }  
    }  
}
```
	2. 二级缓存
	二级缓存创建Entry时的处理也很有意思，传入的v有两种情况，一种为null，一种为Task。
	如果为null的话，创建任务，调用ConcurrentHashMap的putIfAbsent方法，这个方法保证了设置的线程安全性，
		如果result为null，说明放入成功，本线程成为任务的创建者，任务执行完，创建者将结果放入map
		如果不为空，取出Value，如果value是任务，等待任务完成返回结果，如果是结果，直接返回
	如果传入的v为Task，等待结果并返回。
	这里相当于做了一个异步调度，能产生插入表的只有creator一个人，其他的线程要知道creator何时插入表，就要有一个通知机制，这个通知机制就是task，线程阻塞再task.get(),当task完成后，线程得以继续运行。
```java
protected V createEntry(final K key, KK cacheKey, Object v) {  
    boolean creator = false;  
    FutureTask task;  
    Object result;  
    if (v != null) {  
        task = (FutureTask)v;  
    } else {  
        task = new FutureTask(new Callable<V>() {  
            public V call() throws Exception {  
                return LoadingCache.this.loader.apply(key);  
            }  
        });  
        result = this.map.putIfAbsent(cacheKey, task);  
        if (result == null) {  
            creator = true;  
            task.run();  
        } else {  
            if (!(result instanceof FutureTask)) {  
                return result;  
            }  
  
            task = (FutureTask)result;  
        }  
    }  
  
    try {  
        result = task.get();  
    } catch (InterruptedException var9) {  
        throw new IllegalStateException("Interrupted while loading cache item", var9);  
    } catch (ExecutionException var10) {  
        Throwable cause = var10.getCause();  
        if (cause instanceof RuntimeException) {  
            throw (RuntimeException)cause;  
        }  
  
        throw new IllegalStateException("Unable to load cache item", cause);  
    }  
  
    if (creator) {  
        this.map.put(cacheKey, result);  
    }  
  
    return result;  
}
```