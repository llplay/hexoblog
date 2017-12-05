---
title: 生产者和消费者模型初探
date: 2017-12-04 19:25:11
tags:
- 多线程
- 生产者消费者模型
category:
- java
---

本文将尝试构造一个生产者，消费者模型，通过模型的构建，学习一下多线程编程。
代码见：[GitHub](https://github.com/llplay/muti-thread-learn)


# 1.生产者消费者模型

关于生产者消费者模型的基本含义不在赘述。本实验的拟构造一个生产者，从文件中按行读取数据，放入队列中，构建十个消费者，从队列中读取数据，将数据写回文件。

## 1.1 starter

首先我们构造一个方法，改方法负责初始化生产者和消费者线程


``` java
public static void startMutiTheads(String inputPath, String outPath) {
       LocalDateTime startTime = LocalDateTime.now();
       List<Thread> threads = new ArrayList<>();
       LinkedBlockingQueue<Optional<String>> queue = new LinkedBlockingQueue<>();
       FileUtil.clearFileContents(outPath);
       Producer producer = new Producer(inputPath, queue);
       threads.add(new Thread(producer));
       Consumer consumer = new Consumer(outPath, queue);
       for(int i = 0; i < Consumer.consumerThreadCount; i++){
           Thread consumerThread = new Thread(consumer);
           threads.add(consumerThread);
           producer.addConsumer(consumer);
       }
       threads.forEach(Thread :: start);
       threads.forEach(thread -> {
           try {
               thread.join();
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
       });
       consumer.getFileUtil().flushAndClose();
       // get consumer's totalCount:
       logger.debug("Consumer's totalCount: " + consumer.getTotalCount());
       LocalDateTime endTime = LocalDateTime.now();
       logger.info(String.format("It takes %s seconds to finish", 		LocalDateTime.from(startTime).until(endTime, ChronoUnit.SECONDS)));
       }
```

函数的有两个参数，分别表表示文件的输入路径和输出路径。
下面我们构建一个thread列表，该列表存储所有的线程实例。

``` java
LinkedBlockingQueue<String> queue = new LinkedBlockingQueue<>();
```

我们选用concurrent包中的LinkedBlockingQueue队列，生产者线程将内从从文件读出放至该队列，消费者线程从改队列读出数据写回到文件。

LinkedBlockingQueue实现是线程安全的，实现了先进先出等特性，可以指定容量，也可以不指定，不指定的话，默认最大是Integer.MAX_VALUE，其中主要用到put和take方法，put方法在队列满的时候会阻塞直到有队列成员被消费，take方法在队列空的时候会阻塞，直到有队列成员被放进来。

下面我们构建了一个生产者线程，并放到theads列表中

``` java
Producer producer = new Producer(inputPath, queue);
threads.add(new Thread(producer));
```

再之后构建十个消费者线程，其中Consumer.consumerThreadCount是在Counsmer中定义的一个静态变量，值为10

``` java
Consumer consumer = new Consumer(outPath, queue);
for(int i = 0; i < Consumer.consumerThreadCount; i++){
    Thread consumerThread = new Thread(consumer);
    threads.add(consumerThread);
    producer.addConsumer(consumer);
  }
```


下面需要做的就是启动线程

``` java
threads.forEach(Thread :: start);
threads.forEach(thread -> {
    try {
        thread.join();
        } catch (InterruptedException e) {
        e.printStackTrace();
        }
     });
```

thread.join()方法会一直等待，直到改线程结束。

以上就是starter需要做的。

## 1.2 Produce

本实验只设置了一个消费者模型，基本代码如下：

``` java 
public class Producer implements Runnable {
    private LinkedBlockingQueue<Optional<String>> queue;
    private String inputFile;
    public Producer(LinkedBlockingQueue<Optional<String>> queue, String inputFile) {
        this.queue = queue;
        this.inputFile = inputFile;
    }
    @Override
    public void run() {
        FileUtil.readFileLineByLine(inputFile, line -> {
            queue.add(Optional.of(line));
        });
        for(int i = 0; i < Consumer.consumerThreadCount; i++) {
            queue.add(Optional.empty());
        }
    }
}
```

producer类只是简单的重载了Runable的run方法，run方法一开始，将文件内容读入到queue队列中，这里面用到了Java8里面lanmda表达式。

`line -> {queue.add(Optional.of(line)}`

FileUtil是一个文件读取工具类，readFileLineByLine将文件按行读出到queue中,这里面同样用到了Java8中的Consumer类，关于这个类暂时按下不表，可以理解为接收一个lanmda表达式，并对每个accept的参数进行lanmda表达式的操作：

``` java
public static void readFileLineByLine(String filePath, Consumer<String> consumer) {
       try {
           BufferedReader br = new BufferedReader(new InputStreamReader(new FileInputStream(filePath), "Cp1252"));
           String line;
           while ((line = br.readLine()) != null) {
               consumer.accept(line);
           }
           br.close();
       } catch(IOException e) {
           e.printStackTrace();
       }
   }
```

后面的for循环是将十个Optional空对象放入queue中，这里做的目的是实现对Consumer线程的结束控制，具体原理会在Consumer类中进行表述。Optional是Java8的特性，其官方解释是：

`这是一个可以为null的容器对象。如果值存在则isPresent()方法会返回true，调用get()方法会返回该对象`

较为通俗的说法是：

`如果你开发过Java程序，可能会有过这样的经历：调用某种数据结构的一个方法得到了返回值却不能直接将返回值作为参数去调用别的方法。而是首先判断这个返回值是否为null，只有在非空的前提下才能将其作为其他方法的参数。Java8中新加了Optional这个类正是为了解决这个问题。`

其具体用法以后解释，在本文中可简单理解为，在生产者将所有数据都写进队列后，我们放置10个空元素进入队列，消费者可根据空元素进行停止判断。

## 1.3 Consumer

Consumer的构建也是比较简单， 同producer一样，继承Runable，并重写Run方法：

``` java
public class Consumer implements Runnable {
    private AtomicInteger totalCount = new AtomicInteger(0);
    public static final int consumerThreadCount = 10;
    private FileUtil fileUtil;
    private final static Logger logger = Logger.getLogger(Consumer.class);
    private LinkedBlockingQueue<Optional<String>> queue;
    public Consumer(LinkedBlockingQueue<Optional<String>> queue, String outputFile) {
        this.queue = queue;
        this.fileUtil = new FileUtil(outputFile);
    }
    @Override
    public void run() {
        try {
            while (true) {
                Optional<String> line = queue.take();
                if (!line.isPresent()) break;
                totalCount.incrementAndGet();
                String processedLine = fileUtil.processLine(line.get());
                fileUtil.appendToFile(processedLine);
            }
        } catch (InterruptedException e) {
            e.printStackTrace();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
    public FileUtil getFileUtil() {
        return fileUtil;
    }
    public int getTotalCount() {
        return totalCount.get();
    }
}
```

主要看一下run方法，run方法使用while循环，循环读取queue中的数据。上文介绍过，queue.take()方法会一直阻塞直到队列中塞进数据。此外run方法中有使用fileUtil的processLine方法：

``` java
public String processLine(String line) throws IOException {
        int min = 1;
        int max = 100;
        int randomMillisecconds = min + (int)(Math.random() * ((max - min) + 1));
        try {
            Thread.sleep(randomMillisecconds);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        // readLineCount is not accurate in multi-thread mode
        readLineCount++;
        logger.info(String.format("%d lines were read.", readLineCount));
        return line;
    }
```

processLine主要是对Consumer读取的行数记性计数，并在log中打印出来。为什么这样做，未在测试实验时说明。
关于Consumer还有一点需要关注，我们看到Consumer的run方法体中是一个while循环，那Consumer线程什么时候会停止就变成了一个问题。按照我们设计初衷，Producer把所有的文件按行读取到queue中，Consumer回去queue中读取数据，写回到另一个文件。按照这种逻辑，如果producer读完了所有文件，Consumer也将queue中的所有数据写回文件，此时Consumer就应该停止了。

``` java
if (!line.isPresent()) break;
```

这段代码就是负责停止Consumer线程的。记得我们在Producer,当所有数据都读取到queue中时，会在queue中塞入十个optional.empty变量，那如果在Consumer中queue.take()返回的是optinal.empty，就说明queue已经无数据了，当前Consumer就可以停止了。关于如何在循环中停止线程，还有很多方法，待后面有时间再做解析。以上就是Consumer类的构造。

# 2.实验

实验代码：

``` java
@Test
   public void multiThreadTest() throws IOException {
       starter.startMultiThreadTask(inputFile, outputFile);
       Assert.assertTrue(isFileDataSame(inputFile, outputFile));
   }
```

实验结果,实验结果会打印总耗时以及两个count变量：

```
2017-03-01 13:52:04 INFO  FileUtil:118 - 2954 lines were read.
2017-03-01 13:52:04 INFO  FileUtil:118 - 2955 lines were read.
2017-03-01 13:52:04 INFO  FileUtil:118 - 2956 lines were read.
2017-03-01 13:52:04 INFO  FileUtil:118 - 2957 lines were read.
2017-03-01 13:52:04 INFO  FileUtil:118 - 2958 lines were read.
2017-03-01 13:52:04 DEBUG Starter:50 - Consumer's totalCount: 3000
2017-03-01 13:52:04 INFO  Starter:53 - It takes 16 seconds to finish
```

解释一下，诸如2958 lines were read，是从FileUtil工具类processLine函数中打印出的readLine变量，其代表意义是线程Consumer线程从queue中读取了多少行。Consumer’s totalCount: 3000 是直接打印的Consumer全局变totalCount，其代表意义同样是十个线程总共从queue中读取了多少行，按道理来说，这两个值是应该相同的，然后结果明显不一致。

从代码我们可以看到readLine是一个int型变量，而totalCount是一个AtomicInteger变量，很显然问题出在了这里。我们知道Java中++这种操作是线程不安全的，而readLineCount是个全局变量，所以如果多个线程同事在执行++操作时，就会产生totalCount的值不一致的问题，解决方法可以粗暴的在processLine中加上synchronized关键字：

``` java
public String processLine(String line) throws IOException {
       synchronized (this) {
           int min = 1;
           int max = 100;
           int randomMillisecconds = min + (int) (Math.random() * ((max - min) + 1));
           try {
               Thread.sleep(randomMillisecconds);
           } catch (InterruptedException e) {
               e.printStackTrace();
           }
           // readLineCount is not accurate in multi-thread mode
           readLineCount++;
           logger.info(String.format("%d lines were read.", readLineCount));
           return line;
       }
   }
```

我们再次运行测试脚本：

```
2017-03-01 14:09:34 INFO  FileUtil:119 - 2994 lines were read.
2017-03-01 14:09:34 INFO  FileUtil:119 - 2995 lines were read.
2017-03-01 14:09:34 INFO  FileUtil:119 - 2996 lines were read.
2017-03-01 14:09:34 INFO  FileUtil:119 - 2997 lines were read.
2017-03-01 14:09:35 INFO  FileUtil:119 - 2998 lines were read.
2017-03-01 14:09:35 INFO  FileUtil:119 - 2999 lines were read.
2017-03-01 14:09:35 INFO  FileUtil:119 - 3000 lines were read.
2017-03-01 14:09:35 DEBUG Starter:50 - Consumer's totalCount: 3000
2017-03-01 14:09:35 INFO  Starter:53 - It takes 160 seconds to finish
```

可以看到两个变量的值相等了，说明这种方法可行，但是花费的时间确从16s到了160s，说明synchronized关键字极大的增加了时间的消耗，我们分析processLine方法，其实问题只出在readLineCount上，concurrent包中提供了AtomicInteger变量，它实现了对int变量的封装，实现了对自增操作的原子性。为此我们将readLineCount定义为：

``` java
private AtomicInteger readLineCount = new AtomicInteger(0);
```

processLine函数变为：

``` java
public String processLine(String line) throws IOException {
       int min = 1;
       int max = 100;
       int randomMillisecconds = min + (int) (Math.random() * ((max - min) + 1));
       try {
           Thread.sleep(randomMillisecconds);
       } catch (InterruptedException e) {
           e.printStackTrace();
       }
       // readLineCount is not accurate in multi-thread mode
       readLineCount.incrementAndGet();
       logger.info(String.format("%d lines were read.", readLineCount.get()));
       return line;
   }
```

再次运行测试代码：

```
2017-03-01 14:18:23 INFO  FileUtil:120 - 2994 lines were read.
2017-03-01 14:18:23 INFO  FileUtil:120 - 2995 lines were read.
2017-03-01 14:18:23 INFO  FileUtil:120 - 2996 lines were read.
2017-03-01 14:18:23 INFO  FileUtil:120 - 2997 lines were read.
2017-03-01 14:18:23 INFO  FileUtil:120 - 2998 lines were read.
2017-03-01 14:18:23 INFO  FileUtil:120 - 2999 lines were read.
2017-03-01 14:18:23 INFO  FileUtil:120 - 3000 lines were read.
2017-03-01 14:18:23 DEBUG Starter:50 - Consumer's totalCount: 3000
2017-03-01 14:18:23 INFO  Starter:53 - It takes 15 seconds to finish
```

可以看到，两个变量值相等了，然后耗时只用了15s。
从以上实验可以看到使用concurrent包中的变量和方法，比简单粗暴的使用synchronized这种方法在耗时方便具有很大的优势。

# 3.总结

以上我们初步试验了一个简单的生产者消费者模型，使用了Cocurrent包中的一些方法。时间比较急，先写这么多，后续有内容再行添加。
