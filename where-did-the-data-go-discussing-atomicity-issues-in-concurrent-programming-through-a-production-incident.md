---
title: 数据去哪了？：从一次生产事故聊聊并发编程原子性问题
toc: true
date: 2021-09-30 18:04:28
tags:
  - Java
---

## 1. 引言

最近公司小伙伴的服务遇到一个奇怪的丢数据问题：每天总是莫名其妙的丢几条数据，经过分析排查之后发现是没有处理好并发而导致的。

问题复盘之后我认为这是并发编程中典型的**原子性问题**。对于并发编程不是很熟悉的小伙伴来说是一个很好的例子。

## 2. 问题复盘

整个业务的逻辑其实是比较简单：不断的接收消息，定时的把收集的消息发送到一个目标地址。

### 2.1 关键代码

> talk is cheap, show me the code!

我仿写了引起并发问题的类，只保留了核心逻辑，除了`lombok`与`logback`之外没有引入其他第三方包。

```java
@Slf4j
public class ConcurrentPostResult {

    private List<String> cache = new ArrayList<>(1000);

    /**
     * 模拟接收数据
     * 
     * @param data
     */
    public void receive(String data) {
        cache.add(data);
        log.info("增加数据 {}, 当前数据容量 {}", data, cache.size());
    }

    /**
     * 模拟发送数据
     * 
     * @throws InterruptedException
     */
    public void postResult() throws InterruptedException {
        log.info("当前缓存数据数量 {}", cache.size());
        // 等待10ms，模拟发送数据耗时，这里实际会拷贝一份数据进行发送
        TimeUnit.MILLISECONDS.sleep(10);
        log.info("发送数据后的缓存数据数量 {}", cache.size());
        cache.clear();
        log.info("清除缓存数据 {}", cache.size());
    }

    public static void main(String[] args) throws InterruptedException {
        ConcurrentPostResult postResult = new ConcurrentPostResult();

        int threadCount = 8;
        ForkJoinPool forkJoinPool = new ForkJoinPool(threadCount);

        // 模拟并发接收数据
        forkJoinPool.execute(() -> IntStream.range(0, 1000)
                .mapToObj(String::valueOf)
                .parallel().forEach(postResult::receive));

        postResult.postResult();

    }
}
```

### 2.2 输出结果

```
00:29:46.671 [main] INFO org.example.ConcurrentPostResult - 当前缓存数据数量 0
00:29:46.678 [ForkJoinPool-1-worker-0] INFO org.example.ConcurrentPostResult - 增加数据 85, 当前数据容量 169
...省略日志
00:29:46.686 [ForkJoinPool-1-worker-5] INFO org.example.ConcurrentPostResult - 增加数据 547, 当前数据容量 616
00:29:46.686 [main] INFO org.example.ConcurrentPostResult - 发送数据后的缓存数据数量 616
00:29:46.686 [ForkJoinPool-1-worker-6] INFO org.example.ConcurrentPostResult - 增加数据 797, 当前数据容量 617
...省略日志
00:29:46.686 [ForkJoinPool-1-worker-0] INFO org.example.ConcurrentPostResult - 增加数据 57, 当前数据容量 12
00:29:46.686 [main] INFO org.example.ConcurrentPostResult - 清除缓存数据 2
```

## 3. 问题分析

从上一节的日志输出可以看到，在执行`postResult()`方法发送数据，实际上会经过一个比较长的网络I/O操作[^注1]。并且执行该操作时，上游系统还在不断推送数据加入到缓存中，如下图所示：

![一次并发问题](https://noteedit.oss-cn-beijing.aliyuncs.com/picGo/0259eb7083fd47dd87d36b589a1cfc6b.png)




我们进入到`postResult()`方法时只发送缓存中的10条数据，但实际上在这个过程中可能不断有新数据加入到缓存中，这部分数据并没有发送给下游服务。

最后在发送完成之后执行了`cache.clear()`操作导致数据丢失。

## 4. 解决

我们没有意识到这是个并发的操作，因此下意识的认为接收与发送数据都是原子性的：执行接收数据的时候不会发送数据，执行发送数据的时候不会接收数据。

比较直接方法是对缓存的对象加锁，这里有两个注意点：

1. 所有涉及到共享对象(这里是cache)的操作都需要加速
2. 保证加的是同一把锁

```java
@Slf4j
public class ConcurrentPostResult {

    private List<String> cache = new ArrayList<>(1000);

    /**
     * 模拟接收数据
     *
     * @param data
     */
    public void receive(String data) {
        // 对缓存加锁
        synchronized (cache) {
            cache.add(data);
        }
        log.info("增加数据 {}, 当前数据容量 {}", data, cache.size());
    }

    /**
     * 模拟发送数据
     *
     * @throws InterruptedException
     */
    public void postResult() throws InterruptedException, IOException, ClassNotFoundException {
        List data2Send;
        // 执行发送操作前加锁
        synchronized (cache) {
            // 深拷贝数据，避免cache对象被阻塞太久，对性能造成影响
            data2Send = deepCopy(cache);
            log.info("当前缓存数据数量 {}, 待发送的数据数量 {}", cache.size(), data2Send.size());
            cache.clear();
        }
        // 等待20ms，模拟发送数据耗时，这个时间基本上能保证把1000条数据消耗完
        TimeUnit.MILLISECONDS.sleep(20);
        log.info("发送数据后的缓存数据数量 {}, 发送的数据数量 {}", cache.size(), data2Send.size());
    }

    public static void main(String[] args) throws InterruptedException, IOException, ClassNotFoundException {
        ConcurrentPostResult postResult = new ConcurrentPostResult();

        int threadCount = 8;
        ForkJoinPool forkJoinPool = new ForkJoinPool(threadCount);

        // 模拟并发接收数据
        forkJoinPool.execute(() -> IntStream.range(0, 1000)
                .mapToObj(String::valueOf)
                .parallel().forEach(postResult::receive));

        log.info("执行前");
        TimeUnit.MILLISECONDS.sleep(5);
        log.info("执行后");

        postResult.postResult();
    }

    public static <T> List<T> deepCopy(List<T> src) throws IOException, ClassNotFoundException {
        ByteArrayOutputStream byteOutput = new ByteArrayOutputStream();
        ObjectOutputStream out = new ObjectOutputStream(byteOutput);
        out.writeObject(src);

        ByteArrayInputStream byteInput = new ByteArrayInputStream(byteOutput.toByteArray());
        ObjectInputStream input = new ObjectInputStream(byteInput);
        List<T> dest = (List<T>) input.readObject();
        return dest;
    }
}

```

这里我特意增加了等待时间，可以看到已发送的数据与未发送数据之和为1000，确保数据未丢失。

```
21:13:17.268 [main] INFO org.example.ConcurrentPostResult - 执行前
21:13:17.274 [ForkJoinPool-1-worker-4] INFO org.example.ConcurrentPostResult - 增加数据 159, 当前数据容量 26
...省略日志
21:13:17.276 [ForkJoinPool-1-worker-3] INFO org.example.ConcurrentPostResult - 增加数据 932, 当前数据容量 157
21:13:17.277 [main] INFO org.example.ConcurrentPostResult - 执行后
21:13:17.277 [ForkJoinPool-1-worker-3] INFO org.example.ConcurrentPostResult - 增加数据 933, 当前数据容量 178
21:13:17.277 [ForkJoinPool-1-worker-7] INFO org.example.ConcurrentPostResult - 增加数据 204, 当前数据容量 176
21:13:17.288 [main] INFO org.example.ConcurrentPostResult - 当前缓存数据数量 178, 待发送的数据数量 178
21:13:17.288 [ForkJoinPool-1-worker-7] INFO org.example.ConcurrentPostResult - 增加数据 205, 当前数据容量 1
...省略日志
21:13:17.301 [ForkJoinPool-1-worker-4] INFO org.example.ConcurrentPostResult - 增加数据 874, 当前数据容量 822
21:13:17.311 [main] INFO org.example.ConcurrentPostResult - 发送数据后的缓存数据数量 822, 发送的数据数量 178
```

## 结论

本文从一个真实案例说起，分析了代码中隐藏的并发问题以及解决方案。

在这个例子中，以下几点内容是我们需要关注的：

1. 分析你的服务是否存在并发场景
2. 是否有对共享对象的操作（上文的例子中是`cache`对象）
3. 加锁[^注2]可以解决并发问题中的原子性、一致性、顺序性问题

在这里我们只讨论了最直观的解决方案，有机会在后续的文章中将深入讨论Java内存模型来对并发问题追根溯源。

[^注1]: 这里是让线程暂停了10ms模拟网络I/O耗时
[^注2]:例子中用了java的synchronized关键字
