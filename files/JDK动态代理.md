### JDK动态代理
首先说明，在JDK的动态代理中，实现类不是必须的，因此可以用来动态地生成实现类，这个实现类就是代理类，由于代理类继承于Proxy类，所以JDK只能用来代理接口。
### 例子
首先给出一个例子，再做分析
1. 接口及其实现类
```java
public interface SmsService {  
    String send(String message);  
}

public class SmsServiceImpl implements SmsService {  
    public String send(String message) {  
        System.out.println("send message:" + message);  
        return message;  
    }  
}
```
2. 实现`InvocationHandler`
```java
public class DebugInvocationHandler implements InvocationHandler {  
    /**  
     * 代理类中的真实对象  
     */  
    private final Object target;  
  
    public DebugInvocationHandler(Object target) {  
        this.target = target;  
    }  
  
  
    public Object invoke(Object proxy, Method method, Object[] args) throws InvocationTargetException, IllegalAccessException {  
        //调用方法之前，我们可以添加自己的操作  
        System.out.println("before method " + method.getName());  
        Object result = method.invoke(target, args);  
        //调用方法之后，我们同样可以添加自己的操作  
        System.out.println("after method " + method.getName());  
        return result;  
    }  
}
```
3. 工厂类获取代理对象
```java
public class ProxyFactory {  
    public static Object getProxy(Object target) {  
        return Proxy.newProxyInstance(  
                target.getClass().getClassLoader(), // 目标类的类加载  
                target.getClass().getInterfaces(),  // 代理需要实现的接口，可指定多个  
                new DebugInvocationHandler(target)   // 代理对象对应的自定义 InvocationHandler        );  
    }  
}
```
4. 设置保存代理类的参数，并且获取代理类，运行方法
```java
public static void main(String[] args) {  
    System.getProperties().put("jdk.proxy.ProxyGenerator.saveGeneratedFiles", "true");  
    SmsService smsService = (SmsService) ProxyFactory.getProxy(new SmsServiceImpl());  
    smsService.send("java");  
}
```
### 方法调用分析
1. 首先来看代理类的send方法。
```java
// $Proxy0.class
public final String send(String var1) {  
    try {  
        return (String)super.h.invoke(this, m3, new Object[]{var1});  
    } catch (RuntimeException | Error var2) {  
        throw var2;  
    } catch (Throwable var3) {  
        throw new UndeclaredThrowableException(var3);  
    }  
}
```
这里是调用了父类的h的invoke方法，父类为Proxy类，接下来来到Proxy类里。
***
2. 查看Proxy类的字段h，可以知道，h为我们定义的InvocationHandler，那么h的invoke就是我们自定义的invoke方法，在我们的例子中，首先用目标对象作为自定义的InvocationHandler构造的一部分，再在invoke方法中通过反射机制来调用原方法，这里的构造函数和invoke都是可以高度自定义的。
```java
// Proxy
protected InvocationHandler h;

protected Proxy(InvocationHandler h) {  
    Objects.requireNonNull(h);  
    this.h = h;  
}

// DebugInvocationHandler
public Object invoke(Object proxy, Method method, Object[] args) throws InvocationTargetException, IllegalAccessException {  
    //调用方法之前，我们可以添加自己的操作  
    System.out.println("before method " + method.getName());  
    Object result = method.invoke(target, args);  
    //调用方法之后，我们同样可以添加自己的操作  
    System.out.println("after method " + method.getName());  
    return result;  
}
```
***
3. 至此，调用链条结束。

### 如何生成代理类
1. 首先在工厂类的getProxy方法中调用Proxy的newProxyInstance方法，newProxyInstance方法如下，参数分别为

   `类加载器` `需要代理的接口` `自定义的InvocationHandler`
```java
// Proxy
public static Object newProxyInstance(ClassLoader loader,  
                                      Class<?>[] interfaces,  
                                      InvocationHandler h) {  
    Objects.requireNonNull(h);  
  
    final Class<?> caller = System.getSecurityManager() == null  
                                ? null  
                                : Reflection.getCallerClass();  
  
    /*  
     * Look up or generate the designated proxy class and its constructor.     */    Constructor<?> cons = 			getProxyConstructor(caller, loader, interfaces);  
  
    return newProxyInstance(caller, cons, h);  
}
```
2. 首先进入getProxyConstuctor方法，这个方法获取一个代理对象的单参数(InvocationHandler)的构造器类，
```java
// Proxy
private static Constructor<?> getProxyConstructor(Class<?> caller,  
                                                  ClassLoader loader,  
                                                  Class<?>... interfaces)  
{  
    // optimization for single interface  
    if (interfaces.length == 1) {  
        Class<?> intf = interfaces[0];  
        if (caller != null) {  
            checkProxyAccess(caller, loader, intf);  
        }  
        return proxyCache.sub(intf).computeIfAbsent(  
            loader,  
            (ld, clv) -> new ProxyBuilder(ld, clv.key()).build()  
        );  
    } else {  
        // interfaces cloned  
        final Class<?>[] intfsArray = interfaces.clone();  
        if (caller != null) {  
            checkProxyAccess(caller, loader, intfsArray);  
        }  
        final List<Class<?>> intfs = Arrays.asList(intfsArray);  
        return proxyCache.sub(intfs).computeIfAbsent(  
            loader,  
            (ld, clv) -> new ProxyBuilder(ld, clv.key()).build()  
        );  
    }  
}
```
这里会用到缓存，通过缓存来获取相应的构造器如果缓存不存在则生成一个构造器，而构造器的生成要通过生成代理对象class
```java
// Proxy
private static final ClassLoaderValue<Constructor<?>> proxyCache =  
    new ClassLoaderValue<>();

// ClassLoaderValue
// 这里的key是代理类要实现的interface
public <K> Sub<K> sub(K key) {  
    return new Sub<>(key);  
}
```
这里可以看出，proxyCache主要起了引出Sub的作用，Sub是AbstractClassLoaderValue的inner class和 subclass，可以无限嵌套，组成任意长度的key（多级缓存），ClassLoaderValue作为key只有一层。
在sub的computeIfAbsent方法中，首先取出ClassLoader内置的一个ConcurrentHashMap
然后构造mv，构造参数为clv（此sub，内部存放着key，即为接口）和 类加载器 和 mappingFuction
先以sub为key，mv为value，对类加载器执行putIfAbsent。
然后将执行mv.get操作，这一步就会通过一些机制通过接口和类加载器和mappingFuction获取构造器对象，即v
（get的底层会用到`new ProxyBuilder(ld, clv.key()).build()` 这条语句才是生成构造器的关键)
之后将map中以sub为key的entry的value替换成mv的v再返回，上层的程序就能获取到构造器对象了
这里的多线程操作也很值得思考

``` java
// Sub
Sub(K key) {  
    this.key = key;   // key是实现的接口
}
public V computeIfAbsent(ClassLoader cl,  
                         BiFunction<  
                             ? super ClassLoader,  
                             ? super CLV,  
                             ? extends V  
                             > mappingFunction) throws IllegalStateException {  
    ConcurrentHashMap<CLV, Object> map = map(cl);               // 这里的map是ClassLoader内置的 classLoaderValueMap
    @SuppressWarnings("unchecked")  
    CLV clv = (CLV) this;                                                // CLV泛型代表构造器类
    Memoizer<CLV, V> mv = null;                                       
    // 以下过程大致是存放一个以构造器类为键，Memozizer的v为Value的entry在类加载器内置的map中
    // Memozizer的v在Memozizer的get方法中生成
    while (true) {  
        Object val = (mv == null) ? map.get(clv) : map.putIfAbsent(clv, mv);  
        if (val == null) {  
            if (mv == null) {  
                // create Memoizer lazily when 1st needed and restart loop  
                mv = new Memoizer<>(cl, clv, mappingFunction);  
                continue;            }  
            // mv != null, therefore sv == null was a result of successful  
            // putIfAbsent            try {  
                // trigger Memoizer to compute the value  
                V v = mv.get();  
                // attempt to replace our Memoizer with the value  
                map.replace(clv, mv, v);  
                // return computed value  
                return v;  
            } catch (Throwable t) {  
                // our Memoizer has thrown, attempt to remove it  
                map.remove(clv, mv);  
                // propagate exception because it's from our Memoizer  
                throw t;  
            }  
        } else {  
            try {  
                return extractValue(val);  
            } catch (Memoizer.RecursiveInvocationException e) {  
                // propagate recursive attempts to calculate the same  
                // value as being calculated at the moment                throw e;  
            } catch (Throwable t) {  
                // don't propagate exceptions thrown from foreign Memoizer -  
                // pretend that there was no entry and retry                // (foreign computeIfAbsent invocation will try to remove it anyway)            }  
        }  
        // TODO:  
        // Thread.onSpinLoop(); // when available    }  
}

// Memozizer
// 一开始，v为null，inCall为false，生成的v主要看apply的操作，clv是Sub对象
public V get() throws RecursiveInvocationException {  
    V v = this.v;  
    if (v != null) return v;  
    Throwable t = this.t;  
    if (t == null) {  
        synchronized (this) {  
            if ((v = this.v) == null && (t = this.t) == null) {  
                if (inCall) {  
                    throw new RecursiveInvocationException();  
                }  
                inCall = true;  
                try {  
                    this.v = v = Objects.requireNonNull(  
                        mappingFunction.apply(cl, clv));  
                } catch (Throwable x) {  
                    this.t = t = x;  
                } finally {  
                    inCall = false;  
                }  
            }  
        }  
    }  
    if (v != null) return v;  
    if (t instanceof Error) {  
        throw (Error) t;  
    } else if (t instanceof RuntimeException) {  
        throw (RuntimeException) t;  
    } else {  
        throw new UndeclaredThrowableException(t);  
    }  
}

// 匿名类中apply的操作，clv是Sub，key是接口类对象
(ld, clv) -> new ProxyBuilder(ld, clv.key()).build()  


```
由上可以看出，如果没有缓存，最终完整的调用链条还是用类加载器和接口来构造一个构造器类，而中间的操作主要是操作缓存。
缓存利用的是ClassLoader内部的缓存，key为Sub对象，也就是一个AbstractClassLoaderValue对象，这个对象是Proxy类跟据要实现接口类型生成的，我们来看AbstractClassLoaderValue的equals方法,可以看出以Sub为key，就是以“要实现的接口和Proxy对象中的一个静态字段，以为静态字段总是固定）为key。value是一个构造器。
简单来说，整个缓存机制就是每个类加载器存储不同的ConcurrentHashmap，map以接口Class为key，以构造器对象为value。
而cglib中的缓存机制是，每个增强类存储一个WeakHashMap，以类加载器为key，data为value，这个第一层的map中只能有一个key，而data中有一个ConcurrentHashMap，以目标类的信息为key，以代理类的Class对象为value，value为弱引用

```java
public boolean equals(Object o) {  
    if (this == o) return true;  
    if (!(o instanceof Sub)) return false;  
    @SuppressWarnings("unchecked")  
    Sub<?> that = (Sub<?>) o;  
    return this.parent().equals(that.parent()) &&  
           Objects.equals(this.key, that.key);  
}
```
3. 获取Constrctor后，调用另一个newProxyInstance
```java
private static Object newProxyInstance(Class<?> caller, // null if no SecurityManager  
                                       Constructor<?> cons,  
                                       InvocationHandler h) {  
    /*  
     * Invoke its constructor with the designated invocation handler.     */    
    try {  
        if (caller != null) {  
            checkNewProxyPermission(caller, cons.getDeclaringClass());  
        }  
  
        return cons.newInstance(new Object[]{h});  
    } catch (IllegalAccessException | InstantiationException e) {  
        throw new InternalError(e.toString(), e);  
    } catch (InvocationTargetException e) {  
        Throwable t = e.getCause();  
        if (t instanceof RuntimeException) {  
            throw (RuntimeException) t;  
        } else {  
            throw new InternalError(t.toString(), t);  
        }  
    }  
}
```
这个方法，用Constructor和h来构造一个对象并且返回对象（反射机制）。
