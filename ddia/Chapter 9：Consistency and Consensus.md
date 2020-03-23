# Chapter 9: 一致性

>*Is it better to be alive and wrong or right and dead?*
>—Jay Kreps, *A Few Notes on Kafka and Jepsen* (2013)

上一章节中（Chapter 8: The Trouble with Distributed System ）讨论了分布式系统会遇到的各种问题，包括

- 系统发出的报文，可能会延迟到达远端，并且有可能丢失。同理，无法得知响应报文是远端系统没有接受到报文，或则是没有响应，或者是响应报文被丢弃了。
- 节点的时钟无法与其他节点的始终保持一致（即使NTP正常工作），因而在需要保证完全一致性的场景下，依赖于节点时钟是不可靠的。
- 进程由于Stop-The-World GC，系统负载过高等原因假死，暂停，被其他远端认为节点已经宕机，之后进程恢复服务，但是进程本身并不知道自己暂停服务过。

本章节将会讨论如何分布式系统如何处理解决类似的问题



## 一致性保证 Consistency Guarantees 

在**Chapter 5：复制**中提到过，由于写入请求到不同的节点的时间是不一致的，**因而在同一时间**，我们在不同的节点上看到的数据可能是不一致的。复制系统中出现不一致性并不取决于系统使用的复制模型（单leader，多leader，或者是leaders less系统）。

绝大多数数据库复制系统提供**最终一致性**（eventual consistency）。

> Eventual Consistency means that if you stop writing to the database and wait for some unspecified length of time, then eventually all read requests will return the same value。

对于应用层而言，使用最终一致性的存储系统会导致，应用set一个新值，系统返回成功，但是随后再次读取可能获得的是旧值。

本章节会探索数据库系统能够提供的更强的一致性模型。更强的一致性模型并不是免费的，我们需要进行权衡取舍，我们会从以下的内容进行展开

- 我们首先了解**线性一致性**（*linearizability *），最强的一致性性模型之一，以及它的优点和缺点
- 审视在分布式系统中对事件排序所遇到的问题
- 探索如何实现分布式事务

## 线性一致性（Linerizability）

线性一致性（又称，原子一致性，强一致性，外部一致性等）的基本想法是：

> 让分布式存储系统看上去只存在一份数据拷贝。所有的读写操作都是原子性的

考虑以下的案例

![image-20200323110333468](http://liang2020.oss-cn-hangzhou.aliyuncs.com/uPic/blog/image-20200323110333468.png)

Alice先于Bob进行查看，已经获得最终结果，Bob迟于Alice查看，却并未获得最终得结果，如果只存在一份数据拷贝，则不会出现以上的场景。以上的场景并不符合线性一致性。

### 符合线性一致性的系统

符合线性一致的系统应该满足以下约束

1. 如果某一个write操作与read操作并发，则read的操作可能会返回新值或是旧值

2. 如果一个read操作读取到了新值，所有之后的read操作也只会读取到新值，即使新的read操作与写入新值的write操作依然是并发的

第二条参照下图，client A的第二次read，Client B的第二次read都与Client C的write操作并发。Client A的read结果可以是新值或是旧值，下图中Client A返回了新值，因而在之后读取的Client B返回的结果必须时是新值。

![image-20200323110354528](http://liang2020.oss-cn-hangzhou.aliyuncs.com/uPic/blog/image-20200323110354528.png)



> **线性一致性的原子性与事务的原子性的区别：**
>
> 事务的原子性指的是一批操作被原子性得递交，或者成功，或者不成功。
>
> 线性一致性的原子性指的是Client提交的所有读写操作在系统中**原子性**生效



下面的实例中，标注出具体的write生效的时间点。

![image-20200323110408343](http://liang2020.oss-cn-hangzhou.aliyuncs.com/uPic/blog/image-20200323110408343.png)

- 上图中Client A的read操作与Client C的cas操作并发，并且返回了新值，因而在之后的读取操作，都需要返回新值，也就是x=4，才能符合线性一致性要求。因而Client B读取的x=2是不符合线性一致性的。





### 线性一致性的应用

- Leader选举以及分布式锁

分布式系统中，很多时候需要确认一个leader进行一系列操作，多个leader则导致系统出现脑裂（split brain），出现数据不一致。在类似的场景下，需要所有的node节点达成一致，只有一个节点能够持有锁，成为leader。

分布式协调服务，例如Apache ZooKeeper，etcd，Google的Chubby（非开源）提供了分布式锁以及leader选举功能，他们通过一致性算法实现线性一致性。

分布式锁在许多分布式数据库中被应用。例如Oracle Real Application Cluster(RAC)。RAC使用锁保护每一个磁盘页，多个计算机节点可以共享同一个磁盘存储系统。

- 约束保证

许多应用场景需要唯一性约束，例如注册用的邮箱是唯一的，注册用户名也需要是唯一的，类似的场景下存储的系统需要满足线性一致性，插入的操纵类似compare-and-set，需要获得最新的唯一性约束字段，然后在更新，过程中，读取到的值应该是最新的，只有字段数值没有被占用，操作才能成功。

保证飞机座位不会超卖，银行账户的余额不为负，买出的股票不会操作账户持有数目，类似的场景约束中，都需要存储系统有一个up-to-date的，并且其他节点都认同的值，类似的场景下也需要线性一致性。

- 时间依赖的多渠道

一开始的案例中，Alice和Bob由于复制系统的延迟，导致他们获得结果不一致，而这个不一致在他们二者相互通讯的场景下，导致了问题。

![image-20200323110541050](http://liang2020.oss-cn-hangzhou.aliyuncs.com/uPic/blog/image-20200323110541050.png)

参考以上的案例，用户上传图片到分布式文件系统，同时也有一个消息服务进行图片的resize工作。如果在消息服务获取得数据是文件系统中得一个没有最新版本图片得节点，则会导致他将老得图片进行resize，然后存储回文件系统。

这两个案例本质上是类似得，都是由于不同的client都会去读写的同一条消息，并且他们之间存在时间依赖。线性一致性并不是解决类似问题的唯一方法，不过它是一个简单明了的方案。



## 线性一致性的实现

符合线性一致性的系统，**如同只有一份数据的拷贝，并且所有的操作是原子性的**。因而将只存储一份数据拷贝，是比较简单的实现线性一致的方案。不过类似的方案服务承受数据所在的分区节点宕机，系统将完全不可用。最为常见的数据高可用方案是复制，以下我们将回顾下各种复制系统，以及他们是否能够实现线性一致性。

- 单leader复制系统（可实现线性一致性）

单leader复制系统中，存在一个leader以及多个follower，数据先写入leader，再通过复制日志同步到follower。如果读取完全通过leader完成，或者从与leader同步复制的follower读取数据，系统可能是线性一致的。

reader使用leader作为数据源进行去读的一个前提是，reader知道哪一个节点是reader。但是系统有可能出现fail-over，使用新的leader，而旧的leader依然为reader进行服务，这将破坏系统的线性一致性。

- 一致性算法（符合线性一致性）

一致性算法，在后续的章节中我们会进行讨论。它能够容忍单leader系统出现的系统重新进行角色分配，防止脑裂，以及读取过期副本。

- 多leader复制（不是线性一致的）

多leader系统中，可能对于同一个数据对象进行写入操作，然后通过异步复制同步给其他节点。它不符合线性一致性。

- leaderless复制（可能不符合线性一致性）

在leaderless系统，例如Amazon的Dynamo，有人认为，只要符合r+w>n，系统是符合强一致性的。

Last write wins是Leaderless系统解决冲突的一种策略（例如，Cassandra），它依赖于日期时间戳，由于不同数据节点的日志不能保证一致，因而类似的策略是不符合线性一致性的。而Sloppy quorums(草率仲裁)也同样会影响系统的线性一致性。不过即使使用严格的仲裁，系统依然不能满足线性一致性。

### 线性一致性与多数派仲裁

![image-20200323110556265](http://liang2020.oss-cn-hangzhou.aliyuncs.com/uPic/blog/image-20200323110556265.png)

以上的Dynamo-style模型是，使用的配置是n=3，w=3，r=1。Reader A能够读取到新的值，Reader B在之后执行，但是由于读取到延迟的节点，因而读取到旧值。这个过程中，符合严格仲裁的要求（w + r > n)， 不过却不符合线性一致性。

如果reader在读取之前，做一次read repair操作，则其能够读取到最新的值，不过这也会带来较高的性能损失。Riak不会使用同步的read repair操作，由于操作可能带来的性能损失。Cassandra可以配置等待read repair完成后进行仲裁读取，不过由于可能存在的并发写入以及使用last-write-win策略解决冲突，它并不能符合线性一致性。

以上的方案，只能适用于read或是write操作，不能实现符合线性一致性的compare-and-set操作，它的实现需要依赖一致性算法。

