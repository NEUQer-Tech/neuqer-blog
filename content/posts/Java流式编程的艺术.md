---
title: "Java流式编程的艺术"
date: 2022-01-09T19:03:17+08:00
author: 朱坤帅
authorLink: https://github.com/JooKS-me
tags: ["技术分享","Java Stream"]
categories: ["java基础"]
draft: false
---

### 特性

1. 流不存储元素，但是可以根据需要进行计算转化
2. 流的数据可以来自数据结构、数组、文件等
3. 支持多种聚合操作，如fliter/map/reduce/find/match/sorted等
4. 很多流操作的返回也是一个流
5. 用流操作进行迭代，用户感知不到循环遍历

### 语法

类似sql语句，select对应着map，where对应filter

### 流的工作流程

1. 创建流
2. 转换流，可包含多个步骤，但是在真正的计算之前不会运行，是惰性操作
3. 计算。强制执行之前的惰性操作。计算后，流就不能用了。

### 流的创建

- Collection接口的stream方法

  ```java
  Stream<String> as = new ArrayList<String>().stream();
  ```

- Arrays.stream，将数组转化为stream

  ```java
  Stream<String> as = Arrays.stream("a,b,c,d".split(","), 3, 5);
  ```

- **Stream类**

  - of方法，可以直接将数组转化为stream

    ```java
    Stream<Integer> as = Stream.of(new Integer[5]);
    ```

  - empty方法，产生一个空流

  - generate方法，接受一个lambda表达式

  - iterate方法，接收一个种子，和一个lambda表达式

    ```java
    Stream<BigInteger> e3 = Stream.iterate(BigInteger.ZERO, n->n.add(BigInteger.ONE));
    ```

- 基本类型流（只有三种）：IntStream、LongStream、DoubleStream

  这几个类跟Stream是兄弟关系，没有继承关系。

  ```java
  IntStream s1 = Intream.of(1,2,3,4,5); //of方法
  s1 = IntStream.generate(()->(int)(Math.random()*100)); //generate方法
  s1 = IntStream.range(1,5); //1,2,3,4
  s1 = IntStream.rangeClosed(1,5); //1,2,3,4,5
  Stream<Integer> s2 = s1.boxed(); //装箱
  IntStream s3 = s2.mapToInt(Integer::intValue); //拆箱
  ```

- 并行流

  所有中间转换操作被并行化

  Collections.parallelStream() 可以将任何集合转为并行流；Stream.parallel()方法（基本类型流也可以）也可以产生一个并行流

  ```java
  IntStream s1 = IntStream.range(1,10000000);
  long evenNum = s1.parallel().filter(n->n%2==0).count(); //并行捞出所有偶数
  ```

  > 注意：需要保证传给并行流的操作不存在竞争

- 其他

  - Files.lines方法。可以取代readline

    ```java
    Stream<String> contents = Files.lines(Path.get("C:/abc.txt"));
    ```

  - Pattern的splitAsStream方法

    ```java
    Stream<String> words = Pattern.compile(",").splitAsStream("a,b,c");
    ```

### 流的转换

- 过滤 filter

  filter(Preficate<? super T> predicate)

- 去重 distinct

  distinct()

  先调用hashCode方法，再调用equals，两个都一样就表示重复，跟HashMap类似

- 排序 sorted

  sorted() / sorted(Comparator)

  - sorted() 对流的基本类型元素进行排序

  - sorted() 也可以根据对象的compareTo方法进行排序

  - sorted(Comparator) 提供comparator，对流的元素进行排序

    ```java
    // 按字符串长度排序
    String[] planets = new String[] {"Messs", "ssssssww", "ddda", "scsacasf"};
    Stream<String> s3 = Stream.of(planets).sorted(Comparator.comparing(String::length));
    ```

- 映射 map

  map(xxx)

  括号中可以是lambda表达式，也可以是方法引用

- 映射并合并 flatmap

- 抽取 limit

  limit(long)

  限定得到的元素个数，从前面开始记

- 跳过 skip

  skip(long)

  跳过指定元素个数

- 连接 concat

  concat(stream, stream)

  连接两个流

- 额外调试 peek

  peek(Consumer)

  保持原来的流不变，但是会额外执行peek中的函数

### Optional类型

- 创建
  - Optional.of(T) 可以生成 `Optional<T>` 对象
  - Optional.ofNullable(T) 对于对象为null的情况下，安全创建。如果参数为null，则用empty()方法创建一个空盒子
  - Opionnal.empty() 生成一个空的盒子

- 使用
  - get()方法，获取值，但是不安全，如果里面是null会发生 NoSuchElementException 异常
  - orElse(T) 获取值，可以在Optional包装着null时返回一个默认值
  - orElseGet() 获取值，如果为null，采用lambda表达式值返回
  - orElseThrow() 获取值，如果为null，抛出异常
  - ifPresent() 判断是否空，不为空返回true
  - isPresent(Consumer)，判断是否为空，如果为空不做任何操作；如果不为空则进行Consumer的操作
  - map(Function) 将值传递给Function函数进行计算；如果为空，则不计算

### 流的计算

- 约简（聚合）

  - 简单约简（聚合函数） n -> 1

    count()，计数

    max(Comparator)，最大值

    min(Comparator)，最小值

    findFirst()，找到第一个元素

    findAny()，找到任意一个元素（随机返回一个元素）

    anyMatch(Predicate)，如果有任意一个元素满足Predicate，返回true

    allMatch(Predicate)，所有元素满足Predicate时，返回true

    noneMatch(Predicate)，没有任何元素满足Predicate时，返回true

  - 自定义约简 reduce

    传递一个二元函数BinaryOperator即可，也可以设置初始值

    ![image-20211230201914926](https://img.jooks.cn/img/202112302019987.png)

- 查看/遍历元素

  - iterator，迭代器

    流可以调用iterator()方法，返回一个迭代器

  - forEach(Consumer)，把Consumer函数应用到每个元素上

- 收集

  - toArray()，转为数组
  - collect(Collectors.toList())，转为List
  - collect(Collectors.toMap())，转为Map
  - ...
  - collect(Collectors.joining())，将结果连接起来

- 分组/分区（属于收集的一部分）

  - groupingBy 和 partitioningBy 的区别是，分组函数的返回值没有限定，分区函数的返回值是布尔类型的。
  - 他们都有三个重载方法：(func)、(func, Collectors#func)、(func, MapNewFunc, Collectors#func)
  - Collectors#func可以对分组后的value进行计算，有以下方法：
    - counting
    - summingInt / summingLong/ summingDouble(func)，接受一个参数应用到下游元素中。
    - maxBy / minBy 接受一个比较器
    - mapping，做一次映射，并且绑定一个下游收集器，这就意味着mapping可以无限嵌套。
    - reducing，用于分组后的约简操作
    - toList、toSet、toMap
    - 据说java 9和java 12有增加。

### 原理探究

JDK将stream所有的操作分类，如下：

![image-20220108221433557](https://img.jooks.cn/img/202201082214658.png)

其中，

- 无状态，表示对元素的处理不受前面元素的影响

- 有状态，表示必须等所有元素都处理完成后才知道最终状态
- 短路操作，表示不用遍历全部的元素，就可以得到结果。
- 非短路与短路相反。

如果要设计这么一个流式编程框架，我们大概需要解决以下几个问题：

1. 用户的操作如何记录？
2. 操作如何连接？
3. 连接之后的操作如何执行？

![image-20220109144657749](https://img.jooks.cn/img/202201091446807.png)

##### 用户的操作如何记录？

流操作都被封装到Sink中。

```java
interface Sink<T> extends Consumer<T> {
  default void begin(long size) {}
  default void end() {}
  default void accept(int value) {
        throw new IllegalStateException("called wrong accept method");
  }
  ...
}
```

##### 操作如何连接？

sink之间通过onWrapSink方法进行连接

```java
@Override
Sink<P_OUT> opWrapSink(int flags, Sink<P_OUT> sink) {
    return new Sink.ChainedReference<P_OUT, P_OUT>(sink) {
        @Override
        public void begin(long size) {
            downstream.begin(-1);
        }

        @Override
        public void accept(P_OUT u) {
            if (predicate.test(u))
                downstream.accept(u);
        }
    };
}
```

但是，调用opWrapSink的句柄是pipeline，所以我们还需要记录下pipeline。

```java
previousStage.nextStage = this;
this.previousStage = previousStage;
```

##### 连接之后的操作如何执行？

```java
final <R> R evaluate(TerminalOp<E_OUT, R> terminalOp) {
  ...
}
```

```java
@Override
final <P_IN, S extends Sink<E_OUT>> S wrapAndCopyInto(S sink, Spliterator<P_IN> spliterator) {
    copyInto(wrapSink(Objects.requireNonNull(sink)), spliterator);
    return sink;
}
```

### 性能测试

https://www.cnblogs.com/CarpenterLee/p/6675568.html











