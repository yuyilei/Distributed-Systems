### 6.824 2014 Lecture 5: Raft（1）

[《Lecture 5: Raft (1)》原文地址](https://pdos.csail.mit.edu/6.824/notes/l-raft.txt)

#### 本课程
+ 今天：Raft（lab2）
+ 下一步计划：在kv服务中使用Raft(lab3)

#### 整体的主题:使用复制状态机实现容错 (SRM ??)
+ [客户端、副本服务器]
+ 每一个副本服务器以相同的顺序执行一样的操作
+ 他们让副本的执行就行自己执行一样，当如果有失败的情况任何副本可以接管工作，比如：在失败的时候，客户端会切换到其他服务器，GFS和VMware FT都有这种“方式”（flavor）

#### 典型的“状态机”是怎么样的？
+ 应用程序/服务的内部状态、输入命令序列、输出都被复制
+ 顺序意味着没有并行性，必须是确定的
+ 除了通过输入和输出不能和外界的状态机进行通讯
+ 在这里我们将讨论相当高层次的服务
    + 例子：配置服务,比如MapReduce或者GFS master
    + 例子：键值存储，get/put操作（lab3）

#### 一个关键的问题:怎么避免脑裂？
+ 建设客户端可以连接副本A但是不能连接副本B，客户端可以只跟副本A交互吗？
+ 如果B已经崩溃，我们必须在处理的时候离开B，不然我们不能容错!
+ 如果B启动，但是网络阻止我们连接副本B，也许我们不应该在处理的时候离开B,
因为它可能存活同时在为其他客户端提供服务---存在脑裂网络分区的风险
+ 在一般情况下我们不能区分脑裂和崩溃

#### 使用单个主节点的方式可以避免脑裂吗？
+ 主服务器计算决定A和B哪个会成为主节点，因为这边只有一个主节点所以它不会出现不同意自己决定的情况。
+ 客户端和主节点交互,如果主节点失败了怎么办？这是个单节点故障问题---这种方式不是很好

#### 我们想使用复制状态机解决如下问题：
+ 不存在单节点故障(single point of failure) --可以处理任何一台机器挂掉的情况
+ 处理脑裂问题（partition w/o split brain）

#### The big insight for coping w/ partition: majority vote ？？
+ 2f+1个服务器实例，比如3,5 
+ 必须获取到不少于f+1票之后才能处理，比如：成为主节点，因此即使f个服务实例失败我们还是可以继续工作，这样就避免了单点故障问题。
+ 为什么这样做就不存在脑裂了？
    + 多数情况下，一个分区可以拥有多数实例
    注意：多数是指2f+1中的多数，不是现在存活的实例中的多数，任何两个交叉的实例（intersect servers in the intersection）在投票的时候只能投一次这是非常重要的事情，我们在后面会看到交叉（intersection）传达着其他信息。

#### 两个多数复制方案发明在1990年左右
+ 这两个方案是Paxos和 View-Stamped Replication
+ 在过去的十几年间，这些方案在现实世界中多次使用，Raft是这个想法的一个非常不错的描述。

#### MapReduce、GFS和VMware FT都从SRM中获益  
+ MapReduce的master节点并没有复制
+ GFS的master虽然被复制，但是没有自动切换到备份
+ VMware FT shared disk atomic test-and-set was (apparently) not replicated

#### Raft实现的状态机复制
+ Raft选择一个实例充当领导者
+ 客户端通过RPC请求发送Put、Get、Append命令给领导者的键值层（k/v layer）
+ 领导者将这些请求全部发送给其他的副本  
    + 每个追随者往后追加日志
    + 目标是含有相同的日志
    + 将并发的客户端请求Put(a,1) Put(a,2) 转换成顺序的请求
+ 如果多数将它添加到自己的日志，而且是持久化的，那么这个条目被“确认”，多数意味着即使少数服务器失败还是可以正常处理
+ 服务器当领导者说条目被确认之后执行一次，键值层将put操作应用到数据库，或者提取得到的结果，然后领导者返回给客户端写的结果

#### 为什么是日志？
+ 为什么服务器不是通过其他方式，比如数据库来保证状态机状态呢？
+ 如果追随者错过了一些领导者的命令会怎么样？
    + 如果有效的更新到最新？回答：重发错过的命令
+ 到目前为止，日志是一些顺序的命令,它相对于状态---开始的状态 + 日志 = 最终的状态
+ 日志经常提供一个方便的编号方案，给操作排队同时日志处于隐藏区域直到我们确认命令被提交

#### Raft的日志总是会被精确的复制吗？
+ 不：有些副本也许会滞后
+ 不：我们可以看到他们会拥有不同的条目
+ 好消息是：
    + 如果一个服务器已经执行了给定条目中的命令，那么没有其他服务器会执行这个条目中的其他命令.比如：the servers will agree on the command for each entry.State Machine Safety (Figure 3)


#### 实验2:Raft接口
+ rf.Start(command) (index, term, isleader)
    + 启动新日志条目的协议
    + 成功还是失败都会马上返回
    + 如果服务器在提交命令之前失去领导者
    + index表示要观察的日志条目
+ ApplyMsg, with Index and Command
    + 当服务（k/v server）需要执行一个新命令的时候,Raft会在通道上面产生一个消息。它同时通知客户端的rpc处理器，所以同时可以给客户端回复。

+ 注意：领导者不需要等待给the AppendEntries RPCs的回复
    + 不希望被失败的服务所阻塞
    + 所以会在各自的goroutine中发送
    + 这样（？？）意味着很多RPC请求到达的时候是无序的

#### Raft的设计主要包括两个部分：
+ 领导者选举
+ 在故障后确保相同的日志

#### Raft给领导者编号
+ 新的领导者--->新的项目(new term)
+ 一个项目至少有一个领导者；在某些情况下没有领导者（a term has at most one leader; might have no leader）
+ 编号会帮助服务器选择最新的领导，而不是取得领导

#### Raft什么时候开启领导者选举？
+ 其他服务器在一段时间内感受不到领导者的的时候
+ 他们增加本地currentTerm，成为候选人，开始选举

#### 怎么保证在一个项目（term）中只有一个领导者？
+ (Figure 2 RequestVote RPC and Rules for Servers)
+ 领导者必须获取到大多数服务器的投票
+ 每个服务器可以为每一项投一次，投票给请求的第一个服务器（within Figure 2 rules）
+ 最多一个服务器可以获得给定项目的多数票
    + 最多一个leader，即使网络分区
    + 即使一些服务器发生故障，选举也可以成功

#### 服务器如何知道选举成功了？
+ 赢家获得是多数票
+ 其他人看到AppendEntries从胜利者的心跳

#### 选举可能不成功
+ 大于3的候选人分裂投票，没有获得多数票
+ even # of live servers,两个候选人每个得到一半,少于大多数服务器可达

#### 选举失败后会发生什么？
+ 另一个超时，增量currentTerm，成为候选
+ 较高的项目优先，候选的较旧的项目退出

#### Raft如何减少分裂投票导致选举失败的机会？
+ 每个服务器在开始候选前，延迟一个随机时间
+ 延迟的作用是？
    + [diagram of times at which servers' delays expire] ???
    + 一台服务器将选择最低的随机延迟
    + 希望有足够的时间在下一个超时到期之前选择
    + 其他人将看到新领导者的AppendEntries心跳，而不会成为候选人

#### 如何选择随机延迟范围？
+ 时间太短：第二个候选在第一个结束前开始
+ 时间太长：系统在引导器故障后闲置太长时间
+ 粗略指南：
   + 假设完成一个无人选举需要10毫秒，同时我们有5台服务器
   + 我们希望延迟被20ms分开,因此随机延迟从0到100 ms

#### Raft选举遵循一个共同的模式：安全与进展的分离
+ 硬机制排除> 1领导，以避免裂脑，但可能不是领导者，或未知的结果
+ 软机制试图确保进展，总是安全地在一个新的期限开始新的选举
+ 软机制试图避免不必要的选举
    + 来自领导者的心跳（提醒服务器不开始选举）
    + 超时时间（不要开始选择太快）
    + 随机延迟（给一个领导者时间被选举）

#### 如果老领导不知道一个新的选举产生了怎么办？
+ 也许老领导接收不到新领导的信息
+ 新领导的产生意味着在大多数服务器已经增加了currentTerm
   + 所以老领导(w/ old term)不能获取到大多数AppendEntries
   + 所以老领导不会提交或执行任何新的日志条目
   + 从而没有分裂脑，尽管分裂
   + 但少数可以接受旧服务器的AppendEntries，因此日志可能在旧期限结束时分歧
        
#### 现在让我们谈谈在失败后同步日志
#### 我们想要确认什么？
+ 也许：每个服务器以相同的顺序执行相同的客户端命令，失败的服务器可能无法执行任何操作
+ 因此：if any server executes, then no server executes something
    else for that log entry
    Figure 3's State Machine Safety
+ 只要是单个领导者，这样很容易阻止日志中不一致的情况

#### 日志怎么样会不一致？
+ 日志可能会缺少---在term的结尾处缺少
    + 在发送所有AppendEntries之前，term 3的领导者崩溃
    
            S1: 3
            S2: 3 3
            S3: 3 3
    + 日志可能在同一条目中具有不同的命令！
        + 在一系列领导者崩溃后，10 11 12 13  <- log entry #
            
                S1:  3
                S2:  3  3  4
                S3:  3  3  5
                
#### 的领导者将强制其追随者使用自己的日志;比如
+ S3被选为term6的新领导者
+ S3发送新命令，entry 13, term 6  AppendEntries, previous entry 12, previous term 5
+ S2回复false(AppendEntries step 2)
+ S3 将nextIndex[S2]增加到12
+ S3 sends AppendEntries, prev entry 11, prev term 3
+ S2删除entry 12 (AppendEntries step 3)
+ S1的行为类似，但必须再回一个更远

#### 回滚的结果  
+ 每个存活的跟踪者删除不同于领导的尾部
+ 因此实时追踪者的日志是领导者日志的前缀日志
+ 存活的追随者将与领导者保持相同的日志，除非他们可能缺少最近的几个条目