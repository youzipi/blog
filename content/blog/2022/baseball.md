---
title: 'baseball'
date: 2021-11-06 20:06:00
updated: 2022-10-08 14:44:30
categories: 
- paper-reading
tags: 
- ARIES
- paper
- db
description: ARIES 如何使用 WAL 实现 原子性和一致性
draft: true
---

- DONE clock  
:logbook:
	  CLOCK: [2022-07-15 Fri 11:02:22]--[2022-07-18 Mon 15:36:34] =>  76:34:12
:END:
- ## Abstract  
- During a baseball game, for example, different participants (the scorekeeper, umpire, sportswriter, and so on) benefit from six different consistency guarantees when reading the current score.  
	- 全文基于这个类比：棒球比赛中，不同角色的工作人员对于当前的比分，需要不同的一致性保证。  
	  进而说明了不同的一致性保证的存在价值。  
- ## Introduction  
- 比较了一下 MS's Azure Storage 和 aws s3 提供的不同的保证  
	- azure: strongly consistency  
	- s3: weak, eventually consistency  
- Some recent systems, recognizing the need to support different classes of applications, `offer a choice` of operations of accessing storage.  
	- amazon's SimpleDB: `eventually consistent reads`, `consistent reads`  
	- GAE Datastore: `eventually consistent reads`, `strong consistency`  
	- Yahoo's PNUTS: `read-any`, `read-critical`, `read-latest`; `write`, `test-and-set-write`  
- The reason for exploring different consistency models is that there are `fundamental tradeoffs` between consistency, performance, and availability.  
- this paper attempts to answer these questions, by examining an example application: the game of baseball:  
	- are different consistencies useful in practice?  
	- Can application developers cope with eventual consistency?  
	- Should cloud storage systems offer an even greater choice of consistency than the consistent and eventually consistent reads offered by Amazon’s SimpleDB?  
- 具体地，我们假设 `比分` 存储在一个 `cloud-based`,`replicated` 的存储服务中，看一下，不同的角色需要怎样的一致性保证。  
- ## Read Consistency Guarantees  
- Frequently, one needs to understand `how a system operates` in order to understand `what consistency` it provides in `what situations`.  
- The six consistency guarantees that I advocate in this section can be described in a `simple`, `implementation-independent` way.   This not only benefits application developers but also can `permit flexibility` in the `design`, `operation`, and `evolution` of the underlying storage system.  
- [:span]  
- what is `consistent prefix`?  
- how client handle with the all `old` writes?  
- ### consistent prefix  
- [:span]  
	- 类似场景：微信朋友圈，a 评论；b 评论 a；c 先看到了 b 的评论，后看到了 a 的评论。  
	- 打破了因果关系  
	- consistent prefix 给出以下承诺：  
		- if a sequence of writes happens in a certain order,then anyone reading those writes will see them appear in the same order  
	- 可能的方案：  
		- 确保有因果关系的 write 发生在同一个 partition 上。  
		- capture the `happens-before` relationship  
			- [:span]  
			- [:span]  
			- client 负责 merge 不同版本的数据写入。  
			  面对的问题和 multi-leader replication 中的 冲突 类似。  
- a reader is guaranteed to observe `an ordered sequence of writes starting with the first write` to a data object.  
- in other words, the reader sees a version of the data store that existed at the master `at some time in the past`.  
- this is similar to the `snapshot isolation` offered by many db.  
- ### bounded staleness  
- late, but not too late.  
- ensures that read results are `not too out-of-date`.  
- typically, staleness if defined by a time period `T`, say 5 minutes.  
- alternative, some systems define staleness in terms of `the number of missing writes` or even `amount of inaccuracy` in a data value.  
- I find that time-bounded staleness is the `most natural concept` for application developers.  
- 如果 延迟设置为 0，那么就和 `strong` 一样，  
  如果 延迟设置为 无穷，那么就和 `eventual` 一样。  
- ### monotonic reads  
- is a property that applies to a sequence of read operations that are performed by `a given storage system client`.  
- aka: `session guarantee`  
- client is guaranteed to observe a data store that is `increasingly up-to-date over time`.  
- ### read my writes  
- is a property that also applies to a sequence of operations performed by a single client  
- It guarantees that the `effects` of all writes that were performed by the client are visible to the client’s `subsequent reads`.  
- ### 小结  
- These last four read guarantees are all `a form of eventual consistency` but `stronger than` the eventual consistency model  
- none of these four guarantees is `stronger than any of the others`.  
- applications may want to request `multiple` of these guarantees.  
- [:span]  
	- Labeling each cell in this table is not an exact science  
	- But, the general comparisons between the various consistency guarantees are qualitatively accurate  
	- 不用管每个单元格的描述用词 (比如：good，okey)，这个每个人的定义不一样，而且会受到具体实现方案的影响。  
	- 这里，只需要关注，每一列上的对比：对于特定需求，不同 guarantee 在 光谱上的排布。  
		- ![](https://docs.microsoft.com/en-us/azure/cosmos-db/media/consistency-levels/five-consistency-levels.png)  
	- 作者和他的微软同事已经实现了一个原型：在一个存储服务上，提供上面提到的所有 一致性保证。  
		- 这个原型，后来发展成了 `Cosmos DB`，在 2017 年发布。  
- ## Baseball as a Sample Application  
- 一场棒球比赛的流程抽象：  
	- [:span]  
- 比赛进行了一段时间之后：  
	- [:span]  
- 在不同的 一致性保证下，client 读到的结果：  
	- [:span]  
	- eventual consistency：会读到所有可能的组合。比如 2:0 这种。  
	- read my writes: 对于只写过 visitor 分数的 client，也会读到 2:0。  
	- bounded staleness: 是一致性比较好的，这里设置的最大延迟是：`1轮`。如果最大延迟设置为 `7轮`的话，client 可能看到的结果和 `eventual consistency` 一样。  
	- consistent prefix: 看到的比分都是正确的，至少是之前的某一时刻正确的。不会看到 2:0 这样的结果。  
		- scores that actually existed at some time.  
	- `consistent prefix` vs `monotonic reads`  
		- consistent prefix 强调的是 因果关系  
		  monotonic reads 强调的是 不会看到旧版本的数据  
		  比如 1-5 是不符合因果关系的，但是是满足 单调读的。（具体场景是 客队的数据写入落后了）  
	- `eventual consistency` 和 `read my writes` 不满足 `因果性`  
- ## Read Requirements for Participants  
- ### Official scorekeeper  
- [:span]  
- 记分员负责维护比赛的最新比分。  
- 一致性需求：需要 `strongly consistent data`，  
  但是，他是`唯一的`写入者，所以，只需要 `read my writes` 就可以了，  
  不需要 `strong consistency reads`。  
- 类比到系统中：  
	- 如果只有 唯一一个写入者，不需要在副本之间达成共识，执行 `strong consistency reads`，只需要把这个写入者`固定`在一个 session 上就可以。  
	- 如果有多个写入者，尝试 拆分 `partition`，看能不能在单个 partition 上，转换成 `唯一写入者` 的场景。  
	- 一个类比：[【Paper Reading 预告】Greenplum 6：A Hybrid Database for Transactional and Analytical Workloads](https://asktug.com/t/topic/695266) 直播中，看到一个实践优化：`greenplum 中，如果分布式事务只涉及一个 instance，2pc 降级为 1pc`  
- ### Umpire  
- [:span]  
- 比赛的大部分时候，裁判不关心场上的比分，唯一的例外是：  
  在 最后一轮的上半场结束后，也就是客队击球完毕，轮到主队开始击球时，如果主队得分领先，那么`比赛直接结束，主队获胜`。  
  (最后一轮下半场，是主队击球 - 跑垒 - 得分的轮次，客队整场比赛的得分机会已经用完了。)  
- (类似于：足球的点球大战，客队 xxx，主队 ooo，主队直接获胜，后面 4 次点球不需要了。）  
- 这里，裁判需要 `strong consistency`，否则可能会进行没必要的下半场。  
- ### Radio reporter  
- [:span]  
- 周期性广播比分，类似的例子：文字直播，电视上的底部轮播条。  
- 用户可以接受延迟，但不能接受比分的顺序错乱。  
- radio reporter 需要 `consistent prefix`，保证每一次读取的比分都是真实存在的，至少是`曾经真实存在的`。  
- 但这还不够，reporter 可能还会先读到 2-5，后读到 1-3，  
  当 reporter 的请求被负载到不同的 server 上时，可能会出现这样的结果。  
- 因此，reporter 需要保证自己的读同时满足 `consistent prefix` 和 `monotonic reads`。  
	- 只有 `consistent prefix` 是不行的。  
	- 只有 `monotonic reads` 也不行，对于真实顺序：`1-0, 2-0, 3-0, 3-1, 3-2, 3-3`，client 可能会读到 `0-1, 0-2, 0-3, 1-3, 2-3, 3-3`。  
- ### Sportswriter  
- [:span]  
- sportswriter 需要绝对正确的结果，但不急着要。  
  `strong consistency` 可以满足要求，但成本太高了。  
  `bounded staleness` + 1 小时的 bound，就足够了。  
- In fact, an `eventual consistency` read is `likely` to return the correct score after an hour,  
  but requesting `bounded staleness` is the only way for the sportswriter to be `100% certain` that he is obtaining the final score.  
	- 实际情况中，`eventual consistency` 在一个小时之后就可以提供正确的结果了。但是，它没有这样一个`一定正确`的保证，即使延迟一个世纪去拿数据，它也不能给你这个保证。  
	    
	  > this is going to be your league `in a little while`.	    
	  ![https://ftw.usatoday.com/2014/06/tim-duncan-lebron-james-nba-finals](https://fadeawayworld.net/.image/ar_4:3%2Cc_fill%2Ccs_srgb%2Cfl_progressive%2Cq_auto:good%2Cw_1200/MTkwMjE4ODExNzUwNzUzNjIy/duncan-james.jpg)  

- ### Statistician  
- [:span]  
- The team statistician is responsible for `keeping track` of the `season-long` statistics for the team and for individual players.  
- figure8 中的例子是：记录主队整个赛季的总得分。  
- 对于 `今天比赛的得分`，需要 `strong consistency` 或者 `bounded staleness`  
- 对于 `本赛季的总得分`，需要 `strong consistency`，如果只有一个分析师的话，`read-my-writes`也可以。  
- ### Stat watcher  
- 其他的角色，通常使用 `eventual consistency` 就够了。  
- 比如：球迷，想看下主队的赛季数据。  
- [:span]  
- ### conclusions  
- ![](https://user-images.githubusercontent.com/5629307/179383055-7ae58814-d9f6-402c-9373-618b471482d7.png)  
- In particular, each participant would `be okay` with strong consistency, but, by `relaxing the consistency` requested for his reads, he will likely observe `better performance and availability`.  
- Additionally, the storage system may be able to better `balance the read workload` across servers since it has more flexibility in selecting servers to answer weak consistency read requests.  
- client 降低自己对一致性的要求，可以在满足要求的同时，获得更快的响应和更高的可用性。  
- server 也可以更好地负载请求，主要是那些一致性要求低的请求。  
- 这个模型中，还展示了现实中 可能出现的 不同的 `client 访问 server 的方式`:  
	- scorekeeper, sportswriter: client，based on application-specific knowledge，知道，即使自己只要求`弱一致性的读取`(比如：`read-my-writes`,`bounded staleness`)，也可以保证能拿到`强一致性的数据`。  
	- radio reporter: client 需要为一个请求，提供多个保证。  
	- statistician: client 需要为不同的请求，提供不同的保证。  
	- 从这个例子中，我们可以总结出这些结论：  
		- All of the six presented consistency guarantees are useful.  
			- 如果系统只提供 `eventually consistency` 也可以满足一定的需求。  
			- 如果系统只提供 `strong consistency`，在很多情况下，太影响性能了，没必要。  
		- Different clients may want different consistencies even when accessing the same data.  
			- 很多时候，系统都为特定的数据，绑定同一种一致性保证。  
			- 比如：银行数据都要求强一致性。  
			- 但是，从棒球的例子中，我们看到，一致性需求除了取决于被访问的数据，还取决于`谁`访问这个数据，`为了什么`访问这个数据。  
		- Even simple databases may have diverse users with different consistency needs.  
			- 这个例子里，全部的数据就是 `2个数字`，但是，提供不同的一致性保证也能发挥价值。  
		- Clients should be able to choose their desired consistency.  
			- server 无法预测 client 需要的一致性保证。  
			- 一致性保证的要求取决于：`如何使用数据`，`谁写入的数据`，`数据最后一次更新的时间`。这些信息只能由 client 负责。  
	- what about the cost of eventual consistency? #TODO  
	- what should be offered by cloud storage providers?  
		- Allowing cloud storage clients to read from diverse replicas with `a choice of consistency guarantees` could benefit a broad class of applications as well as lead to `better resource utilization` and `cost savings`.  
- ## Additional Readings  
- 6.Additional Readings  
- 整理了各类一致性的实现和选择，价值很高。  
- ## summary  
- 更深入地理解了这句话：`ACID 中的 C，不是数据库的固有属性，更多是应用层的事情。`  
	- Atomicity,  isolation,  and  durability  are  properties  of  the  database,  whereas  consis‐tency (in the ACID sense) is a property of the application. The application may relyon  the  database’s  atomicity  and  isolation  properties  in  order  to  achieve  consistency,but it’s not up to the database alone. Thus, the letter C doesn’t really belong in ACID.i  
	- acm 的 review articles 模块里，有另外一版论文，多了一块 `key insights` 的总结： 
	  这篇文章的主要亮点是：`client 的主体性`。  
	  传统上，一致性都是配置在存储服务上的，但是，我们是不是可以把选择权交给`每一个client`，甚至 `每一个读请求`？  
	  一致性是这样？隔离性呢？再推广一下，还有哪些 `server guarantee`，我们可以配置到 `client`，`request` 层面？  
	  比如：前面提到的：pulsar 的 复制策略可以配在 broker 的 namespace 或者 topic 上，也可以配在 `producer.sendMsg` 上。  
	- although replicated cloud services generally offer strong or eventual consistency, intermediate consistency guarantees may better meet an application’s needs.  
	- consistency guarantees can be defined in an implementation-independent manner and `chosen for each read operation`.  
	- Dealing with relaxed consistency need not place an excessive burden on application developers.  
	- 当我们谈论 `consistency` 的时候，我们在谈论什么？  
	  对于`consistency`，并没有一个普适的共识。  
	  甚至，整个 `ACID` 都是如此，我们无法宽泛地进行讨论，得在前面加上多个限定：innoDB 中，单机部署下。。。  
	  《DDIA》关于这一点，提出了相当悲观的论调：  
	  各家数据库都说自己实现了 `ACID`，但是各家的实现都不一样，只有比如：`mysql 特色 ACID`。  
	  以至于，如果一个系统声称自己是 `ACID compliant`，你甚至没法判断到底它能提供什么样的保证。  
	  面对每一个这样的 `ACID`，我们都是`新手`。  
	  `ACID` 事实上已经变成了一个 `marketing term`。  
	  至于 `BASE`，就更加啥也不是了。它只能保证了自己`不是 ACID`，仅此而已。  
	- However,  in  practice,  one  database’s  implementation  of  ACID  does  not  equalanother’s  implementation.  For  example,  as  we  shall  see,  there  is  a  lot  of  ambiguityaround the meaning of isolation [8]. The high-level idea is sound, but the devil is inthe  details.  Today,  when  a  system  claims  to  be  “ACID  compliant,”  it’s  unclear  whatguarantees  you  can  actually  expect.  ACID  has  unfortunately  become  mostly  a  mar‐keting term.  
	- (Systems  that  do  not  meet  the  ACID  criteria  are  sometimes  called  BASE,  whichstands  for  Basically  Available,  Soft  state,  and  Eventual  consistency  [9].  This  is  evenmore vague than the definition of ACID. It seems that the only sensible definition ofBASE is “not ACID”; i.e., it can mean almost anything you want.)  
- ## refer  
- [Replicated Data Consistency Explained Through Baseball, Doug Terry 20140319_ytb](https://www.youtube.com/watch?v=gluIh8zd26I)  
	- 作者关于本篇论文，在 ACM 上的分享  
	- [30:49 开始分析，各个角色需要的一致性保证](https://youtu.be/gluIh8zd26I?t=1849)  
	- https://github.com/youzipi/blog/issues/3  
		- 截图了一些 ppt，主要比文章多了 每个一致性保证 对应的 `under the covers`  
- [Consistency levels in Azure Cosmos DB](https://docs.microsoft.com/en-us/azure/cosmos-db/consistency-levels)  
	- 这个是最新的文档  
- [Azure Cosmos DB 一致性级别_zhihui_zhuanlan](https://zhuanlan.zhihu.com/p/143227315)  
	- 这个是最新的文档的笔记  
- [A technical overview of Azure Cosmos DB](https://azure.microsoft.com/en-us/blog/a-technical-overview-of-azure-cosmos-db/)  
	- [Azure Cosmos DB 技术性解读_infoq 的中文版](https://www.infoq.cn/article/technical-interpretation-of-azure-cosmos-db)  
	- [Foundations of Azure Cosmos DB with Dr. Leslie Lamport_ytb](https://www.youtube.com/watch?v=L_PPKyAsR3w)  
		- 和文章一起的，lamport 的一小段 ~~采访~~(看提词板读广告。  
		- 讲了下 cosmos DB 是啥，cosmos DB 是第一个提供了`5个可选一致性保证`的服务。  
		  自己参与的主要是用 TLA+ 验证了这些一致性。  
		- lamport 的 tla 视频教程：  
			- 短链：[http://aka.ms/tla](http://aka.ms/tla)  
			  原链接：[https://lamport.azurewebsites.net/video/videos.html](https://lamport.azurewebsites.net/video/videos.html)  
		- lamport 同志讲话主要内容：  
			- Cosmos DB 是个好服务，xxx  
			- Cosmos DB 第一个提供了 5 种可选一致性保证  
			- 我用 TLA+ 帮忙验证了这些一致性保证的正确性  
			- 下面是我的 ~~公众号二维码~~ (TLA+ 视频教程）。  