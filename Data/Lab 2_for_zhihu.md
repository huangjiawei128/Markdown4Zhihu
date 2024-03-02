# Lab 2

## Raft struct

<img src="https://https://github.com/huangjiawei128/Markdown4Zhihu/blob/master/OriData/6.824Lab2/Untitled.png" alt="Untitled" style="zoom:50%;" />

<img src="Untitled%201.png" alt="Untitled" style="zoom:50%;" />

<img src="Untitled%202.png" alt="Untitled" style="zoom:50%;" />

- 基本成员变量
    - mu sync.Mutex：互斥锁
    - peers []*labrpc.ClientEnd：所有服务器的RPC端口
    - persister *Persister：持久化器
    - me int：服务器在peers中的索引
    - dead int32：Raft层服务是否停止
    - gid int：服务器所在GID
- 所有服务器持久化状态成员变量
    - currentTerm int：当前term
        - 初始值为0，单调递增
    - votedFor int：当前term内投票支持的服务器
        - 未投票时为-1
    - nextRpcId int：下一个分配的RPC id
        - 初始值为0，单调递增
    - log []LogEntry：日志（日志条目列表）
        - LogEntry struct：日志条目
            - Command interface{}：命令
            - Term int：日志条目产生时的term
        - 索引从1开始
    - lastIncludedIndex int：快照所包含的最后一个日志条目索引
        - 保证不大于最后一个日志条目索引
    - lastIncludedTerm int：快照所包含的最后一个日志条目对应的term
- 所有服务器非持久化状态成员变量
    - commitIndex int：被commit的最高日志条目索引
        - 初始值为0，单调递增
        - 保证不大于最后一个日志条目索引
    - lastApplied int：被apply至状态机的最高日志条目索引
        - 初始值为0，单调递增
        - 保证不大于commitIndex
    - initialTime time.Time：当前计时周期开始时间
    - role Role：服务器角色
    - voteNum int：当前term内收到的票数
- leader服务器非持久化状态成员变量
    - nextIndex []int：向各个服务器发送的下一个日志条目索引
        - 初始值为leader服务器最后一个日志条目索引+1
    - matchIndex []int：各个服务器匹配的最高日志条目索引
        - 初始值为0，单调递增
        - 保证matchIndex[i] < nextIndex[i]
- 管道及其控制变量
    - applyCh chan ApplyMsg：apply消息管道
        - 控制变量
            - nextApplyOrder int：下一个插入applyCh的消息（队列）顺序编号
            - finishedApplyOrder int：已知插入applyCh的消息（队列）最大顺序编号
            - applyCond *sync.Cond：applyCh消息（队列）插入控制条件变量
        - 无长度
    - newCommand chan bool：新命令标志管道
        - 长度为1

## 2个Service层交互接口

### rf.Start

- 功能：添加命令
- 参数：**●**command interface{}：命令
- 返回值：**●**int：命令所对应的日志条目索引 **●**int：命令添加时的term **●**bool：服务器是否为leader
- 流程
    - 直接返回-1, -1, false的情况
        - Raft服务停止
        - 服务器非leader
    1. 向日志中添加命令对应的日志条目（**状态变量持久化**）
    2. 返回日志条目索引, rf.currentTerm, true
    3. 若rf.newCommand管道中无数据，插入true作为新命令标志（在rf.mu临界区外进行）

### rf.Snapshot

- 功能：截取快照
- 参数：**●**index int：快照所包含的最后一个日志条目索引 **●**snapshot []byte：快照数据
- 无返回值
- 流程
    - 直接返回的情况
        - Raft服务停止
        - index不大于rf.lastIncludedIndex
    1. 截断日志中日志条目索引不大于index的部分
    2. 更新rf.lastIncludedIndex及rf.lastIncludedTerm（**状态变量及快照持久化**）

## 3个角色转变

<img src="Untitled%203.png" alt="Untitled" style="zoom:50%;" />

- 各个角色转变方法本身均不开启新的计时周期
- 若更新到所有服务器持久化状态成员变量，需进行**状态变量持久化**

### rf.BecomeFollower

- 参数：**●**term int：所发现的term
- 无返回值
- 更新变量
    - rf.currentTerm：term（若term > Raft.currentTerm）
    - rf.votedFor：-1（若term > Raft.currentTerm）
    - rf.role：Follower
    - rf.voteNum：0

### rf.BecomeCandidate

- 无参数
- 无返回值
- 更新变量
    - rf.currentTerm：++
    - rf.votedFor：rf.me
    - rf.role：Candidate
    - rf.voteNum：1

### rf.BecomeLeader

- 无参数
- 无返回值
- 更新变量
    - rf.role：Leader
    - rf.nextIndex[i]：最后一个日志条目索引+1
    - rf.matchIndex[i]：0

## 3个Ticker

- 各个Ticker在Raft服务未停止时不断循环执行相应过程

### rf.electTicker

- 功能：控制leader选举的过程
- 循环过程
    1. 休息随机长度（[MinTimeout, MaxTimeout]）的时段
    - 直接重新开始循环过程的情况
        - 服务器为leader
        - 在休息时段内有新的计时周期开启
    1. **转变为candidate**
    2. **开启新的计时周期**
    3. 向其余服务器异步发送**RequestVote RPC**（含前置/后置处理）

### rf.appendTikcer

- 功能：控制日志条目附加 / 心跳发送的过程
- 循环过程
    1. 休息固定长度（AppendPeriod）的时段（若rf.newCommand管道中有新命令标志，提前结束休息）
    2. 向其余服务器异步发送InstallSnapshot RPC或AppendEntries RPC（含前置/后置处理）
        1. 若rf.nextIndex[server] ≤ rf.lastIncludedIndex，发送**InstallSnapshot RPC**
        2. 若rf.nextIndex[server] > rf.lastIncludedIndex，发送**AppendEntries RPC**

### rf.applyTicker

- 功能：控制日志条目被apply至状态机的过程
- 循环过程
    1. 休息固定长度（ApplyPeriod）的时段
    2. 若rf.lastApplied ＜ rf.lastIncludedIndex，向apply消息列表中添加快照消息，并更新rf.lastApplied及rf.commitIndex
    3. 依次向apply消息列表中添加尚未apply的日志条目所对应的命令消息，保证最终rf.lastApplied == rf.commitIndex
    4. 若apply消息列表非空，将apply消息列表插入applyCh管道
        1. 插入前：取出消息队列顺序编号，若该顺序编号不等于rf.finishedApplyOrder+1，在条件变量rf.applyCond上陷入等待
        2. 插入中：依次将apply消息列表中的消息按顺序插入applyCh管道（在rf.mu临界区外进行）
        3. 插入后：将rf.finishedApplyOrder更新为当前消息队列顺序编号，唤醒在条件变量rf.applyCond上等待的协程

## 3个RPC

- 各个RPC需关注的部分
    - 输入参数
    - 返回参数
    - 发送者前置处理：生成输入参数，准备好空返回参数
        - 参数：**●**server int：接收者
        - 返回值：**●***XXXArgs：生成的输入参数 **●***XXXReply：准备好的空返回参数 **●**ok：告知发送者是否可向接收者发送RPC
    - 接收者Handler：由输入参数生成完整的返回参数
        - 参数：**●**args *****XXXArgs：输入参数 **●**reply *****XXXReply：需要生成的返回参数
        - 无返回值
    - 发送者后置处理：根据输入参数和完整的返回参数进行
        - 参数：**●**server int：接收者 **●**args *****XXXArgs：输入参数 **●**reply *****XXXReply：完整的返回参数 **●**ok bool：发送者是否收到接收者的ACK
        - 无返回值

### RequestVote RPC

<img src="Untitled%204.png" alt="Untitled" style="zoom:50%;" />

- 输入参数：RequestVoteArgs struct
    - RpcId int：RPC id
    - Term int：发送者发送时的term
    - CandidateId int：发送者（candidate）id
    - LastLogIndex int：最后一个日志条目索引
    - LastLogTerm int：最后一个日志条目对应的term
- 返回参数：RequestVoteReply struct
    - Term int：接收者接受时的term
    - voteGranted bool：接收者是否投票支持
- 发送者前置处理：rf.sendRequestVotePre
    - 直接返回&RequestVoteArgs{}, &RequestVoteReply{}, false的情况
        - 发送者非candidate
    1. 生成输入参数
    2. 返回生成的输入参数, 空返回参数, true
- 接收者Handler：rf.RequestVote
    1. 若args.Term > rf.currentTerm，**转变为follower**，其term更新为args.Term
    - 直接令reply.Term = rf.currentTerm, reply.VoteGranted = false并返回的情况
        - args.Term < rf.currentTerm（发送者的term落后于接收者）
        - rf.votedFor ≠ -1且rf.votedFor ≠ args.CandidateId（接收者已投票支持非发送者服务器）
        - 发送者的日志落后于接收者
            - 发送者的最后一个日志条目对应的term小于接收者
            - 发送者的最后一个日志条目对应的term等于接收者，但其索引小于接收者
    1. 令reply.Term = rf.currentTerm, reply.VoteGranted = true，令rf.votedFor = args.CandidateId以投票支持发送者（**状态变量持久化**）
    2. **开启新的计时周期**
- 发送者后置处理：rf.sendRequestVotePro
    - 直接返回的情况
        - 发送者非candidate（发送者的角色前后不一致）
        - rf.currentTerm ≠ args.Term（发送者的term前后不一致）
        - reply.Term > rf.currentTerm（发送者的term落后于接收者）
            - 返回前需**转变为follower**，其term更新为reply.Term
    1. 若reply.VoteGranted == true，rf.voteNum++，当rf.voteNum大于服务器数量的一半时，**转变为leader**

### AppendEntries RPC

<img src="Untitled%205.png" alt="Untitled" style="zoom:50%;" />

- 输入参数：AppendEntriesArgs struct
    - RpcId int：RPC id
    - Term int：发送者发送时的term
    - LeaderId int：发送者（leader）id
    - PrevLogIndex int：需附加的日志条目列表前一日志条目索引
    - PrevLogTerm int：需附加的日志条目列表前一日志条目对应的term
    - Entries []LogEntry：需附加的日志条目列表
    - LeaderCommit int：发送者发送时的commitIndex
- 返回参数：AppendEntriesReply struct
    - Term int：接收者接收时的term
    - Status AppendStatus：日志条目附加完成状态
        - Success：成功
        - TermLag：发送者的term落后于接收者
        - EntrySnapshot：发送者args.PrevLogIndex+1处日志条目已被截断至接收者的快照中
        - EntryMissmatch：发送者与接收者args.PrevLogIndex处日志条目不匹配
    - NextIndex int：发送者应更新的rf.nextIndex值
- 发送者前置处理：rf.sendAppendEntriesPre
    - 直接返回&AppendEntriesArgs{}, &AppendEntriesReply{}, false的情况
        - 发送者非leader
    1. 生成输入参数（根据rf.nextIndex）
    2. 返回生成的输入参数, 空返回参数, true
- 接收者Handler：rf.AppendEntries
    1. 若args.Term ≥ rf.currentTerm，**转变为follower**，其term更新为args.Term，并**开启新的计时周期**
    - 直接令reply.Term = rf.currentTerm, reply.Status ≠ Success并返回的情况
        - args.Term < rf.currentTerm（发送者的term落后于接收者）
            - 令reply.Status = TermLag，不关心reply.NextIndex
        - args.PrevLogIndex+1 ≤ rf.lastIncludedIndex（发送者args.PrevLogIndex+1处日志条目已被截断至接收者的快照中）
            - 令reply.Status = EntrySnapshot, reply.NextIndex = rf.lastIncludedIndex+1
        - 发送者与接收者args.PrevLogIndex处日志条目不匹配
            - 令reply.Status = EntryMismatch，按如下方式确定reply.NextIndex：
                - 若接收者最后一个日志条目索引lastLogIndex小于args.PrevLogIndex，令reply.NextIndex = lastLogIndex
                - 若接收者最后一个日志条目索引lastLogIndex不小于args.PrevLogIndex
                    - 若args.PrevLogIndex > rf.lastIncludedIndex，令reply.NextIndex等于args.PrevLogIndex处日志条目的term所对应的第一个日志条目索引
                    - 若args.PrevLogIndex == rf.lastIncludedIndex，令reply.NextIndex = rf.lastIncludedIndex+1
            - 返回前需截断rf.log中日志条目索引不小于args.PrevLogIndex的部分（**状态变量持久化**）
    1. 令reply.Term = rf.currentTerm, replyStatus = Success, reply.NextIndex = args.PrevLogIndex+1+len(args.Entries)
    2. 按如下方式确定rf.log，以保证rf.log中日志条目的有序性（**状态变量持久化**）：
        1. 若reply.NextIndex > lastLogIndex，或args.Entries非空且args.Entries中最后一个日志条目对应的term大于reply.NextIndex对应的term，从原rf.log的args.PrevLogIndex+1处开始附加args.Entries，其后若还有原rf.log的日志条目则截断
        2. 若reply.NextIndex ≤ lastLogIndex，且args.Entries为空或args.Entries中最后一个日志条目对应的term不大于reply.NextIndex对应的term，从原rf.log的args.PrevLogIndex+1处开始附加args.Entries，保留其后原rf.log的日志条目
    3. 若args.LeaderCommit > rf.commitIndex，令rf.commitIndex = Min(args.LeaderCommit, lastLogIndex)
- 发送者后置处理：rf.sendAppendEntriesPro
    - 直接返回的情况
        - 发送者非leader（发送者的角色前后不一致）
        - rf.currentTerm ≠ args.Term（发送者的term前后不一致）
        - reply.Status == TermLag（发送者的term落后于接收者，必有reply.Term > rf.currentTerm）
            - 返回前需**转变为follower**，其term更新为reply.Term
        - reply.Status == EntrySnapshot或reply.Status == EntryMismatch
            - 若reply.NextIndex ＞ rf.matchIndex[server]，返回前需令rf.nextIndex[server] = reply.NextIndex
        - reply.Status == Success且reply.NextIndex-1 ≤ rf.matchIndex[server]
    1. 令rf.matchIndex[server] = reply.NextIndex-1, rf.nextIndex[server] = reply.NextIndex
    2. 根据rf.matchIndex计算大部分其他服务器均匹配的日志条目索引及对应的term，若该索引值大于rf.commitIndex且对应的term等于rf.currentTerm，将rf.commitIndex更新为该索引值（注意Figure 8的场景）
       
        <img src="Untitled%206.png" alt="Untitled" style="zoom:50%;" />

### InstallSnapshot RPC

<img src="Untitled%207.png" alt="Untitled" style="zoom:50%;" />

- 输入参数：InstallSnapshotArgs struct
    - RpcId int：RPC id
    - Term int：发送者发送时的term
    - LeaderId int：发送者（leader）id
    - LastIncludedIndex：发送者发送时快照所包含的最后一个日志条目索引
    - LastIncludedTerm：发送者发送时快照所包含的最后一个日志条目对应的term
    - Data []byte：快照数据
- 返回参数：InstallSnapshotReply struct
    - Term int：接收者接收时的term
    - LastIncludedIndex int：接收者处理完请求后快照所包含的最后一个日志条目索引
- 发送者前置处理：rf.sendInstallSnapshotPre
    - 直接返回&InstallSnapshotArgs{}, &InstallSnapshotReply{}, false的情况
        - 发送者非leader
    1. 生成输入参数
    2. 返回生成的输入参数, 空返回参数, true
- 接收者Handler：rf.InstallSnapshot
    1. 若args.Term ≥ rf.currentTerm，**转变为follower**，其term更新为args.Term，并**开启新的计时周期**
    - 直接令reply.Term = rf.currentTerm, reply.LastIncludedIndex = rf.lastIncludedIndex并返回的情况
        - args.Term < rf.currentTerm（发送者的term落后于接收者）
        - args.LastIncludedIndex ≤ rf.lastIncludedIndex（发送者的lastIncludedIndex不大于接收者）
    1. 更新rf.lastIncludedIndex、rf.lastIncludedTerm及rf.log（**状态变量及快照持久化**），令reply.LastIncludedIndex = rf.lastIncludedIndex（更新后的值）
    2. 若rf.lastApplied < rf.lastIncludedIndex，更新rf.lastApplied及rf.commitIndex，将快照消息插入applyCh管道
        1. 插入前：取出快照消息顺序编号，若该顺序编号不等于rf.finishedApplyOrder+1，在条件变量rf.applyCond上陷入等待
        2. 插入中：将快照消息插入applyCh管道（在rf.mu临界区外进行）
        3. 插入后：将rf.finishedApplyOrder更新为快照消息顺序编号，唤醒在条件变量rf.applyCond上等待的协程
- 发送者后置处理：rf.sendInstallSnapshotPro
    - 直接返回的情况
        - 发送者非leader（发送者的角色前后不一致）
        - rf.currentTerm ≠ args.Term（发送者的term前后不一致）
        - reply.Term > rf.currentTerm（发送者的term落后于接收者）
            - 返回前需**转变为follower**，其term更新为reply.Term
    1. 令rf.matchIndex[server] = reply.LastIncludedIndex, rf.nextIndex[server] = reply.LastIncludedIndex+1

## 注意点

- 注意消息apply至状态机时的顺序控制
    - 利用nextApplyOrder、finishedApplyOrder、applyCond三个变量，保证rf.InstallSnapshot中apply至状态机的快照消息不会插入至rf.applyTicker中apply至状态机的消息列表
- 注意新命令标志管道newCommand的使用
    - 令rf.appendTikcer提前开始附加日志条目，以保证Lab3中客户端操作完成得足够快（否则无法通过TestSpeed3A / TestSpeed3B）
- 注意确保Raft的各属性及Raft struct中的各个成员变量应满足的不变式始终成立，需考虑异步网络延迟、中断等各种异常情况
    - 各个RPC的接收者Handler、发送者后置处理中直接返回的情况应罗列全面，尤其注意AppendEntries RPC的接收者Handler中确定rf.log的方式，防止已commit的日志条目被截断