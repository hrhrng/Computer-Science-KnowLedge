## 小问题

### 1. try-resource的问题
Throwable类新增了addSuppressed
我们可以看到，out变量实际上代表的是被装饰的FileOutputStream类。在调用out变量的close方法之前，GZIPOutputStream还做了finish操作，该操作还会继续往FileOutputStream中写压缩信息，此时如果出现异常，则会out.close()方法被略过，然而这个才是最底层的资源关闭方法。正确的做法是应该在try-with-resource中单独声明最底层的资源，保证对应的close方法一定能够被调用。在刚才的例子中，我们需要单独声明每个FileInputStream以及FileOutputStream： 

