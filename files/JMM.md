### final内存语义

1. **写**：构造函数内堆一个final域的写入，与随后把这个被构造函数对象的引用复制给一个引用，这两个操作之间不能重排序（而普通域可能会重排序）
    1. final写之后，return之前，构造一个 StoreStore屏障
2. **读**：初次读一个包含final域的对象的引用，与随后初次读这个final域，这两个操作之间不能重排序
    1. 在final读之前插入一个LoadLoad屏障
    
    构造函数中**写final** happen-before **对象引用**  happen-before **读final**
    
3. **对final修饰的引用类型**
    
    在构造函数内对一个final引用的对象的成员域的写入，与随后在构造函数外把这个被构造对象的引用复制给一个引用变量，这两个操作之间不能重排
    
4. 注意不能在构造函数中提前暴露本对象 (this) ，不然会导致内存语义被破坏

### happens-before

1. 为什么要定义happens-before
    1. 程序员需要易于理解、执行顺序可见的强内存模型
    2. 编译器需要性能更好、束缚更少的弱内存模型
    
    为了找到二者之间的平衡点，JMM提供了happens-before规则，一方面为程序员保证足够强的内存可见性，一方面对处理器的限制尽可能地放松
    
2. 规则
    1. 程序顺序规则：一个线程中的每个操作，hb于该线程中的任意后续操作（as-if-serial）
    2. 监视器锁规则：对一个锁的解锁，hb于随后对这个锁的加锁
    3. volatile变量规则：对一个volatile域的写，hb域对这个volatile变量的读
    4. 传递性
    5. start()规则：线程A执行操作ThreadB.start()并成功返回，线程A中的start()hb于线程B中的任意操作。
    6. join()规则：如果A执行操作ThreadB.join()并成功返回，线程B中的任意操作hb于A的后续操作。
    

类初始化的语义（具体JVM的实现不同，甚至能用双重锁检验实现，但是编码比较简洁）

只有一个线程能对类完成初始化,可以用来对静态字段进行延迟初始化（如单例模式）

```java
public class SingletonIniti {

        private SingletonIniti() {

        }

        private static class SingletonHolder {

                private static final SingletonIniti INSTANCE = newSingletonIniti();

         }

        public static SingletonIniti getInstance() {

                return SingletonHolder.INSTANCE;

        }

}
```

当getInstance方法第一次被调用的时候，它第一次读取SingletonHolder.instance，内部类SingletonHolder类得到初始化；而这个类在装载并被初始化的时候，会初始化它的静态域，从而创建Singleton的实例，由于是静态的域，因此只会在虚拟机装载类的时候初始化一次，并由虚拟机来保证它的线程安全性。

内存模型主要是针对重排序，JMM采用了折中的方案，提供了几个happen-before原则给程序员让他们获得强内存模型，其他的重排序不被保证，因具体的处理器而定。