## 一 raft协议概述

raft用于解决分布式系统中一致性问题，raft基于复制，其工作机制包括：leader选举、日志复制、安全性等。  

分布式系统中每个节点都有自己的角色：
- Leader：
- Follower：
- Candicate：候选者。在选举阶段才会存在，正常工作中不存在该节点。 

在raft协议中，会将时间分割为一个一个任期（term）。  

复制状态机：如果分布式系统每个节点都执行相同的命令序列，那么整个系统就能保持一致性，即每个节点的日志要保持一样。所有，保证日志复制一致，就是raft算法的工作。  

心跳与超时机制：主要依赖2个超时机制来实现Leader选举
- 选举定时器（eletion timeout）：即Follower等待称为Candidate状态的等待时间，这个时间被随机设定为150ms~300ms之间
- 心跳超时：

Leader选举过程:
- 所有节点以Follower角色启动，同时每个节点启动自己的选举定时器（时间随机，以降低冲突概率）
- Follower节点的选举定时器时间到达时间后，就会变为Candicate，触发选举过程
- Candicate把当前任期+1，为自己进行投票，并发起RPC请求，要求其他节点也为自己投票，当得到半数以上的票时称为Leader
- 如果任意时间到了，但是还没有选出Leader，则进入下一轮选举。
- 如果称为了Leader，则开始工作（执行客户端的写入），并将数据同步到到Follower


选举规则:
- 1 每个节点在一个任期内,最多能给一个候选节点投票,且采用谁的投票请求会先到,就给谁先投票
- 2 如果没有投过票,则对比后选择的日志与当前节点的日志哪个更新,谁的最近log的term越大,谁就越新,如果term相同,则谁的最近日志的index越大,谁越新,如果当前节点更新,则拒绝投票.

选举结束后，客户端的命令到达Leader，开始进行日志复制
- Leader首先将命令追加到本地日志，此时命令处于 uncomitter 状态，复制状态机不会执行该命令
- Leader将命令复制给其他节点，并等待其他节点将命令写入日志
  - 如果有节点失败、响应慢，会一直重试，直到所有节点都保存了命令到日志中
- Leader提交命令，即状态机开始执行命令，并将结构返回给Client节点
- Ledaer提交命令后，下一次的心跳包中就带有通知其他节点提交命令的消息，其他节点收到Leader的消息后，就将命令应用到自己的状态机中，最红每个节点的日志都保持了一致性

安全性：一个节点要赢得选举，需要与网络中的大部分节点进行通信。如果候选人的日志至少和大多数服务器上的日志一样新，那么它一定包含所有的已提交的日志条目。ReuqestVote RPC实现了这个限制：这个RPC包括候选人的日志信息，如果节点自己的日志比候选人的日志新，则会拒绝候选人的投票请求。判断日志新的标准：
- 任期号大的日志较新
- 任期号一样大，则根据日志最后一个命令的索引index，谁大谁新


