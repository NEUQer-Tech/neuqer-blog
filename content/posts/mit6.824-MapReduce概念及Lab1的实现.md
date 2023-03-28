---
title: "Mit6.824 MapReduce概念及Lab1的实现"
date: 2023-03-27T23:50:34+08:00
draft: false
author: 吴亚洲
tags: ["技术分享","mit6.824"]
categories: ["分布式系统"]
---

本文是我学习`MIT 6.824 Lab1`的笔记，主要内容是对于`MapReduce`的理解和`Lab1`的实现。

@[TOC]

## `MapReduce`框架

如果还没有接触过`MapReduce`，最好先阅读一下[MapReduce论文](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)，如果阅读英文论文对你来说有些困难，也可以阅读[MapReduce论文译文](https://www.cnblogs.com/xiaoxiongcanguan/p/16724085.html)。

### `MapReduce`是什么

`MapReduce`是一个软件框架，基于该框架能够容易地编写应用程序，这些应用程序能够运行在由上千个商用机器组成的大集群上，并以一种可靠的，具有容错能力的方式并行地处理上`TB`级别的海量数据集。

它极大地方便了编程人员在不会分布式并行编程的情况下，将自己的程序运行在[分布式系统](https://baike.baidu.com/item/分布式系统/4905336?fromModule=lemma_inlink)上。

### `MapReduce`能做什么

`MapReduce`的思想是“分而治之”，因此`MapReduce`尤其擅长处理大数据。

比如Google可以利用`MapReduce`框架处理大量的爬虫获取到的文档、网络请求日志等原始数据，获得倒排索引等衍生数据。

`MapReduce`由两个主要的过程构成，即“`Map`（映射）”和“`Reduce`（归约）”。

1. `Mapper`负责“分”，即把大量、复杂的任务分解为若干个“简单的任务”，其中“简单的任务”需要满足以下几点要求：
   + 数据规模相较于原任务要极大缩小
   + 满足“就近计算原则”，即计算过程最好发生在存放需要计算数据的节点上
   + 每个小任务之间几乎没有依赖关系，这些小任务可以并行计算

2. `Reducer`负责“汇总”，即把`Mapper`处理完的数据收集起来。

举一个简单的例子：

假如我们有`1TB`的英文文本数据，怎么才能统计出文本中每个不同的单词出现的次数呢？

当数据量很小的时候，我们很容易想到，只需要声明一个哈希表`map`，把每个单词出现的次数记录到`map`中就可以解决问题。但是现在文本的大小是以`TB`为单位，如果此时我们继续采取上述策略，在一台机器上运行，那么无论是在时间上还是在空间上都是不可行的。

此时，`MapReduce`就派上用场了。

![在这里插入图片描述](/images/WordCount.png)


如上图，

1. 在进行`MapReduce`之前我们需要先把数据分割成若干份小数据，以便让它们在多台机器上并行运算，分割后的结果是`<Key,Value>`类型，其中`Key`可能有多种含义，在后续运算中一般不会用到，每一个`Value`是若干单词的集合。
2. 分割完成后，在每一台机器上都会先进行`Map`过程，`Map`方法会统计出输入数据中每个单词出现的次数，并以`<Key,Value>`形式展现，其中`Key`是单词，`Value`是该单词的出现次数，在本例中`Value`值都为`1`。
3. 完成`Map`过程后，有可能会有一个`Combine`聚合过程，该过程会把`Key`相同的几组数据聚合成一组。
4. 最后是`Reduce`过程，该过程会汇总所有机器`Combine`后的结果，并把`Key`相同的几组数据聚合成一组，最终得到原始数据中每个单词出现的次数。

### MapReduce的工作机制

![在这里插入图片描述](/images/MapReduce.png)


上图是论文中对于`MapReduce`流程的描述。

它主要包含以下四个角色：

1. `User Program`：用户程序，即客户端，用来提交`MapReduce`任务。
2. `Master`：`master`进程用于分配任务，协调任务的进行。
3. `Map Worker`：执行`Map`方法的进程。
4. `Reduce Worker`：执行`Reduce`方法的进程。

输入数据以文件形式进入系统。在`master`进程的调整、分配下一些进程运行`map`任务，拆分了原任务，产生了一些中间体，这些中间体可能以键值对形式存在。另外一些进程运行了`reduce`任务，利用中间体产生最终输出。

## Lab1: MapReduce

### 前言

+ `Lab1`的官方说明在[Lab1 Note](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)。
+ 在动手实现 `Lab1`之前，一定要先阅读[MapReduce论文](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)。
+ `mit 6.824`的所有`Lab`都是用`GoLang`来实现的，如果之前没有学习过`Go`语言，可以从以下途径中任选一条进行学习：
  + [Go 官方指南](https://tour.go-zh.org/)
  
  + [Go语言编程快速入门](https://www.bilibili.com/video/BV1fD4y1m7TD/)
  
  + [Go构建Web程序](https://github.com/astaxie/build-web-application-with-golang/blob/master/zh/preface.md)
+ 这是初始实验的git仓库:`git://g.csail.mit.edu/6.824-golabs-2020 6.824`,所有代码需要运行在`Linux`或`Mac OS`上。

### 总览

`Lab1`要求我们实现一个和[MapReduce论文](https://pdos.csail.mit.edu/6.824/papers/mapreduce.pdf)类似的机制，也就是数单词个数`Word Count`。

在正式开始写分布式代码之前，我们先理解一下任务和已有的代码。

输入文件在`src/main`中，文件名是`pg-*.txt`，其中每一个文件都是一本电子书，内容很多，我们的任务是统计出所有`pg-*.txt`文件中所有出现的单词以及它们的出现次数。

### 非分布式实现

在单机中，实现这个程序很简单，初始实验条件中已经给出了示例，在`src/main/mrsequential.go`中可以看到代码实现。

尝试在合适的系统中运行该示例：

```sh
cd src/main
go build -buildmode=plugin ../mrapps/wc.go
go run mrsequential.go wc.so pg*.txt
```

如果运行成功，输出结果将存在于`src/main/mr-out-0`文件中，里面展示了所有文章出现的所有单词以及它们的出现次数。

其中，`go build -buildmode=plugin ../mrapps/wc.go`将 `wc.go` 文件编译为一个插件模块，可以在运行时被其他代码加载和使用。`wc.go`中定义了名为`Map`和`Reduce`的函数，这两个函数在`mrsequential.go`被加载和调用。

`go run mrsequential.go`后面的两项是传入的命令行参数，可以在`mrsequential.go`中用`os.Args`获取到，第一个参数表示要加载的插件，第二个参数表示要统计单词次数的输入文件名。

`mrsequential.go`的代码比较容易理解，在此不再过多解释，在我的[GitHub仓库](https://github.com/asiaWu3/mit6.824-Lab1)中该代码有详细的中文注释，可以去查看以方便理解。

`mrsequential.go`的实现是非分布式的，但分布式下的代码和非分布式下的代码运行得到的结果应该是完全一样的，因此`mr-out-0`的内容将作为分布式下代码运行结果的评估标准的一部分。

### 分布式实现

我们要写的代码在`src/mr`中，`src/mr`中的代码将由`src/main/mrmaster.go`和`src/main/mrworker.go`调用，这两个代码的作用是启动进程、加载`wc.so`插件。

其中，前者需要运行一次以启动一个`master`进程，后者需要运行多次以启动多个`worker`进程。

`master`进程用于监听`worker`进程的`RPC`调用，并给它们分配合适的`map/reduce`任务，在此过程中`master`需要关注`worker`是否完成了任务，如果没有按时完成，需要将任务重新分配给其他`worker`。

`worker`进程启动后会主动请求`master`进程要任务，具体要到的是`map`还是`reduce`任务由`master`决定。

`map`方法传入文件名和文件内容，对文件内容进行处理，分割每一个单词，形成许多`key-value`键值对并作为结果返回，`key`是一个个单词，`value`的值都为 $1$  ，因为一个输入文件中可能有很多相同的单词，所以会出现很多相同的`key`。

`reduce`方法接收到的是同一个键（`key`）所对应的所有值（`value`）集合，而这些值（`value`）来自于不同的 `map` 任务，由于`map`返回的所有`value`都为 $1$ ，所以`reduce`只需要返回传入集合中数据的个数，这个值就是单词`key`在所有输入文件中出现的次数，合并所有`reduce`任务的返回结果就可以得到`Word Count`的结果。

运行过程:

```sh
go build -buildmode=plugin ../mrapps/wc.go
rm mr-out*
go run mrmaster.go pg-*.txt
# 运行一个或多个mrworker
go run mrworker.go wc.so
#将所有mr-out-*文件合并、排序后查看结果
cat mr-out-* | sort | more
```

`Lab1`的原始代码中已经提供了`RPC`的实现和调用示例，我们只需要按照示例写出发送的信息`args`、要接收的信息`reply`和远程调用的方法名（反射实现）。

我们最终的目标是通过`src/main/test-mr.sh`中的五个测试：

1. 基本的`Word Count`测试:先生成正确的输出，然后使用 `mrmaster` 和 `mrworker` 运行测试。如果输出结果与正确的结果相同，则测试通过。
2.  `indexer` 测试：先生成正确的输出，然后使用 `mrmaster` 和 `mrworker` 运行测试。如果输出结果与正确的结果相同，则测试通过。
3. `map` 并行度测试：在启动 `mrmaster` 和 `mrworker` 后，同时启动两个 `mtiming.so`的 `worker`，通过比较输出结果判断是否达到预期的并行度。
4. `reduce` 并行度测试：在启动 `mrmaster` 和 `mrworker` 后，同时启动两个 `rtiming.so` 的 `worker`，通过比较输出结果判断是否达到预期的并行度。
5. 容错测试：在启动 `mrmaster` 和 `crash.so worker` 后，它会不断重启 `crash.so worker` 直到完成任务。完成后，比较输出结果与正确结果是否相同。

仔细查看`test-mr.sh`中第一个测试部分的代码，其中有一行

```sh
sort mr-out* | grep . > mr-wc-all
```

意思是把所有`mr-out-*`文件合并并排序后存放在`mr-wc-all`中，之后再拿`mr-wc-all`和`Word Count`正确的结果对比。

因此，我们的任务不是像`mrsequential.go`一样生成最终的一个`mr-out-0`文件，而是生成多个`mr-out-*`文件，具体个数由`mrmaster`中传入的`nReduce`的值决定，即一个`reduce`任务最终生成一个`mr-out-*`文件，这就要求我们每个`reduce`任务处理的单词不能一样，即所有相同的单词都要被放到同一个`reduce`任务里执行。

[Lab1 Note](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)中已经给出了提示，每个`map`任务处理一个`pg-*txt`文件，一个`map`任务产生的中间文件应该命名为`mr-X-Y`，可能有多个，如`mr-X-0`、`mr-X-1`......

其中`X`为`mapWorkerId`，`Y`是由处理到的单词通过`ihash`方法联合`nReduce`决定的，表示把这个单词分给哪个`reduce`任务。因此，不同文件中相同的单词被分到的`mr-*-Y`文件`Y`的值一定相同，之后执行第`i`个`reduce`任务时只需要读取所有的`mr-*-i`文件并合并处理就可以实现所有相同的单词被同一个`reduce`任务处理的效果，最终经过`reduce`任务的处理，产生文件`mr-out-i`。

### GoLand配置

如果在`Ubuntu`或`Mac OS`上用的`IDE`是`GoLand`，有以下几个配置可以简化运行时的操作：

1. 因为每次更改`worker.go`后，在运行测试前都要重新加载插件，所以可以把加载插件的语句放在一个`sh`脚本里，比如我代码里的`src/main/wc-build.sh`，里面其实只有一句：

   ```sh
   CGO_ENABLED=1 go build -race  -buildmode=plugin ../mrapps/wc.go
   ```

2. 另外因为运行`mrmaster.go`和`mrworker.go`需要传入命令行参数，所以可以在`GoLang`右上角"编辑配置"中进行如下配置：

   + `mrsequential.go`:

     ![在这里插入图片描述](/images/mapreduce-1.png)


   + `mrmaster.go`：

     ![在这里插入图片描述](/images/mapreduce-2.png)


   + `mrworker.go`：

     ![在这里插入图片描述](/images/mapreduce-3.png)


注意，因为`GoLand`的实参不支持通配符，所以在写文件名时不能写`pg-*txt`，要把所有文件的文件名都写上，具体如下：

```tex
pg-being_ernest.txt pg-dorian_gray.txt pg-frankenstein.txt pg-grimm.txt pg-huckleberry_finn.txt pg-metamorphosis.txt pg-sherlock_holmes.txt pg-tom_sawyer.txt
```

	上面两项都配置好之后，就可以按如下顺序启动并测试程序：

1. 运行`wc-build.sh`
2. 运行`mrmaster.go`
3. 运行`mr-worker.go`

### 程序运行流程

因为所有代码加起来代码量很大，所以在这里不会展示具体的代码，只详细解释程序运行的流程，代码可以去[GitHub仓库](https://github.com/asiaWu3/mit6.824-Lab1)查看。

![在这里插入图片描述](/images/mapreduce-process.png)


上图简要展示了程序运行的流程，具体流程如下：

1. 客户端通过运行一次`mrmaster.go `启动一个`master`进程，启动时，会进行以下操作：
   + 创建一个`Master`结构体
   + 根据命令行传入的文件生成所有`map`任务并存放在`Master`的`MapChannel`中
   + 开启一个新的线程不断循环判断有没有过期(分配出去但十秒内未完成)的`map`和`reduce`任务
   + 开始监听来自`worker`的`RPC`调用

2. 客户端通过运行多次`mrworker.go`启动多个`worker`进程，每个`worker`启动时，都会进行以下操作：
   + 创建一个`AWorker`结构体
   + 只要`master`进程没有通知`worker`进程所有任务都已经完成，`worker`进程就一直向`master`进程要任务

3. `master`接收到`worker`的要任务请求后，根据所有任务的完成情况给`worker`分配任务，分配规则是：所有`map`任务全部完成后才可以去分配`reduce`任务

4. 当`worker`要到任务后，`worker`会首先判断这个任务是`map`任务还是`reduce`任务，并根据任务类型执行不同的逻辑
   + 如果是`map`任务，会执行`doMap`逻辑，全部执行完后会在`/var/tmp`目录下生成文件名为`mr-X-Y`的临时文件，其中`X`是当前`mapWorker`的`ID`，`Y`是通过`ihash`方法计算出来的
   + 如果是`reduce`任务，会执行`doReduce`逻辑，`worker`会从 `src/main`中读取所有`mr-*-Y`文件，其中`Y`是当前`reduceWorker`的`reduce`任务的编号，最终会在`/var/tmp`目录下生成文件名为`mr-out-Y`的临时文件
5. `worker`执行完自己的任务后，会执行`mapTaskDone`或`reduceTaskDone`方法，告诉`master`自己的任务完成了，通知完之后会立马继续向`master`要新的任务
6. `master`在接收到`worker`完成任务的通知后，会先判断这个`worker`完成这个任务有没有超时，具体就是判断这个任务的状态是不是`Running`，如果是就没有超时；否则，如果状态是`Ready`，就说明该任务已经被`master`中的循环检测任务过期的线程判定为过期并设为`Ready`状态；如果状态是`Finished`，说明这个任务被判定为超时后又被分配给其他`worker`并被那个`worker`按时完成
7. 如果`master`判断`worker`在规定时间内完成了任务，则：
   + `worker`执行的是`map`任务：`master`会调用`generateMapFile`方法，将`worker`在`/var/tmp`中生成的临时文件`mr-X-Y`复制到 `src/main`中，作为正式的该`map`任务完成后产生的中间文件，表示`master`接受了该`worker`的成果
   + `worker`执行的是`reduce`任务：`master`会调用`generateReduceFile`方法，将`worker`在`/var/tmp`中生成的临时文件`mr-out-Y`复制到 `src/main`中，作为正式的该`reduce`任务完成后产生的最终文件，表示`master`接受了该`worker`的成果

8. 当`master`的`MapChannel`中存储的所有任务都被完成后，`master`会将任务分配阶段由`MapPhase`调整为`ReducePhase`，在这之后，`master`就会给来请求任务的`worker`分配`reduce`任务
9. 当`master`的`ReduceChannel`中存储的所有任务都被完成后，表示所有的`map`任务和`reduce`任务都已经完成，此时`master`会调用`finish`方法，准备终止`master`进程，并通知所有的`master`任务完成了，终止所有`worker`

### 注意事项

+ 因为同一时间可能会有多个`worker`请求同一个`master`，所以一定要注意`master`里共享变量的读写，在适当的地方加锁。

+ 每次运行测试前一定要清空上一次测试生成的文件，防止对本次运行产生干扰。

+ 多输出调试信息，可以用`fmt.Print*`，也可以用`log.Print*`，在第一次输出正确的`Word Count`结果之前，输出越详细越好。

+ 用于`RPC`通信的结构体、变量名首字母要大写，否则会拿不到数据。
 因为同一时间可能会有多个`worker`请求同一个`master`，所以一定要注意`master`里共享变量的读写，在适当的地方加锁。

+ 每次运行测试前一定要清空上一次测试生成的文件，防止对本次运行产生干扰。

+ 多输出调试信息，可以用`fmt.Print*`，也可以用`log.Print*`，在第一次输出正确的`Word Count`结果之前，输出越详细越好。

+ 用于`RPC`通信的结构体、变量名首字母要大写，否则会拿不到数据。
