# Lab 4a

## 重要结构体

### Client端结构体 ：Clerk struct

- servers []*labrpc.ClientEnd：Server端RPC端口列表
- clientId Int64Id：client的唯一标识
    - 通过nrand函数随机生成
- nextOpId int：下一个分配的操作id
    - 初始值为0，单调递增
- targetLeader int：目标服务器
- hostInfo string：client所在主机信息

### Server端结构体：ShardCtrler struct

- mu sync.Mutex：互斥锁
- me int：服务器在rf.peers中的索引
- rf *raft.Raft：服务器的Raft层
- applyCh chan raft.ApplyMsg：apply消息管道
- dead int32：服务是否停止
- configs []Config：配置列表
- clientId2executedOpId map[Int64Id]int：client id-被execute的最大操作id
- index2processedOpResultCh map[int]chan OpResult：日志条目索引-操作process结果管道
    - 各个管道长度为1

### 配置结构体：Config struct

- Num int：配置序号
    - 初始值为0
- Shards [NShards]int：分片所在RG的GID列表
    - 初始时各元素值为0，表明各分片均未加入任何有效RG
- Groups map[int][]string：GID-服务器列表

### 命令相关结构体

**操作结构体：Op struct**

- Id int：操作id
- ClientId int64Id：发起操作的client id
- Type OpType：操作类型
    - 取值：JoinRG、LeaveRG、MoveSD、QueryCF
- Servers map[int][]string：GID-服务器列表
- GIDs []int：GID列表
- Shard int：分片编号
- ConfigNum int：配置序号

注：Id、ClientId唯一确定一个Op结构体

**操作process结果结构体：OpResult struct**

- Id int：操作id
- ClientId Int64Id：发起操作的client id
- Config *Config：配置

## client端

### 4类RPC参数

**Join RPC参数**

- 输入参数
    - Servers map[int][]string：待加入的GID-服务器列表
    - ClientId Int64Id：发送者（client）id
    - OpId int：Join操作id
- 返回参数
    - Err Err：错误类型

**Leave RPC参数**

- 输入参数
    - GIDs []int：待移除的GID列表
    - ClientId Int64Id：发送者（client）id
    - OpId int：Leave操作id
- 返回参数
    - Err Err：错误类型

**Move RPC参数**

- 输入参数
    - Shard int：待移动的分片编号
    - GID int：目标GID
    - ClientId Int64Id：发送者（client）id
    - OpId int：Move操作id
- 返回参数
    - Err Err：错误类型

**Query RPC参数**

- 输入参数
    - Num int：待查询的配置序号
    - ClientId Int64Id：发送者（client）id
    - OpId int：Query操作id
- 返回参数
    - Err Err：错误类型
    - Config Config：查询到的配置

### 4类操作

**Join操作：ck.Join**

- 参数：**●**servers map[int][]string：待加入的GID-服务器列表
- 无返回值
- 流程
    1. 生成Join RPC的输入参数（根据ck.nextOpId）
    2. 准备好Join RPC的空返回参数
    3. 向ck.targetLeader发送Join RPC请求
    4. 后置处理
        1. 若未收到ck.targetLeader的ACK，更换ck.targetLeader，回到步骤2
        2. 若收到ck.targetLeader的ACK且reply.Err == OK，不进行后置处理
        3. 若收到ck.targetLeader的ACK且reply.Err == ErrWrongLeader，更换ck.targetLeader，回到步骤2
        4. 若收到ck.targetLeader的ACK且reply.Err == ErrOvertime，更换ck.targetLeader，回到步骤2
    5. ck.nextOpId++

**Leave操作：ck.Leave**

- 参数：**●**gids []int：待移除的GID列表
- 无返回值
- 流程
    1. 生成Leave RPC的输入参数（根据ck.nextOpId）
    2. 准备好Leave RPC的空返回参数
    3. 向ck.targetLeader发送Leave RPC请求
    4. 后置处理
        1. 若未收到ck.targetLeader的ACK，更换ck.targetLeader，回到步骤2
        2. 若收到ck.targetLeader的ACK且reply.Err == OK，不进行后置处理
        3. 若收到ck.targetLeader的ACK且reply.Err == ErrWrongLeader，更换ck.targetLeader，回到步骤2
        4. 若收到ck.targetLeader的ACK且reply.Err == ErrOvertime，更换ck.targetLeader，回到步骤2
    5. ck.nextOpId++

**Move操作：ck.Move**

- 参数：**●**shard int：待移动的分片编号 **●**gid int：目标GID
- 无返回值
- 流程
    1. 生成Move RPC的输入参数（根据ck.nextOpId）
    2. 准备好Move RPC的空返回参数
    3. 向ck.targetLeader发送Move RPC请求
    4. 后置处理
        1. 若未收到ck.targetLeader的ACK，更换ck.targetLeader，回到步骤2
        2. 若收到ck.targetLeader的ACK且reply.Err == OK，不进行后置处理
        3. 若收到ck.targetLeader的ACK且reply.Err == ErrWrongLeader，更换ck.targetLeader，回到步骤2
        4. 若收到ck.targetLeader的ACK且reply.Err == ErrOvertime，更换ck.targetLeader，回到步骤2
    5. ck.nextOpId++

**Query操作：ck.Query**

- 参数：**●**num int：待查询的配置序号
- 返回值：**●**Config：查询到的配置
- 流程
    1. 生成Query RPC的输入参数（根据ck.nextOpId）
    2. 准备好Query RPC的空返回参数
    3. 向ck.targetLeader发送Query RPC请求
    4. 后置处理
        1. 若未收到ck.targetLeader的ACK，更换ck.targetLeader，回到步骤2
        2. 若收到ck.targetLeader的ACK且reply.Err == OK，将返回值置为reply.Config
        3. 若收到ck.targetLeader的ACK且reply.Err == ck.targetLeader，更换targetLeader，回到步骤2
        4. 若收到ck.targetLeader的ACK且reply.Err == ErrOvertime，更换ck.targetLeader，回到步骤2
    5. ck.nextOpId++，返回所得的配置

## server端

### Note

- server端服务可分为Service层、Raft层，两层之间通过applyCh管道交互
- 注意区别以下名词：
    - **commit**（Raft层）：操作被commit后可在rf.applyTicker中被apply
    - **apply**（Raft层）：操作被apply后其对应消息被插入applyCh管道
    - **process**（Service层）
        - process操作：决定是否要execute操作
    - **execute**（Service层）：执行SMR中的操作

### 2个RPC Handler子方法

**sc.prepareForProcess**

- 参数：**●**op Op：操作
- 返回值：**●**Err：错误类型 **●**int：日志条目索引
- 执行过程
    1. 执行sc.rf.Start，返回index, _, isLeader
    2. 若isLeader == true，返回OK, index；若isLeader == false，返回ErrWrongLeader, -1

**sc.waitForProcess**

- 参数：**●**op Op：操作 **●**index int：日志条目索引
- 返回值：**●**Err：错误类型 **●***Config：配置
- 执行过程
    1. 根据index得到对应的操作process结果管道ch
    2. 开启计时器
    - 若首先等待到ch管道中有被process的操作processedOp被取出
        1. 若检测到processedOpResult与op不一致（processedOpResult.ClientId ≠ op.ClientId或processedOpResult.Id≠op.Id），将返回的错误类型定为ErrWrongLeader；反之，将返回的错误类型定为OK，将返回的配置置为processedOpResult.Config
    - 若首先等待到计时器过时
        1. 将返回的错误类型定为ErrOvertime
    1. 停止计时器
    2. 删除ch管道

### 4类RPC Handler

**Join RPC Handler (sc.Join)**

- 参数：**●**args *****JoinArgs：输入参数 **●**reply *****JoinReply：返回参数
- 无返回值
- 执行过程
    1. 准备好需要process的Op
    2. 调用sc.prepareForProcess准备process操作
    3. 若sc.prepareForProcess返回的Err值prepareErr ≠ OK，令reply.Err = prepareErr，直接返回
    4. 调用sc.waitForProcess等待操作被process，令reply.Err等于sc.waitForProcess返回的错误类型

**Leave RPC Handler (sc.Leave)**

- 参数：**●**args *****LeaveArgs：输入参数 **●**reply *****LeaveReply：返回参数
- 无返回值
- 执行过程
    1. 准备好需要process的Op
    2. 调用sc.prepareForProcess准备process操作
    3. 若sc.prepareForProcess返回的Err值prepareErr ≠ OK，令reply.Err = prepareErr，直接返回
    4. 调用sc.waitForProcess等待操作被process，令reply.Err等于sc.waitForProcess返回的错误类型

**Move RPC Handler (sc.Move)**

- 参数：**●**args *****MoveArgs：输入参数 **●**reply *****MoveReply：返回参数
- 无返回值
- 执行过程
    1. 准备好需要process的Op
    2. 调用sc.prepareForProcess准备process操作
    3. 若sc.prepareForProcess返回的Err值prepareErr ≠ OK，令reply.Err = prepareErr，直接返回
    4. 调用sc.waitForProcess等待操作被process，令reply.Err等于sc.waitForProcess返回的错误类型

**Query RPC Handler (sc.Query)**

- 参数：**●**args *****QueryArgs：输入参数 **●**reply *****QueryReply：返回参数
- 无返回值
- 执行过程
    1. 准备好需要process的Op
    2. 调用sc.prepareForProcess准备process操作
    3. 若sc.prepareForProcess返回的Err值prepareErr ≠ OK，令reply.Err = prepareErr，直接返回
    4. 调用sc.waitForProcess等待操作被process，令reply.Err及reply.Config上分别等于sc.waitForProcess返回的错误类型及配置

### 3类新配置获得方法

**sc.getNewConfigAfterJoinOp**

- 参数：**●**servers map[int][]string：待加入的GID-服务器列表
- 返回值：**●***Config：新配置
- 执行过程
    1. 得到sc.configs中最后一个配置的副本ret，ret++
    2. 更新ret.Groups（根据输入参数servers）
    3. 得到gid2shards（GID-分片编号列表）及leftShards（未加入任何有效RG的分片编号列表），调用ret.Rebalance进行负载均衡
    4. 返回ret

**sc.getNewConfigAfterLeaveOp**

- 参数：**●**gids []int：待移除的GID列表
- 返回值：**●***Config：新配置
- 执行过程
    1. 得到sc.configs中最后一个配置的副本ret，ret.Num++
    2. 更新ret.Groups（根据输入参数gids），同时得到gid2shards（GID-分片编号列表）及leftShards（未加入任何有效RG的分片编号列表），调用ret.Rebalance进行负载均衡
    3. 返回ret

**sc.getNewConfigAfterMoveOp**

- 参数：**●**shard int：待移动的分片编号 **●**gid int：目标GID
- 返回值：**●***Config：新配置
- 执行过程
    1. 得到sc.configs中最后一个配置的副本ret，ret.Num++
    2. 更新ret.Shards（根据输入参数shard及gid）
    3. 返回ret

### 负载均衡方法：config.Rebalance

- 参数：**●**gid2Shards map[int][]int：GID-分片编号列表 **●**leftShards []int：未加入任何有效RG的分片编号列表
- 返回值：**●***Config：新配置
- 执行过程
    - 若RG数量等于0
        1. 令config.Shards中各元素等于0（表明各分片均未加入任何有效RG），直接返回
    - 若RG数量大于0
        1. 根据gid2Shards得到RG信息（GID, RG的分片数量）列表rgInfos，将rgInfos根据RG包含的分片数量从小到大排序
        2. 得到与rgInfos相对应的RG目标分片数量列表（均分rgNum，保证非严格单调递增）
        3. 若RG的分片数量大于目标分片数量，将RG中多出的分片编号附加至leftShards（调整leftShards）
        4. 若RG的分片数量小于目标分片数量，将leftShards中的分片移动到该RG中（更新config.Shards）

### 总指挥：kv.processor

- 功能：server端运行的协程，控制快照及操作被process的过程
- 循环过程
    1. 从applyCh管道中取出操作消息
    2. 若操作类型为QueryCF，或操作类型为JoinRG / LeaveRG / MoveSD且根据sc.clientId2executedOpId可知操作之前未被execute过，execute操作并更新sc.clientId2executedOpId
        1. 若操作类型为QueryCF，execute操作时，根据op.ConfigNum从sc.configs中获得op.Config
        2. 若操作类型为JoinRG / LeaveRG / MoveSD，execute操作时，先调用相应的新配置获得方法获得新配置newConfig，再将newConfig附加至sc.configs末尾
    3. 将操作process结果插入其日志条目索引所对应的操作process结果管道中（表示操作已被process）

## 注意点

- 注意map / slice的引用问题
    - 获得配置config的副本configCopy时，可将config.Shards直接赋值给configCopy.Shards（array非引用）
    - 获得配置config的副本configCopy时，应通过make为configCopy.Groups创建一个新的map，而非仅将config.Groups直接赋值给configCopy.Groups（map为引用）