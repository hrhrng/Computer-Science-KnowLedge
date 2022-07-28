## 实现LoadingCache

利用ConcurrentHashMap 和 FutureTask 实现一个简易的 LoadingCache ，并编写安全性测试

```java
package org.example;


import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.ConcurrentMap;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.FutureTask;
import java.util.function.Function;

public class SimpleLoadingCache<K,V> {

    ConcurrentMap<K,Object> map;
    Function<K, V> loader;

    public SimpleLoadingCache(Function<K, V> loader) {
        this.map = new ConcurrentHashMap<>();
        this.loader = loader;
    }


    public V get(K key) {
        Object value = map.get(key);
        // already loaded
        if(value != null && !(value instanceof FutureTask)) {
            return (V) value;
        }
        // task is not initial or task is running
        return compute(key, value);
    }

    private V compute(K key, Object v) {
        FutureTask<V> task;
        boolean creator = false;
        if(v != null) {
            task = (FutureTask<V>) v;
        } else {
            task = new FutureTask<V>(() -> loader.apply(key));
            // make sure
            Object prevTask = map.putIfAbsent(key, task);
            // this thread is the creator
            if (prevTask == null) {
               creator = true;
               task.run();
            }
            // task is already created by another thread
            else if (prevTask instanceof FutureTask) {
                task = (FutureTask<V>) prevTask;
            }
            // result has computed
            else {
                return (V) prevTask;
            }
        }

        V result;
        try {
            result = task.get();
        } catch (InterruptedException e) {
            throw new IllegalStateException("Interrupted while loading cache item", e);
        } catch (ExecutionException e) {
            Throwable cause = e.getCause();
            if (cause instanceof RuntimeException) {
                throw ((RuntimeException) cause);
            }
            throw new IllegalStateException("Unable to load cache item", cause);
        }
        // when the task is done, the creator will put the result to the map
        // so that those thread who were not involved in the competition can get the result
        if (creator) {
            map.put(key, result);
        }
        return result;
    }
}
```

```java
import org.example.SimpleLoadingCache;
import org.junit.Assert;
import org.junit.Rule;
import org.junit.Test;
import org.junit.rules.ExpectedException;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.BrokenBarrierException;
import java.util.concurrent.CyclicBarrier;
import java.util.concurrent.locks.ReentrantLock;


public class SimpleLoadingCacheTest {

    // 正确的语义是只有一个线程能运行loader


    // 验证其后验条件和不变性条件
    @Test
    public void testBase(){
        SimpleLoadingCache<String, String> loadingCache = new SimpleLoadingCache<>(k->{
            return String.valueOf(System.nanoTime());
        });
        String value1 = loadingCache.get("test");
        String value2 = loadingCache.get("test");
        Assert.assertTrue(value1 == value2);
    }

    // 测试阻塞行为
    @Test
    public void testBlocking() throws InterruptedException {

        // main thread and t2 will wait the latch until t1 notify the latch
        final Object latch = new Object();
        // lock will lock when the main thread block
        // when t2 find lock is locked, it will interrupt main thread
        ReentrantLock lock = new ReentrantLock();
        // main thread
        Thread main = Thread.currentThread();

        SimpleLoadingCache<String, String> loadingCache = new SimpleLoadingCache<>(k->{
            synchronized (latch) {
                latch.notifyAll();
            }
            for (int i = 0; i < 10000000; i++) {
            }
            return "x";
        });


        // t1 will be the creator
        Thread t1 = new Thread(()->{
            String value = loadingCache.get("key");
        });

        // t2 will interrupt the main thread
        Thread t2 = new Thread(()->{
            // waiting here until t1 is creating the Entry
            t1.start();
            synchronized (latch) {
                try {
                    latch.wait();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                }
            }
            // until t2 get the lock
            while (lock.tryLock()){
                lock.unlock();
            }
            main.interrupt();
        });

        t2.start();


        Assert.assertThrows("Interrupted while loading cache item",IllegalStateException.class,()->{
            synchronized (latch){
                latch.wait();
            }
            lock.lock();
            // t1 is creating the Entry, main will block here
            loadingCache.get("key");
            lock.unlock();
        });
    }

    // 验证安全性
    @Test
    public void testSafe() throws BrokenBarrierException, InterruptedException {
        SimpleLoadingCache<String, String> loadingCache = new SimpleLoadingCache<>(k->{
            return String.valueOf(System.nanoTime());
        });
        Map<Thread, String> map = new HashMap<>();
        CyclicBarrier barrier = new CyclicBarrier(100+1);
        for(int i = 0; i < 100; ++i) {
            new Thread(()->{
                try {
                    barrier.await();
                    String value = loadingCache.get("key");
                    map.put(Thread.currentThread(), value);
                    barrier.await();
                } catch (InterruptedException e) {
                    throw new RuntimeException(e);
                } catch (BrokenBarrierException e) {
                    throw new RuntimeException(e);
                }
            }).start();
        }
        barrier.await();
        barrier.await();
        Assert.assertTrue(map.values().stream().distinct().count() == 1);
    }

}
```