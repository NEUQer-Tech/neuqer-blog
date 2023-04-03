---
title: "Mit6.824 Raft概念及Lab2A的实现"
date: 2023-04-03T22:37:22+08:00
draft: false
author: 高昆
tags: ["技术分享","mit6.824"]
categories: ["分布式系统"]
---

@[TOC]

# 0. 分布式共识算法Raft

>  一个更易理解的**共识**算法(论文原文为**In Search of an Understandable Consensus Algorithm**)

## 0.1 什么是分布式共识问题?

- 分布式环境下系统集群存在很多节点，每个节点都可提出议案，我们需要：
  - **对多个节点提出的议案作裁决并得到一个一致的结论；**
  - **让每个节点都感知到最终结论，从而使集群整体状态保持一致**；
  - **允许一部分节点宕机后集群仍可正常工作，先前通过的议案仍可访问，集群状态仍维持一致**；

## 0.2 Raft算法的前世今生

- 在 Raft 算法提出之前，学术界早已有 [Paxos 算法](https://link.zhihu.com/?target=https%3A//zh.wikipedia.org/wiki/Paxos%E7%AE%97%E6%B3%95) ([莱斯利·兰伯特](https://zh.wikipedia.org/wiki/莱斯利·兰伯特)在1990年提出)来解决分布式共识问题。
- 到了 2006 年，Google 在两篇经典论文 [Bigtable:A Distributed Storage System for Structured Data](https://link.zhihu.com/?target=https%3A//static.googleusercontent.com/media/research.google.com/en//archive/bigtable-osdi06.pdf) 和 [The Chubby lock service for loosely-coupled distributed systems](https://link.zhihu.com/?target=https%3A//static.googleusercontent.com/media/research.google.com/en//archive/chubby-osdi06.pdf) 中提及用 Paxos 算法实现了一个分布式锁服务 Chubby，于是 Paxos 算法开始进入工业界领域被广大技术人员熟知。
- 在分布式共识算法领域，Paxos 算法可以说是宗师级角色，统治该领域十余年，大多数共识算法都是在其基础上进行改进和优化，**Raft 算法也不例外**。正因如此，Chubby 的作者 Mike Burrows 曾说过：**只有一种分布式共识算法，那就是 Paxos，其他共识算法都只是 Paxos 算法的不完整版**。
- 即便是大名鼎鼎的Paxos算法,也**存在一些问题**,Raft 算法的作者 `Diego Ongaro` 在研究 Paxos 算法时，就深受其复杂性困扰。他觉得 Paxos 算法是一门极其难懂的算法，其工程化实践更是困难重重，原始的 Paxos 算法不经过一番修改很难应用于工程之中，而修改后的 Paxos 算法又很难保证其正确性。他总结出 Paxos 算法有两个大问题：
  1. **非常难于理解**,Diego Ongaro花了一年时间才掌握Paxos算法
  2. **没有给工程实现提供一个好的基础**,而证明这点的最好论据就是:Paxos 算法自首次提出以来已过去十多年了，开源社区几乎没有一个被**广泛认可的工程实现**，很多 Paxos 算法的实现都是对其完整版的近似。
- 正因如此,Diego Ongaro打算开发一门新的共识算法,而这门新的共识算法就是著名的[Raft算法](https://zh.wikipedia.org/wiki/Raft)



























# 1. Raft概述

- Raft 是用来管理复制日志（replicated log）的一致性协议。它跟 Paxos 作用相同，效率也相当，但是它的组织结构跟 Paxos 不同.
- 目的是实现集群的高可用性,让集群中的每个节点都可用,即具备完整的正确的日志.
- 相比于Paxos，Raft最大的特性就是**易于理解(Understandable)**。为了达到这个目标,
  Raft主要做了两方面的事情：
  - 问题分解：把共识算法分为三个子问题，分别是**领导者选举(leader election)**、**日志**
    **复制(log replication)**、**安全性(safety)**.
  - 状态简化：对算法做出一些限制，减少状态数量和可能产生的变动。
  - 而Raft算法的强大也是被证明了的,[论文](https://baijiahao.baidu.com/s?id=1709264476801287853&wfr=spider&for=pc)中对43个大学生做了个实验，让他们同时学习Paxos和Raft，结果显示，其中有33个人学习Raft的成绩好于学习Paxos的成绩。

- Raft算法一般用于哪些情况呢?
  - 分布式数据库:在分布式数据库系统中，多个节点需要保持数据的一致性。Raft算法可以用于选举主节点（Leader），并确保主节点上的数据与从节点保持同步。
  - 分布式文件系统：在分布式文件系统中，多个节点需要协同工作以提供文件访问和存储服务。Raft算法可以用于确保节点之间的数据一致性和高可用性。
  - 分布式任务调度系统：在分布式任务调度系统中，需要选择一个节点作为任务的调度器，并确保所有节点上的任务调度是一致的。Raft算法可以用于选举任务调度器并确保任务调度的一致性。
  - 其他需要一致性保证的任何场所.










































# 2. 复制状态机

![在这里插入图片描述](/images/image-20221125161310118.png)

- 在具体介绍Raft之前，我们要先了解一下复制状态机（Replicatedstatemachine）的概念。
- <font color = red>相同的初始状态＋相同的输入=相同的结束状态</font>
- 多个节点上，从**相同的初始状态**开始，执行相同的一串命令，产生**相同的最终状态**。
- 在Raft中，leader将客户端请求（command)封装到一个个**日志实体(log entry)**中，再把这些log entries复制到所有follower节点，然后所有节点一起把log应用到自己的状态机(state Machine)上，根据复制状态机的理论，大家的结束状态肯定是一致的。
- 这样,client无论查询那个节点的状态机,查询到的结果都是一致的
- 可以说，我们使用共识算法，就是为了实现复制状态机。一个分布式场景下的各节点间，就是通过共识算法来保证命令序列的一致，从而始终保持它们的**状态一致,**从而实现高可用的。（投票选主是一种特殊的命令）
- 这里稍微拓展一点，复制状态机的功能可以更加强大。比如**数据库两个副本一个采用行存储的数据结构存储数据，另一个采用列存储，只要它们初始数据相同，并持续发给他们相同的命令，那么同一时刻从两个副本中读取到的结果也是一样的**，这就是一种**HTAP**（Hybrid Transaction and Analytical Process，混合事务和分析处理）的实现方法（比如TiDB就是利用这种方式来巧妙的实现自己的HTAP特性的）。















































# 3. 状态简化

- 在任何时刻，每一个服务器节点都处于leader,follower或candidate这三个状态之一。
- 相比于Paxos，这一点就极大简化了算法的实现，因为Raft只需考虑状态的切换，而不用像Paxos那样考虑状态之间的共存和互相影响

![在这里插入图片描述](/images/image-20221125163044598.png)



> 状态描述:
>
> 1. 所有节点开始的时候**都处于Follower状态**,此时,第一个认识到集群中没有Leader的节点会把自己变成candidate
> 2. 节点处于Candidate状态时,会发生一次或多次选举,最后根据选举结果决定自己时切换回Follower状态还是切换到Leader状态
> 3. 如果切换到Leader状态,就会在**Leader状态为客户端提供服务**
> 4. 如果节点在Leader状态的任期结束,或者是节点宕机亦或者其他的问题,就会切换回Follower状态,并开始下一个循环

- Raft把时间分割成任意长度的**任期（term）**，任期用连续的整数标记。
- 每一段任期从一次选举开始。在某些情况下，一次选举无法选出leader(比如两个节点收到了相同的票数,如下图$t_3$），在这种情况下，这一任期会以没有leader结束；一个新的任期(包含一次新的选举）会很快重新开始。Raft保证在任意一个任期内，最多只有一个leader。

![在这里插入图片描述](/images/image-20221125163931237.png)



> **任期的机制可以非常明确地表示集群的状态,而通过任期的比较,也可以确立一台服务器历史的状态**
>
> 比如我们可以通过查看一台服务器是否具有在$t_2$任期内的日志,判断该服务器在$t_2$任期内是否宕机





- Raft算法中服务器节点之间使用**RPC**进行通信，并且Raft中只有两种主要的RPC:
- **RequestVoteRPC（请求投票）**：由candidate在选举期间发起。
- **AppendEntriesRPC(追加条目)**：由leader发起，用来复制日志和提供一种心跳机制。
- 服务器之间通信的时候会**交换当前任期号**；如果一个服务器上的当前任期号比其他的小,该服务器会将自己的任期号更新为较大的那个值。
- 如果一个candidate或者leader发现自己的任期号过期了，它会立即回到follower状态。
- 如果一个节点接收到一个包含过期的任期号的请求，它会直接拒绝这个请求。

> 相比其他共识算法十多种的通信类型,Raft算法的精简设计,极大减少了理解和实现的成本











































# 4. 领导者选举

![在这里插入图片描述](/images/image-20221125170538463.png)

> 信息解读:
>
>     1. $S_5$是一个Leader,它向其它所有server**发送心跳消息**,来维持自己的地位
>
>     2. 如果一个Server在它的进度条读完之前仍没有收到$S_5$的心跳消息的话,该server就会认为系统中没有可用的leader,然后开始选举.
>
>     3. 开始一个选举过程后，follower**先增加自己的当前任期号**，并转换到**candidate**状态。然后**投票给自己**，并且并行地向集群中的其他服务器节点发送投票请求（`RequestVote RPC`)。
>
>     4. 最终会有三种结果:
>
>        - 它获得**超过半数选票**赢得了选举-> 成为Leader并开始发送心跳(告知集群中存在Leader),结束投票选举阶段
>
>        - 其他节点赢得了选举->收到**新leader的心跳**后，如果**新leader的任期号不小于自己当前的任期号**(任期号大概率相等)，那么就从candidate回到follower状态。
>
>        - 一段时间之后没有任何获胜者->每个candidate都在一个自己的**随机选举超时时间**后增加任期号开始新一轮投票。
>
>    5. 为什么会没有获胜者？
>       - 比如有多个follower同时成为candidate，得票太过分散，没有任何一个candidate得票超过半数,进入下一轮选举.
>       - **<font color = red>"注意:当前选举阶段并没有产生任何Leader"这个结论不需要集群所有节点对此产生共识,而是通过每个candidate都在等待一个随机选举超时时间之后,默认去进入下一个选举阶段。</font>**
>
>    6. 论文中给出的随机选举超时时间为 **150~300ms**,这意味着如果candidate没有收到超过半数的选票,也没有收到新Leader的心跳,那么他就会在150到300毫秒之间随机选择一个时间再次发起选举.

```go
//请求投票RPC Request,由candidate发起
type RequestVoteRequest struct{
    term			int		//自己当前的任期号	所有节点都带有任期号,因为raft的节点要通过任期号来确定自身的状态,以及判断接不接收这个RPC.
    candidateld 	int 	//自己的ID			Follower需要知道自己投票给谁
    lastLogIndex 	int 	//自己最后一个日志号
    lastLogTerm 	int 	//自己最后一个日志的任期
}
//请求投票RPC Response,由Follower回复candidate
type RequestVoteResponse struct{
	term		int		//自己当前任期号
	voteGranted bool 	//自己会不会投票给这个candidate
}
```

> Follower的投票逻辑:
>
> 1. 所有节点开始的时候**都处于Follower状态**,此时,第一个认识到集群中没有Leader的节点会把自己变成candidate
> 2. 他会给自己的**任期号加一并发请求投票request**给其他follower。
> 3. 收到一个requestVoteRequest之后会先校验这个candidate是否符合条件。
>
>    - term是否比自己大?
>
>    - 与Request的后两个字段相关,我们之后再进行讲解
> 4. 确认无误后开始投票,没有成为candidate的follower节点，对于同一个任期，会按照**先来先得**的原则投出自己的选票。
> 5. 为什么RequestVoteRPC中要有candidate最后一个日志的信息呢，**安全性**子问题中会给出进一步的说明。















































# 5. Raft日志复制(重点)

- Leader被选举出来后，开始为客户端请求提供服务。

- 客户端怎么知道新leader是哪个节点呢?

  - 非常容易解决,Client仍然向老节点发送请求,此时,会有三种情况
    - 这个节点恰好是Leader
    - 这个节点是Follower,Follower可以通过心跳得知Leader的ID,据此告知Client该找哪个节点
    - 这个节点宕机了,此时Client会向其它任一节点发送请求,重复这个过程
  - 也有一些比如设置第三方节点的做法,但这里不做说明,有兴趣的同学可以去自行了解

- Leader接收到客户端的指令后，会把指令作为一个新的条目追加到日志中去。

- 一条日志中需要具有三个信息：

  - **状态机指令**
  - **leader的任期号**(对于检测多个日志副本之间的不一致情况和判定节点状态,都有重要作用)
  - 日志号（日志索引,区分日志的前后关系）

  > 只有任期号和日志号一起看才能确定一个日志

  - Leader**并行**发送AppendEntries RPC给follower，让它们复制该条目。当该条目被**超过半数**的follower复制后，leader就可以在**本地执行该指令并把结果返回客户端**。
  - 我们把本地执行指令，也就是leader应用日志与状态机这一步， 称作**提交**。

![在这里插入图片描述](/images/image-20221125174946743.png)

1. 在上图中,最上面的一行代表Leader,其余四行代表Follower,可以观察到,Follower节点的进度不一定是一致的
2. 但是在这里,只要有三个节点(包括Leader在内)复制到了日志,Leader就可以提交了,在上图中可以提交的日志号到7
3. 在此过程中，leader或follower随时都有崩溃或缓慢的可能性，Raft必须要在有宕机的情况下继续支持日志复制，并且保证每个副本日志顺序的一致（以保证复制状态机的实现）。具体有三种可能：

   1. 如果有follower因为某些原因**没有给leader响应**，那么leader会不断地**重发**追加条目请求(**AppendEntries RPC**)，哪怕leader已经回复了客户端。
   2. 如果有**follower崩溃后恢复**，这时Raft追加条目的**一致性检查**生效，保证follower能按顺序恢复崩溃后的缺失的日志。
      - Raft的**一致性检查**：leader在每一个发往follower的追加条目RPC中，会放入**前一个日志条目的索引位置和任期号**，如果follower在它的日志中找不到前一个日志，那么它就会拒绝此日志，leader收到follower的拒绝后，会发送前一个日志条目，从而**逐渐向前定位到follower第一个缺失的日志。**
   3. 如果**leader崩溃**，那么崩溃的leader可能已经复制了日志到部分follower但还**没有提交**,而被选出的新leader又可能不具备这些日志,这样就有**部分follower中的日志和新leader的日志不相同。**

![在这里插入图片描述](/images/image-20221125183554411.png)

对于上述问题的第三点,我们以上图举例:

- 可以发现,此时follower中的c和d比leader还多出两个日志,那么为什么leader没有多出的日志还可以当选leader呢?
- 是因为c和d多出的日志还没有提交,也就不构成多数.
- 在这七个节点的集群中,leader可以依靠a,b,e和自己的选票当选leader(当然f也可以投票,因为a,b,e,f的任期都小于等于leader)
- 再看最后一个节点f(宕机的leader节点),它具有2,3任期的日志,别的节点都不具有,这意味着它在这两个任期内担任leader.
- 但是它在2,3任期内的日志都没有正常复制到大多数节点,也就没有提交.
- 这时,如果f恢复了,即便它在2,3任期的日志与leader不同,也不会产生冲突,因为Raft在这种情况下，leader通过**强制follower复制它的日志**来解决不一致的问题，这意味着follower中跟leader冲突的日志条目会被新leader的日志条目覆盖（因为没有提交，所以不违背外部一致性）。
- 这样图中的c,d,e,f节点中与leader不同的日志,最终都会被覆盖掉.
- 也有可能当前的leader宕机,这个时候a,c,d是有机会当上leader的,如果c,d当选leader,就可以把多出的日志复制给follower,来使自己多出的日志提交

**总结一下:**

- 通过这种机制，leader在当权之后就**不需要任何特殊的操作**来使日志恢复到一致状态。
- Leader只需要进行正常的操作，然后日志就能在回复AppendEntries一致性检查失败的时候**自动**趋于一致。
- Leader从来不会覆盖或者删除自己的日志条目。（Append-Only)
- 这样的日志复制机制，就可以保证一致性特性：
  - 只要过半的服务器能正常运行，Raft就能够接受、复制并应用新的日志条目;
  - 在正常情况下，新的日志条目可以在一个RPC来回中被复制给集群中的过半机器;
  - 单个运行慢的follower不会影响整体的性能。

```go
//追加日志RPC Request
type AppendEntriesRequest struct {
    term			int				//自己当前的任期号
    leaderld 		int				//leader(也就是自己)的ID,告诉follower自己是谁
    prevLogIndex 	int 			//前一个日志的日志号		用于进行一致性检查
    prevLogTerm 	int 			//前一个日志的任期号		用于进行一致性检查,只有这两个都与follower中的相同,follower才会认为日志是一致的
        							//如果只有日志号相同,这可能就是上图中f的情况,依旧需要向前回溯
    entries 		[]byte			//当前日志体,也就是命令内容
    leaderCommit 	int				//leader的已提交日志号
}
//追加日志RPC Response
type AppendEntriesResponse struct{
    term				int				// 自己当前任期号
    success 			bool			//如果follower包括前一个日志,则返回true
}
```

> 提交详解:
>
> Request:
>
> 1. 对于follower而言,接收到了leader的日志,并不能立即提交,因为这时候还没有确认这个日志是否被复制到了大多数节点。
> 2. 只有leader确认了日志被复制到大多数节点后,leader才会提交这个日志,也就是应用到自己的状态机中
> 3. 然后leader会在AppendEntries RPC中把这个提交信息告知follower(也就是上面的**leaderCommit**).
> 4. 然后follower就可以把自己复制但未提交的日志设为已提交状态(应用到自己的状态机中).
> 5. 对于还在追赶进度的follower来说,若leaderCommit大于自己最后一个日志,这时它的所有日志都是可以提交的
>
> Response:
>
> - 这个success标志只有在request的term大于等于自己的term,且request通过了一致性检查之后才会返回true,否则都返回false


> 项目地址在[https://github.com/JiuYou2020/MIT6.824-2A](https://github.com/JiuYou2020/MIT6.824-2A)



# 6. test_test.go解读

> 在开始实现之前我们可以先了解测试代码以便于我们进行实现

```go
func TestInitialElection2A(t *testing.T) {
	servers := 3
	//这段代码是用于初始化一个 Raft 集群的配置对象，它接收三个参数：测试对象 t、集群中服务器的数量 servers 和一个布尔型变量（第三个参数为 false）。
	//
	//make_config 函数会创建并返回一个名为 config 的 Config 实例，该实例包含多个 Raft 服务器实例以及一些辅助方法，这些方法可以在测试过程中检查集群状态、模拟网络故障和恢复等操作。在测试结束时，必须调用 config.cleanup() 方法来释放资源。
	//
	//整个测试过程分为两个部分，分别对应两个测试函数 TestInitialElection2A 和 TestReElection2A。在每个测试函数中，都会使用 make_config 函数来创建一个新的 Raft 集群配置，然后进行相关的测试操作。
	cfg := make_config(t, servers, false)
	defer cfg.cleanup()

	cfg.begin("Test (2A): initial election")

	// 是否选举出了一个领导者？
	cfg.checkOneLeader()
	fmt.Println("选举出了一个领导者")

	//为了避免与从节点学习选举结果竞争，等待一段时间后检查所有节点是否在选举中达成一致（即，它们同意的任期编号相同
	time.Sleep(50 * time.Millisecond)
	term1 := cfg.checkTerms()
	if term1 < 1 {
		t.Fatalf("term is %v, but should be at least 1", term1)
	}

	// 如果没有网络故障，领导者和任期编号是否保持不变？
	time.Sleep(2 * RaftElectionTimeout)
	term2 := cfg.checkTerms()
	if term1 != term2 {
		fmt.Printf("尽管没有出现故障，但任期编号发生了变化")
	}

	// 在执行完一定时间后，应该仍存在一个领导者。
	cfg.checkOneLeader()

	cfg.end()
}

func TestReElection2A(t *testing.T) {
	servers := 3
	cfg := make_config(t, servers, false)
	defer cfg.cleanup()

	cfg.begin("Test (2A): election after network failure")

	leader1 := cfg.checkOneLeader()

	// 如果领导者失去连接，应该会选举出一个新的领导者。
	cfg.disconnect(leader1)
	cfg.checkOneLeader()

	//如果旧的领导者重新加入集群，不应该干扰新的领导者。
	cfg.connect(leader1)
	leader2 := cfg.checkOneLeader()

	//如果没有足够的节点构成多数派，就不会选举出领导者。
	cfg.disconnect(leader2)
	cfg.disconnect((leader2 + 1) % servers)
	time.Sleep(2 * RaftElectionTimeout)
	cfg.checkNoLeader()

	// 如果多数派节点重新出现，它们应该能够选举出一个领导者。
	cfg.connect((leader2 + 1) % servers)
	cfg.checkOneLeader()

	// 最后一个节点重新加入集群不应该妨碍领导者的选举。
	cfg.connect(leader2)
	cfg.checkOneLeader()

	cfg.end()
}

```

1. 首先通过`cfg := make_config(t, servers, false)`创建三个节点再进行一系列动作
2. 跟进到make_config()函数中,我们可以发现:

```go
func make_config(t *testing.T, n int, unreliable bool) *config {
	...
	cfg.rafts = make([]*Raft, cfg.n)
	...

	// create a full set of Rafts.启动所有raft节点
	for i := 0; i < cfg.n; i++ {
		cfg.logs[i] = map[int]interface{}{}
		cfg.start1(i)
	}

	// connect everyone 在 StartAll() 方法中，还需要让每个节点连接其他节点的 RPC 客户端和服务端。这里使用了 Raft 库提供的 Connect() 方法：
	for i := 0; i < cfg.n; i++ {
		cfg.connect(i)
	}

	return cfg
}
```

3. 我们可以知道,它是通过raft中的Make()函数来创建raft节点,在通过Start()函数来启动所有raft节点的

# 7. 实现

各个结构体的实现

```go
//节点状态
type Status int

//投票状态
type VoteStatus int

//全局心跳超时时间
var HeartBeatTimeout = 120 * time.Millisecond

//raft节点类型：跟随者，竞选者，领导者
const (
	Follower Status = iota
	Candidate
	Leader
)

type LogEntry struct {
	Term    int
	Command interface{}
}

//
// 一个实现单个Raftraft节点的Go对象。
//
type Raft struct {
	mu        sync.Mutex          // 锁，用于保护共享访问此对等体状态
	peers     []*labrpc.ClientEnd // 所有raft节点的RPC端点
	persister *Persister          // 用于保存此对等体持久化状态的对象
	me        int                 // 此raft节点在peers[]中的索引
	dead      int32               // 由Kill（）设置

	// 在这里添加您的数据（2A、2B、2C）。
	// 参考论文图2，描述Raft服务器必须维护的状态。
	currentTerm   int           //当前任期
	voteFor       int           //当前任期把票投给了谁
	logs          []LogEntry    //日志条目，每个条目包含了命令，任期等信息
	commitIndex   int           //已提交的最大日志条目索引
	lastApplied   int           //最后应用到状态机的日志条目索引
	nextIndex     []int         // nextIndex是一个数组，它记录了每个Follower节点下一个要发送给它的日志条目的索引。
	matchIndex    []int         // 各个节点已知的最大匹配日志条目索引
	status        Status        //当前raft节点角色
	electionTimer *time.Timer   // 选举超时定时器
	applyChan     chan ApplyMsg //raft节点通过这个存取日志
}

//
//示例RequestVote RPC参数结构。字段名称必须以大写字母开头！
//
type RequestVoteArgs struct {
	// Your data here (2A, 2B).
	Term         int //自己当前的任期号
	CandidateId  int //自己的ID
	LastLogIndex int //自己最后一个日志号
	LastLogTerm  int //自己最后一个日志的任期
}

//
//以大写字母开头
//
type RequestVoteReply struct {
	// Your data here (2A).
	Term        int  //自己当前任期号
	VoteGranted bool //自己会不会投票给这个candidate
}

//追加日志RPC Request
type AppendEntriesArgs struct {
	Term         int        //自己当前的任期号
	LeaderId     int        //leader(也就是自己)的ID,告诉follower自己是谁
	PrevLogIndex int        //前一个日志的日志号		用于进行一致性检查
	PrevLogTerm  int        //前一个日志的任期号		用于进行一致性检查,只有这两个都与follower中的相同,follower才会认为日志是一致的
	Entries      []LogEntry //当前日志体,也就是命令内容
	LeaderCommit int        //leader的已提交日志号
}

//追加日志RPC Response
type AppendEntriesReply struct {
	Term          int  // 自己当前任期号
	Success       bool //如果follower包括前一个日志,则返回true
	ConflictIndex int
}
```





我们可以通过我的debug过程来清晰的了解到lab2A是如何实现的

1. 首先,我将Make()中的`rf.logs = make([]LogEntry, 1)`改为`rf.logs = make([]LogEntry, 0)`,人工制造一个错误,使得不至于输出太多
2. 运行测试代码,fmt输出如下:

```txt
Test (2A): initial election ...
节点 1 当前状态： 0 当前任期： 0 投票对象： -1
节点 2 当前状态： 0 当前任期： 0 投票对象： -1
节点 0 当前状态： 0 当前任期： 0 投票对象： -1
节点 1 选举超时
节点 1 当前状态： 1 当前任期： 0 投票对象： -1
节点 1 成为候选人
节点 1 向其他服务器发送投票请求
节点 1 向其他服务器发送投票请求
节点 1 向节点 2 请求投票，请求节点当前任期是 1 ID为 1 自己最后一个日志号为 -1 自己最后一个任期为 0
节点 1 向节点 0 请求投票，请求节点当前任期是 1 ID为 1 自己最后一个日志号为 -1 自己最后一个任期为 0
节点 2 收到投票，节点 2 当前任期为 0 当前状态为 0 请求节点任期为： 1 请求节点ID为 1
节点 0 收到投票，节点 0 当前任期为 0 当前状态为 0 请求节点任期为： 1 请求节点ID为 1
节点 1 收到投票回复： true {任期 是否同意投票} {1 true}
节点 1 收到投票 2 张，成为领导人
节点 1 收到投票回复： true {任期 是否同意投票} {1 true}
节点 1 当前状态： 2 当前任期： 1 投票对象： 1
节点 1 成为领导人
节点 1 发送心跳消息或附加日志的请求
panic: runtime error: index out of range [-1]

goroutine 22 [running]:
_/home/jiuyou/GolandProjects/6.824/src/raft.(*Raft).broadcastAppendEntries.func1(0x2)
        /home/jiuyou/GolandProjects/6.824/src/raft/raft.go:492 +0x986

```

3. 了解完上面代码,我们可以除去错误,继续测试查看输出

```txt
//类似这样,由于输出太多,这里显示一部分
节点 1 选举超时
节点 1 当前状态： 1 当前任期： 203 投票对象： 0
节点 1 成为候选人
节点 1 向其他服务器发送投票请求
节点 1 向其他服务器发送投票请求
节点 1 向节点 2 请求投票，请求节点当前任期是 204 ID为 1 自己最后一个日志号为 0 自己最后一个任期为 0
节点 1 向节点 0 请求投票，请求节点当前任期是 204 ID为 1 自己最后一个日志号为 0 自己最后一个任期为 0
节点 2 收到投票，节点 2 当前任期为 203 当前状态为 0 请求节点任期为： 204 请求节点ID为 1
节点 1 收到投票回复： true {任期 是否同意投票} {204 true}
节点 1 收到投票 2 张，成为领导人
节点 0 收到投票，节点 0 当前任期为 203 当前状态为 2 请求节点任期为： 204 请求节点ID为 1
节点 1 收到投票回复： true {任期 是否同意投票} {204 true}
节点 1 当前状态： 2 当前任期： 204 投票对象： 1
节点 1 成为领导人
节点 1 发送心跳消息或附加日志的请求
节点 1 前一个日志： 0 0
节点 1 向节点 2 发送心跳消息或附加日志的请求
节点 1 向节点 2 请求追加日志，追加节点当前任期为 204 ID为 1 自己最后一个日志号为 0 自己最后一个任期为 0 已提交的日志： 0 日志长度： 0
节点 1 前一个日志： 0 0
节点 1 向节点 2 发送心跳消息或附加日志的请求
节点 1 向节点 0 请求追加日志，追加节点当前任期为 204 ID为 1 自己最后一个日志号为 0 自己最后一个任期为 0 已提交的日志： 0 日志长度： 0
节点 0 准备添加日志 节点 0 当前任期为 204 当前状态为 0 请求节点任期为： 204 请求节点ID为 1
节点 0 添加日志成功 节点 0 当前任期为 204 当前状态为 0 请求节点任期为： 204 请求节点ID为 1
节点 2 准备添加日志 节点 2 当前任期为 204 当前状态为 0 请求节点任期为： 204 请求节点ID为 1
节点 2 添加日志成功 节点 2 当前任期为 204 当前状态为 0 请求节点任期为： 204 请求节点ID为 1
节点 1 收到节点 2 心跳消息或附加日志的回复 true {任期 成功与否 冲突地址} {204 true -1}
节点 1 更新 nextIndex 和 matchIndex 数组 [1 1 1] [0 0 0]
节点 1 收到节点 2 心跳消息或附加日志的回复 true {任期 成功与否 冲突地址} {204 true -1}
节点 1 更新 nextIndex 和 matchIndex 数组 [1 1 1] [0 0 0]
节点 1 收到投票回复： false {任期 是否同意投票} {0 false}
节点 0 收到节点 2 心跳消息或附加日志的回复 false {任期 成功与否 冲突地址} {0 false 0}
节点 2 收到投票回复： false {任期 是否同意投票} {0 false}
节点 2 收到投票回复： false {任期 是否同意投票} {0 false}
节点 2 收到投票回复： false {任期 是否同意投票} {0 false}
节点 1 收到投票回复： false {任期 是否同意投票} {0 false}
节点 0 当前状态： 0 当前任期： 204 投票对象： 1
节点 1 收到投票回复： false {任期 是否同意投票} {0 false}
节点 1 收到投票回复： false {任期 是否同意投票} {0 false}
节点 1 收到投票回复： false {任期 是否同意投票} {0 false}
```

1. 可以观察到,如果一个raft节点当选Leader,它会不停的向其它Follower节点发送心跳消息和日志,如果在一个心跳时间之类没有收到任何回复就会认为选举超时,重新开始选举

# 7. 2A需要注意的细节

1. 清楚选举计时器的重置情况
   - 当节点在 Follower 状态下接收到来自 Leader 的心跳消息时，应该重置选举计时器。这是因为接收到心跳消息意味着 Leader 还活着，因此不需要启动新的选举。
   - 当节点在 Candidate 状态下向其它节点发送投票请求时，也应该重置选举计时器。这是因为节点需要在选举计时器超时之前收到大多数节点的投票才能成为 Leader，因此在发送投票请求时应该重置选举计时器，以便在超时之前收到足够的投票。
2. `rf.logs = make([]LogEntry, 1)`明白这里为什么要为1,//logs初始化为一个空的日志，是因为在一开始启动时，Raft集群中的所有节点都是follower角色，还没有进行过日志条目的复制。此时，如果leader向follower发送AppendEntries消息来同步日志，则 PrevLogIndex 为 -1 是无效的。
3. `randomTime := time.Duration(rand.Intn(150) + 150)`,设计指导上也有指出,选举计时器应设为150ms到300ms的随机值,避免多个节点同时发生选举超时
4. 应该注意对共享资源的保护
5. `time.Sleep(10 * time.Millisecond) // 减少竞争，避免出现死锁`
