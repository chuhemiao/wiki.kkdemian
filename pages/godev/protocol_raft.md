# Raft 协议是什么

## Raft 历史背景

- 在讲 Raft 前，有必要提一下 Paxos 算法，Paxos 算法是 Leslie Lamport 于 1990 年提出的基于消息传递的一致性算法。然而，由于算法难以理解，刚开始并没有得到很多人的重视。其后，作者在八年后，也就是 1998 年在 ACM 上正式发表，然而由于算法难以理解还是没有得到重视。而作者之后用更容易接受的方法重新发表了一篇论文《Paxos Made Simple》。

- 可见，Paxos 算法是有多难理解，即便现在放到很多高校，依然很多学生、教授都反馈 Paxos 算法难以理解。同时，Paxos 算法在实际应用实现的时候也是比较困难的。这也是为什么会有后来 Raft 算法的提出。

### Raft 定义（概念）

- Raft 是用于实现分散式系统的一种容易理解的共识机制，主要用来管理日志複製的一致性。

- Raft 的核心思想很简单：

- 如果在分散式系统中多个数据库的初始状态一致，只要之后进行的操作顺序一致，就能保证之后的执行结果一致。

- 试想一下，假设把三个数据库想像成是你面前的三本账本，只要每次在第一本账本上写上了一笔记录时，同时将这笔记录同样地写在另外两本账本上，那麽这三本账本的数据始终是一样的。

- Raft 取代複杂难懂的 Paxos 算法，并且证明可以提供与 Paxos 相同的容错性以及性能，并且更容易应用到实际系统中。

### Raft 集群中，伺服器的三种角色

![Raft 集群中，伺服器的三种角色](https://cdn.bsatoshi.com/2019/11/11/15734605986049.jpg)

- 一个 Raft 集群（Raft cluster）中，至少包含 5 个伺服器，允许系统有 2 个故障伺服器，在系统初始时，每个伺服器都处于追随者的状态。而每个伺服器将会是这三种角色其中一个：

- 领袖（Leader）：正常状态下只有一个领袖 ，其他都是追随者，它会负责处理所有来自外部 Client 的请求及进行日志複製（日志複製是单向的，即领袖发送给追随者），先就是负责"出块"的节点。

- 候选人（Candidate）：由追随者向领袖转换的中间状态，等待被选举为领袖。

- 追随者（Follower）：它是被动的节点，对来自 Client 和候选人的请求做出响应。如果收到外部 Client 的请求，就会将请求会被转发给领袖。

- 而这三种角色在条件满足下可以互相转换，而在正常情况下只会有一个领袖，其他都是追随者。而领袖会负责所有外部的请求。

### 几个关键概念

- 复制状态机

  - 我们知道，在一个分布式系统数据库中，如果每个节点的状态一致，每个节点都执行相同的命令序列，那么最终他们会得到一个一致的状态。也就是和说，为了保证整个分布式系统的一致性，我们需要保证每个节点执行相同的命令序列，也就是说每个节点的日志要保持一样。所以说，保证日志复制一致就是 Raft 等一致性算法的工作了。

- 任期（Term）概念

  - 在分布式系统中，“时间同步”是一个很大的难题，因为每个机器可能由于所处的地理位置、机器环境等因素会不同程度造成时钟不一致，但是为了识别“过期信息”，时间信息必不可少。

- Raft 算法中就采用任期（Term）的概念，将时间切分为一个个的 Term（同时每个节点自身也会本地维护 currentTerm），可以认为是逻辑上的时间，如下图。

![任期（Term）概念](https://cdn.bsatoshi.com/2019/11/11/15734607621598.jpg)

- 每一任期的开始都是一次领导人选举，一个或多个候选人（Candidate）会尝试成为领导（Leader）。如果一个人赢得选举，就会在该任期（Term）内剩余的时间担任领导人。在某些情况下，选票可能会被评分，有可能没有选出领导人（如 t3），那么，将会开始另一任期，并且理科开始下一次选举。Raft 算法保证在给定的一个任期最少要有一个领导人。

- 心跳（heartbeats）和超时机制（timeout）

  - 在 Raft 算法中，有两个 timeout 机制来控制领导人选举：

  - 一个是选举定时器（eletion timeout）：即 Follower 等待成为 Candidate 状态的等待时间，这个时间被随机设定为 150ms~300ms 之间

  - 另一个是 headrbeat timeout：在某个节点成为 Leader 以后，它会发送 Append Entries 消息给其他节点，这些消息就是通过 heartbeat timeout 来传送，Follower 接收到 Leader 的心跳包的同时也重置选举定时器。

### Raft 有几个新的特性

- 强领导者（Strong Leader）：Raft 使用一种比其他算法更强的领导形式。例如，日志条目只从领导者发送向其他服务器。这样就简化了对日志复制的管理，使得 Raft 更易于理解。
- 领导选取（Leader Selection）：Raft 使用随机定时器来选取领导者。这种方式仅仅是在所有算法都需要实现的心跳机制上增加了一点变化，它使得在解决冲突时更简单和快速。
- 成员变化（Membership Change）：Raft 为了调整集群中成员关系使用了新的联合一致性（joint consensus）的方法，这种方法中大多数不同配置的机器在转换关系的时候会交迭（overlap）。这使得在配置改变的时候，集群能够继续操作。

### 推荐阅读

- [Raft 一致性算法论文译文](https://www.infoq.cn/article/raft-paper/)
- [區塊鏈 Blockchain – 共識機制之 Raft](https://www.samsonhoi.com/583/blockchain-raft)
