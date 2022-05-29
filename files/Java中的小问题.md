## Java中的小问题

### 1. try-resource的问题
1. catch只能catch到一个异常，但是如果打开了多个resource，应该怎么办？
	Throwable类增加了addSuppressed方法，supressed的异常会得到显示，在catch中通过getSuppressed方法也能获取这些异常。
2. 关注close方法的实现逻辑，如果close底层还有其他的资源需要关闭，这个资源应该显式地打开
```java
try (FileInputStream fin = new FileInputStream(new File("input.txt"));
		FileOutputStream fout = new FileOutputStream(new File("out.txt"));
		GZIPOutputStream out = new GZIPOutputStream(fout))
```
```java
```
而不是
```Java
try (FileInputStream fin = new FileInputStream(new File("input.txt"));
				GZIPOutputStream out = new GZIPOutputStream(new FileOutputStream(new File("out.txt"))))
```
否则如果fout的关闭产生了异常，就将得不到处理。
### 2. Java的泛型是如何实现的，什么是泛型擦除

Java的泛型是伪泛型，Java在**编译期**间，所有的泛型信息都会被擦掉

那么是如何检测类型的呢？

java编译器是通过先检查代码中泛型的类型，然后再进行类型擦除，在进行编译的。

产生什么问题？

泛型重写变重载（编译器自己生成**桥方法**来解决这个问题)，桥方法（参数是相当于被擦除了类型）调用coder本来想调用的那个参数是擦除前的类型的函数
静态不能用泛型，静态域是多个对象共享，所以不能确定泛型。

### 3. 
