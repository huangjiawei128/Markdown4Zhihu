# Lab 3

## 重要结构体

### Client端结构体 ：Clerk struct

- servers []*labrpc.ClientEnd：Server端RPC端口列表
- clientId Int64Id：client的唯一标识
    - 通过nrand函数随机生成
- nextOpId int：下一个分配的操作id
    - 初始值为0，单调递增
- targetLeader int：目标服务器

### Server端结构体：KVServer struct

- mu sync.Mutex：互斥锁
- me int：服务器在rf.peers中的索引
- rf *raft.Raft：服务器的Raft层
- applyCh chan raft.ApplyMsg：apply消息管道
- maxraftstate int：Raft层状态信息长度阈值
    - 若＜0则表示Raft层状态信息无长度限制
- dead int32：服务是否停止
- clientId2executedOpId map[Int64Id]int：client id-被execute的最大操作id
- index2processedOpResultCh map[int]chan OpResult：日志条目索引-操作process结果管道
    - 各个管道长度为1
- kvStore KVStore：用于存储K-V对

### 存储结构体：KVStore struct

- KVMap map[string]string：key-value

### 命令相关结构体

**操作结构体：Op struct**

- Id int：操作id
- ClientId int64Id：发起操作的client id
- Type OpType：操作类型
    - 取值：GetV、PutKV、AppendKV
- Key string：key值
- Value string：value值

注：Id、ClientId唯一确定一个Op结构体

**操作process结果结构体：OpResult struct**

- Id int：操作id
- ClientId int64Id：发起操作的client id
- Value string：value值

## client端

### 2类RPC参数

**Get RPC参数**

- 输入参数
    - Key string：key值
    - ClientId Int64Id：发送者（client）id
    - OpId int：Get操作id
- 返回参数
    - Err Err：错误类型
    - Value string：value值

**PutAppend RPC参数**

- 输入参数
    - Key string：key值
    - Value string：value值
    - Op OpType：操作类型
    - ClientId Int64Id：发送者（client）id
    - OpId int：Put/Append操作id
- 返回参数
    - Err Err：错误类型

### 2类操作

**Get操作：ck.Get**

- 参数：**●**key string：key值
- 返回值：**●**string：key值对应的value值
- 流程
    1. 生成Get RPC的输入参数（根据ck.nextOpId）
    2. 准备好Get RPC的空返回参数
    3. 向ck.targetLeader发送Get RPC请求
    4. 后置处理
        1. 若未收到ck.targetLeader的ACK，更换ck.targetLeader，回到步骤2
        2. 若收到ck.targetLeader的ACK且reply.Err == OK，将返回值置为reply.Value
        3. 若收到ck.targetLeader的ACK且reply.Err == ErrWrongLeader，更换ck.targetLeader，回到步骤2
        4. 若收到ck.targetLeader的ACK且reply.Err == ErrOvertime，更换ck.targetLeader，回到步骤2
    5. ck.nextOpId++，返回所得的value值

**Put & Append操作：ck.PutAppend**

- 参数：**●**key string：key值 **●**value string：value值 **●**op string：操作类型
- 无返回值
- 流程
    1. 生成PutAppend RPC的输入参数（根据ck.nextOpId及ck.op）
    2. 准备好PutAppend RPC的空返回参数
    3. 向ck.targetLeader发送PutAppend RPC请求
    4. 后置处理
        1. 若未收到ck.targetLeader的ACK，更换ck.targetLeader，回到步骤2
        2. 若收到ck.targetLeader的ACK且reply.Err == OK，不进行后置处理
        3. 若收到ck.targetLeader的ACK且reply.Err == ErrWrongLeader，更换ck.targetLeader，回到步骤2
        4. 若收到ck.targetLeader的ACK且reply.Err == ErrOvertime，更换ck.targetLeader，回到步骤2
    5. ck.nextOpId++

## server端

### Note

- server端服务可分为Service层、Raft层，两层之间通过applyCh管道交互
- 注意区别以下名词：
    - **commit**（Raft层）：操作被commit后可在rf.applyTicker中被apply
    - **apply**（Raft层）：快照或操作被apply后其对应消息被插入applyCh管道
    - **process**（Service层）
        - process快照：根据快照数据恢复kv.kvStore及kv.clientId2executedOpId
        - process操作：决定是否要execute操作
    - **execute**（Service层）：执行SMR中的操作

### 2个RPC Handler子方法

**kv.prepareForProcess**

- 参数：**●**op Op：操作
- 返回值：**●**Err：错误类型 **●**int：日志条目索引
- 执行过程
    1. 执行kv.rf.Start，返回index, _, isLeader
    2. 若isLeader == true，返回OK, index；若isLeader == false，返回ErrWrongLeader, -1

**kv.waitForProcess**

- 参数：**●**op Op：操作 **●**index int：日志条目索引
- 返回值：**●**Err：错误类型 **●**string：value值
- 执行过程
    1. 根据index得到对应的操作process结果管道ch
    2. 开启计时器
    - 若首先等待到ch管道中有操作process结果processedOpResult被取出
        1. 若检测到processedOpResult与op不一致（processedOpResult.ClientId ≠ op.ClientId或processedOpResult.Id ≠ op.Id），将返回的错误类型定为ErrWrongLeader；反之，将返回的错误类型定为OK，将返回的value值置为processedOpResult.Value
    - 若首先等待到计时器过时
        1. 将返回的错误类型定为ErrOvertime
    1. 停止计时器
    2. 删除ch管道

### 2类RPC Handler

**Get RPC Handler (kv.Get)**

- 参数：**●**args *****GetArgs：输入参数 **●**reply *****GetReply：返回参数
- 无返回值
- 执行过程
    1. 准备好需要process的Op
    2. 调用kv.prepareForProcess准备process操作
    3. 若kv.prepareForProcess返回的Err值prepareErr ≠ OK，令reply.Err = prepareErr，直接返回
    4. 调用kv.waitForProcess等待操作被process，令reply.Err及reply.Value分别等于kv.waitForProcess返回的错误类型及value值

**Put & Append RPC Handler (kv.PutAppend)**

- 参数：**●**args *****PutAppendArgs：输入参数 **●**reply *****PutAppendReply：返回参数
- 无返回值
- 执行过程
    1. 准备好需要process的Op
    2. 调用kv.prepareForProcess准备process操作
    3. 若kv.prepareForProcess返回的Err值prepareErr ≠ OK，令reply.Err = prepareErr，直接返回
    4. 调用kv.waitForProcess等待操作被process，令reply.Err等于kv.waitForProcess返回的错误类型

### 总指挥：kv.processor

- 功能：server端运行的协程，控制快照及操作被process的过程
- 循环过程
    1. 从applyCh管道中取出消息
    - 若取出的消息为快照消息
        1. 根据快照数据恢复kv.clientId2executedOpId及kv.kvStore
    - 若取出的消息为操作消息
        1. 若操作类型为GetV，或操作类型为PutKV / AppendKV且根据kv.clientId2executedOpId可知操作之前未被execute过，execute操作并更新kv.clientId2executedOpId
            1. 若操作类型为GetV，execute操作时，根据op.Key从kv.kvStore中获得opResult.Value
            2. 若操作类型为PutKV / AppendKV，execute操作时，根据op.Key-op.Value更新kv.kvStore
        2. 将操作process结果插入其日志条目索引所对应的操作process结果管道中（表示操作已被process）
        3. 若kv.maxraftstate > 0且当前Raft层状态信息长度大于kv.maxraftstate，调用kv.rf.Snapshot截取快照

## 注意点

- 注意kv.waitForProcess中计时器的设置
    - 考虑Figure8中的场景（(a)→(b)：S1、S2与S3、S4、S5分区，S5变成term 3的candidate，分区消失，S5被选举为term 3的leader，新日志条目加入S5；(b)→(c)：S5 crash，S1被选举为term 4的leader），防止S1在term 2中添加日志条目后由于无过时机制而导致其在term 4中无法再次添加同一client操作对应的日志条目，造成无限等待状态（term 2中添加的日志条目始终无法在term 4中被主动commit，除非有term 4中添加的日志条目被commit）
    - 考虑新添加的日志条目由于网络分区一直无法提交的情况
- 注意操作process结果管道的创建及删除问题
    - 仅在kv.waitForProcess中创建及删除（保证有“创建”必有“删除”），而不在kv.NotifyProcessedOpResultCh中创建，防止产生多余的操作process结果管道（但不影响正确性）