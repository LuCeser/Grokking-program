---
title: 使用jmh进行性能基准测试
date: 2021-09-30 18:04:28
tags:
  - Java
---
## 介绍 
我们在选择不同框架、算法时，不同场景下的性能是很重要考虑因素。JMH这个Java的微基准测试框架提供简单的方式来实现性能测试的需求。本文将以一个对比序列化器性能的例子简单介绍JMH的使用。
## 创建项目
不同于 `JUnit` 这种测试框架，JMH推荐创建独立的项目来做测试。
### 使用maven创建
  ``` bash 
  mvn archetype:generate \
  -DinteractiveMode=false \
  -DarchetypeGroupId=org.openjdk.jmh \
  -DarchetypeArtifactId=jmh-java-benchmark-archetype \
  -DgroupId=org.sample \
  -DartifactId=test \
  -Dversion=1.0
  ```
  执行命令后生成项目
### IDEA中创建项目

除了maven命令直接创建之外，也可以选择在IDE中创建maven，以IDEA为例。

在创建项目时，选择Maven项目，勾选 `Create from archetype` 并选择 `Add Archetype...`

在弹出的窗口中填入对应信息（当前最新版本为1.33）

![](https://noteedit.oss-cn-beijing.aliyuncs.com/uPic/Vu299C1632985257.png){:height 660, :width 764}

之后就可以选择JMH的archetype在IDEA中创建项目了。

## 编写测试代码 

项目自动生成的 `pom.xml` 文件中已经包含JMH运行最小依赖了，只需要加上待测试相关的依赖包。这里我要测试的是 `spring-data-redis` 中序列化对象相关的内容，因此需要添加以下依赖：

``` xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
    <version>2.1.1.RELEASE</version>
</dependency>
<!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-core -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-core</artifactId>
    <version>2.12.3</version>
</dependency>
<!-- https://mvnrepository.com/artifact/com.fasterxml.jackson.core/jackson-databind -->
<dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
    <version>2.12.3</version>
</dependency>
```
之后编写测试代码，这里我使用了对比了 `ObjectHashMapper` 和 `Jackson2HashMapper` 两个类的 `toHash` 方法平均调用时间。预热5轮，实际测试5轮并fork 5 个进程来进行测试。

``` java
@BenchmarkMode(Mode.AverageTime)  
@OutputTimeUnit(TimeUnit.NANOSECONDS)  
@Warmup(iterations = 5, time = 1)  
@Measurement(iterations = 5, time = 1)  
@Fork(5)  
@State(Scope.Benchmark)  
public class MyBenchmark {  

private HashMapper objectHashMapper;  
 private HashMapper jacksonHashMapper;  

 @Setup  
 public void setup() {  
    objectHashMapper = new ObjectHashMapper();  
 jacksonHashMapper = new Jackson2HashMapper(false);  
 }  

@Benchmark  
 public void testObjectHashMapper() {  
    SesAnswerRate answerRatePredictor = new SesAnswerRate(0.3F, 0.5F);  
 objectHashMapper.toHash(answerRatePredictor);  
 }  

@Benchmark  
 public void testJacksonHashMapper() {  
    SesAnswerRate answerRatePredictor = new SesAnswerRate(0.3F, 0.5F);  
 jacksonHashMapper.toHash(answerRatePredictor);  
 }  

public static void main(String[] args) throws RunnerException {  
    Options options = new OptionsBuilder()  
            .include(MyBenchmark.class.getSimpleName())  
            .build();  
 new Runner(options).run();  
 }  
}
```
  建议IDEA用户安装idea-jmh-plugin插件，便于运行测试。
## 执行测试

如果没有安装IDE插件，可以执行 `mvn clean package` 打包，之后在项目下的target文件夹中执行 `java -jar benchmarks.jar` 运行。

最终运行结果如下：

```
Benchmark                          Mode  Cnt     Score     Error  Units
MyBenchmark.testJacksonHashMapper  avgt   25   536.386 ±  25.589  ns/op
MyBenchmark.testObjectHashMapper   avgt   25  1601.561 ± 139.910  ns/op
```

可以看到使用 `Jackson2HashMapper` 序列化对象的速度要比 `ObjectHashMapper` 快上3倍。

## 总结

可以看到利用JMH能够快速编写，运行测试代码，对于method级别的性能测试非常有用，篇幅所限在此不展开讲述更加具体的用法。

建议有需要的同学们阅读官方示例： [jmh-samples](http://hg.openjdk.java.net/code-tools/jmh/file/2be2df7dbaf8/jmh-samples/src/main/java/org/openjdk/jmh/samples/)
