---
title: 一个线程安全的单例模式
date: 2017-12-05 10:30:21
thumbnail: /css/images/thread-safe-singleton.jpg
tags:
- 单例模式
- 线程安全
category:
- java
---


单例模式的一般构造方法

``` java
public class SingletonConsumer {
    private final static Logger logger = Logger.getLogger(SingletonConsumer.class);
    private static SingletonConsumer instance;
    private SingletonConsumer() {
    }
    public SingletonConsumer getInstance() {
        if (instance == null) {
            logger.debug("instance is null, trying to instantiate a new one");
            instance = new SingletonConsumer();
        } else {
            logger.debug("instance is not null, return the already-instantiated one");
        }
        return instance;
    }
}
```

以上这种构造方法在单线程下运行是安全的，但是如果放到多线程下，则会出现各种各样的问题，为此我们设计一个实验来验证多线程下，以上方法会出现什么问题。

# Experiment

实验中我们设置10个线程去创建SingletonConsumer实例，最后验证到底创建了多少个实例。

``` java 
@Test
  public void singletonConsumerTest() throws InterruptedException {
       ExecutorService executors = Executors.newFixedThreadPool(10);
       Set<SingletonConsumer> set = new HashSet<>();
       for(int i = 0; i < 10; i++){
           executors.execute(
                   () -> set.add(SingletonConsumer.getInstance())
           );
       }
       executors.shutdown();
       executors.awaitTermination(1, TimeUnit.HOURS);
       Assert.assertEquals(10, set.size());
   }
```

运行测试，输出结果如下

```
2017-02-26 13:50:52 DEBUG SingletonConsumer:20 - instance is null, trying to instantiate a new one
2017-02-26 13:50:52 DEBUG SingletonConsumer:20 - instance is null, trying to instantiate a new one
2017-02-26 13:50:52 DEBUG SingletonConsumer:20 - instance is null, trying to instantiate a new one
2017-02-26 13:50:52 DEBUG SingletonConsumer:20 - instance is null, trying to instantiate a new one
2017-02-26 13:50:52 DEBUG SingletonConsumer:20 - instance is null, trying to instantiate a new one
2017-02-26 13:50:52 DEBUG SingletonConsumer:20 - instance is null, trying to instantiate a new one
2017-02-26 13:50:52 DEBUG SingletonConsumer:20 - instance is null, trying to instantiate a new one
2017-02-26 13:50:52 DEBUG SingletonConsumer:20 - instance is null, trying to instantiate a new one
2017-02-26 13:50:52 DEBUG SingletonConsumer:20 - instance is null, trying to instantiate a new one
2017-02-26 13:50:52 DEBUG SingletonConsumer:20 - instance is null, trying to instantiate a new one
set size:10
```

会发现此时实际上构造了是个SingletonConsumer实例，那怎么才能构造线程安全的单例模式？
首先想到的方法是将getInstance的代码用synchronized包起来，这样就能够保证getInstance方法每次只能有一个线程访问到，于是代码就变成了
下面的样子

``` java
public static SingletonConsumer getInstance() {
       synchronized (SingletonConsumer.class) {
           if (instance == null) {
               logger.debug("instance is null, trying to instantiate a new one");
               instance = new SingletonConsumer();
           } else {
               logger.debug("instance is not null, return the already-instantiated one");
           }
           return instance;
       }
   }
```

我们再次运行测试脚本

```
2017-02-26 14:01:12 DEBUG SingletonConsumer:21 - instance is null, trying to instantiate a new one
2017-02-26 14:01:12 DEBUG SingletonConsumer:24 - instance is not null, return the already-instantiated one
2017-02-26 14:01:12 DEBUG SingletonConsumer:24 - instance is not null, return the already-instantiated one
2017-02-26 14:01:12 DEBUG SingletonConsumer:24 - instance is not null, return the already-instantiated one
2017-02-26 14:01:12 DEBUG SingletonConsumer:24 - instance is not null, return the already-instantiated one
2017-02-26 14:01:12 DEBUG SingletonConsumer:24 - instance is not null, return the already-instantiated one
2017-02-26 14:01:12 DEBUG SingletonConsumer:24 - instance is not null, return the already-instantiated one
2017-02-26 14:01:12 DEBUG SingletonConsumer:24 - instance is not null, return the already-instantiated one
2017-02-26 14:01:12 DEBUG SingletonConsumer:24 - instance is not null, return the already-instantiated one
2017-02-26 14:01:12 DEBUG SingletonConsumer:24 - instance is not null, return the already-instantiated one
java.lang.AssertionError: 
Expected :10
Actual   :1
  <Click to see difference>
```

此时发现，只初始化了一个SingletonConsumer实例，说明这种方法是work的。

但是仔细去想一想，上面的方法是有效率问题的。假设有一个线程A正在synchronized块中判断instance是否为null，此时其他线程只能等待线程A判断完毕才可以再去判断。仔细想想，instance是否为空，其实是可以多个线程同时去判断的，因此我们将代码修改成一下形式：

``` java 
if (instance == null) {
            synchronized (SingletonConsumer.class) {
                logger.debug("instance is null, trying to instantiate a new one");
                instance = new SingletonConsumer();
            }
        } else {
            logger.debug("instance is not null, return the already-instantiated one");
        }
        return instance;
```

上面的代码中，我们将instance是否为空的判断移到了同步块的外面。那这种方法是否是线程安全的呢，再次运行测试脚本，观察结果：

```
2017-02-26 14:14:18 DEBUG SingletonConsumer:28 - instance is null, trying to instantiate a new one
2017-02-26 14:14:18 DEBUG SingletonConsumer:28 - instance is null, trying to instantiate a new one
2017-02-26 14:14:18 DEBUG SingletonConsumer:28 - instance is null, trying to instantiate a new one
2017-02-26 14:14:18 DEBUG SingletonConsumer:28 - instance is null, trying to instantiate a new one
2017-02-26 14:14:18 DEBUG SingletonConsumer:28 - instance is null, trying to instantiate a new one
2017-02-26 14:14:18 DEBUG SingletonConsumer:28 - instance is null, trying to instantiate a new one
2017-02-26 14:14:18 DEBUG SingletonConsumer:28 - instance is null, trying to instantiate a new one
2017-02-26 14:14:18 DEBUG SingletonConsumer:28 - instance is null, trying to instantiate a new one
2017-02-26 14:14:18 DEBUG SingletonConsumer:28 - instance is null, trying to instantiate a new one
2017-02-26 14:14:18 DEBUG SingletonConsumer:28 - instance is null, trying to instantiate a new one
set size:10
```

通过结果发现，依然实例化了10个SingeltonCoumser。
考虑一种情况，初始install为空，线程A判断完instance是否为空，发现instance为null，刚好线程A的时间片用完，轮到线程B去判断instance是否为空，线程B发现instance也是null，此时时间片又回到了线程A的手中，线程A去创建SingletonConsumer对象，创建完成，线程B去创建对象，这样下去，就造成了上述实验的现象，因此，未解决上面的问题，需要带同步块中同样去判断instance是否为null。最后的代码如下：

``` java
public static SingletonConsumer getInstance() {
       if (instance == null) {
           synchronized (SingletonConsumer.class) {
               if (instance == null) {
                   logger.debug("instance is null, trying to instantiate a new one");
                   instance = new SingletonConsumer();
               }
               else {
                   logger.debug("instance is not null, return the already-instantiated one ");
               }
           }
       } else {
           logger.debug("instance is not null, return the already-instantiated one");
       }
       return instance;
   }
```

最有还有一点要注意，由于JVM会对代码进行优化，所以代码的执行顺序在真运行的时候会发生变化，会导致赋值操作编程不可见的，因此才进行赋值操作时，instance有可能只拿到一个为完全初始化的实例，这样会导致一些错误。

`instance = new SingletonConsumer();`

解决办法是将instance生命为volatile的，volatile关键词可以保证可见性和有序性，其具体内容待下次再表。

# Summary

总之，一个线程安全的单例模式需要注意以下3点：

1.getInstance的需要用synchronized关键词

2.为提高效率，instance是否为空可提到同步块以外，但内层的判断依然要保留

3.instance需要声明为volatile

代码见：[GitHub](https://github.com/llplay/muti-thread-learn)
