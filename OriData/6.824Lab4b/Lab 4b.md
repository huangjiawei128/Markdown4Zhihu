# Lab 4b

## 重要结构体

### Client端结构体 ：Clerk struct

- sm *shardctrler.Clerk：访问配置服务器的客户端
- config shardctrler.Config：当前配置
    - config.Num单调递增
- make_end func(string) *labrpc.ClientEnd：将服务器名映射至RPC端口的函数
- clientId Int64Id：client的唯一标识
    - 通过nrand函数随机生成
- nextOpId int：下一个分配的操作id
    - 初始值为0，单调递增
- gid2targetLeader map[int]int：GID-目标服务器

### Server端结构体：ShardKV struct

**基本变量**

- mu sync.Mutex：互斥锁
- me int：服务器在rf.peers中的索引
- rf *raft.Raft：服务器的Raft层
- applyCh chan raft.ApplyMsg：apply消息管道
- make_end func(string) *labrpc.ClientEnd：将服务器名映射至RPC端口的函数
- gid int：服务器所在RG的GID
- ctrlers []*labrpc.ClientEnd：配置服务器RPC端口列表
- maxraftstate int：Raft层状态信息长度阈值
    - 若＜0则表示Raft层状态信息无长度限制
- dead int32：服务是否停止
- mck *shardctrler.Clerk：访问配置服务器的客户端
- clientId2executedOpId map[Int64Id]int：client id-被execute的最大操作id
- index2processedOpResultCh map[int]chan OpResult：日志条目索引-命令process结果管道
    - 各个管道长度为1
- kvStore KVStore：用于存储K-V对
- gid2targetLeader sync.Map：GID-目标服务器

**配置相关变量**

- curConfig shardctrler.Config：当前配置
    - 初始时编号为0
- checkCurConfigTime time.Time：检查当前配置的时刻
- inShards map[int]InShardInfo：待迁入的分片编号-分片信息
- outShards map[int]OutShardInfo：待迁出的分片编号-分片信息
- newConfig chan bool：新配置标志管道
    - 长度为1

### 存储结构体：KVStore struct

- ShardDatas [shardctrler.NShards]ShardData：分片数据列表
    - type ShardData map[string]string
    - 各元素初始值为nil
- ConfigNums [shardctrler.NShards]int：分片对应的配置编号列表
    - 各元素初始值为0

### 分片信息相关结构体

**待迁入分片信息结构体：InShardInfo struct**

- FromGID int：分片的源GID

**待迁出分片信息结构体：OutShardInfo struct**

- ToGID int：分片的目标GID
- Servers []string：分片的目标RG包含的服务器列表
- ConfigNum int：分片对应的配置编号

### 命令相关结构体

**操作结构体：Op struct**

- Id int：操作id
- ClientId int64Id：发起操作的client id
- Type OpType：操作类型
    - 取值：GetV、PutKV、AppendKV
- Key string：key值
- Value string：value值

注：Id、ClientId唯一确定一个Op结构体

**merge请求结构体：MergeReq struct**

- GID int：源服务器所在RG的GID
- ConfigNum int：源服务器发出请求时的配置编号
- Shard int：请求merge的分片编号
- ShardData ShardData：请求merge的分片数据
- ClientId2ExecutedOpId map[Int64Id]int：源服务器发出请求时所记录的client id-被execute的最大操作id

注：GID、ConfigNum、Shard唯一确定一个merge请求结构体

**delete请求结构体：DeleteReq struct**

- ConfigNum int：源服务器发出请求时的配置编号
- Shard int：请求delete的分片编号

注：ConfigNum、Shard唯一确定一个delete请求结构体

**空命令结构体：Nop struct**

**命令process结果结构体：ProcessResult struct**

- Type CommandType：命令类型
    - 取值：OpCmd、MergeReqCmd、DeleteReqCmd、ConfigCmd
- Id int：操作id
- ClientId int64Id：发起操作的client id
- Value string：value值
- GID int：请求来源的GID
- ConfigNum int：源服务器发出请求时的配置编号
- Shard int：请求处理的分片编号
- Err Err：错误类型

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
    2. 根据ck.config.Shards确定当前分片所在RG的GID，若ck.config.Groups中无该GID对应的服务器列表，跳到步骤6
    3. 调用ck.GetTargetLeader确定targetLeader
    4. 准备好Get RPC的空返回参数，向targetLeader发送Get RPC请求
    5. 后置处理
        1. 若未收到targetLeader的ACK，调用ck.UpdateTargetLeader更换targetLeader，回到步骤4
        2. 若收到targetLeader的ACK且reply.Err == OK，将返回值置为reply.Value，跳到步骤7
        3. 若收到targetLeader的ACK且reply.Err == ErrWrongGroup，跳到步骤6
        4. 若收到targetLeader的ACK且reply.Err == ErrNotArrivedShard，回到步骤4
        5. 若收到targetLeader的ACK且reply.Err == ErrWrongLeader，调用ck.UpdateTargetLeader更换targetLeader，回到步骤4
        6. 若收到targetLeader的ACK且reply.Err == ErrOvertime，调用ck.UpdateTargetLeader更换targetLeader，回到步骤4
    6. 休息固定长度（ConfigQueryInterval）的时段，调用ck.sm.Query(-1)更新ck.config
    7. ck.nextOpId++，返回所得的value值

**Put & Append操作：ck.PutAppend**

- 参数：**●**key string：key值 **●**value string：value值 **●**op string：操作类型
- 无返回值
- 流程
    1. 生成PutAppend RPC的输入参数（根据ck.nextOpId及ck.op）
    2. 根据ck.config.Shards确定当前分片所在RG的GID，若ck.config.Groups中无该GID对应的服务器列表，跳到步骤6
    3. 调用ck.GetTargetLeader确定targetLeader
    4. 准备好PutAppend RPC的空返回参数，向targetLeader发送PutAppend RPC请求
    5. 后置处理
        1. 若未收到targetLeader的ACK，调用ck.UpdateTargetLeader更换targetLeader，回到步骤4
        2. 若收到targetLeader的ACK且reply.Err == OK，跳到步骤7
        3. 若收到targetLeader的ACK且reply.Err == ErrWrongGroup，跳到步骤6
        4. 若收到targetLeader的ACK且reply.Err == ErrNotArrivedShard，回到步骤4
        5. 若收到targetLeader的ACK且reply.Err == ErrWrongLeader，调用ck.UpdateTargetLeader更换targetLeader，回到步骤4
        6. 若收到targetLeader的ACK且reply.Err == ErrOvertime，调用ck.UpdateTargetLeader更换targetLeader，回到步骤4
    6. 休息固定长度（ConfigQueryInterval）的时段，调用ck.sm.Query(-1)更新ck.config
    7. ck.nextOpId++，返回所得的value值

## server端

### Note

- server端服务可分为Service层、Raft层，两层之间通过applyCh管道交互
- 注意区别以下名词：
    - **commit**（Raft层）：命令被commit后可在rf.applyTicker中被apply
    - **apply**（Raft层）：快照或命令被apply后其对应消息被插入applyCh管道
    - **process**（Service层）
        - process快照：根据快照数据恢复kv.kvStore及kv.clientId2executedOpId
        - process操作：决定是否要execute操作
        - process配置：决定是否要install配置
        - process merge请求：决定是否要merge分片
        - process delete请求：决定是否要delete分片
    - **execute**（Service层）：执行SMR中的操作
    - **install**（Service层）：安装SMR中的配置
    - **merge**（Service层）：合并SMR中请求指定的分片
    - **delete**（Service层）：删除SMR中请求指定的分片

### 2类参数

**MergeShardDatas RPC参数**

- 输入参数
    - GID int：源服务器所在RG的GID
    - ConfigNum int：源服务器发出请求时的配置编号
    - Shard int：请求merge的分片编号
    - ShardData ShardData：请求merge的分片数据
    - ClientId2ExecutedOpId map[Int64Id]int：源服务器发出请求时所记录的client id-被execute的最大操作id
- 返回参数
    - Err Err：错误类型

**DeleteShardDatas Process参数**

- 输入参数
    - ConfigNum int：源服务器发出请求时的配置编号
    - Shard int：请求delete的分片编号
- 返回参数
    - Err Err：错误类型

### 6个Handler子方法（3个prepare+3个wait）

**kv.prepareForOpProcess**

- 参数：**●**op Op：操作
- 返回值：**●**Err：错误类型 **●**int：日志条目索引
- 执行过程
    1. 执行kv.rf.Start，返回index, _, isLeader
    2. 若isLeader == true，返回OK, index；若isLeader == false，返回ErrWrongLeader, -1

**kv.prepareForMergeReqProcess**

- 参数：**●**mReq MergeReq：merge请求
- 返回值：**●**Err：错误类型 **●**int：日志条目索引
- 执行过程
    - 直接返回的情况
        - 若mReq.ConfigNum < kv.curConfig.Num，返回ErrOutdatedShard, -1
        - 若mReq.ConfigNum ≤ kv.kvStore.ConfigNums[mReq.Shard]，返回ErrRepeatedShard, -1
    1. 执行kv.rf.Start，返回index, _, isLeader
    2. 若isLeader == true，返回OK, index；若isLeader == false，返回ErrWrongLeader, -1

**kv.prepareForDeleteReqProcess**

- 参数：**●**dReq DeleteReq：delete请求
- 返回值：**●**Err：错误类型 **●**int：日志条目索引
- 执行过程
    - 直接返回的情况
        - 若dReq.ConfigNum ≤ kv.kvStore.ConfigNums[dReq.Shard]，返回ErrRepeatedShard, -1
    1. 执行kv.rf.Start，返回index, _, isLeader
    2. 若isLeader == true，返回OK, index；若isLeader == false，返回ErrWrongLeader, -1

**kv.waitForOpProcess**

- 参数：**●**op Op：操作 **●**index int：日志条目索引
- 返回值：**●**Err：错误类型 **●**string：value值
- 执行过程
    1. 根据index得到对应的process结果管道ch
    2. 开启计时器
    - 若首先等待到ch管道中有process结果processedResult被取出
        1. 若检测到processedResult与op不一致（processResult.Type ≠ OpCmd或processedResult.ClientId ≠ op.ClientId或processedResult.Id ≠ op.Id），将返回的错误类型定为ErrWrongLeader；反之，将返回的错误类型定为processedResult.Err，若该错误类型为OK，则将返回的value值置为processedResult.Value
    - 若首先等待到计时器过时
        1. 将返回的错误类型定为ErrOvertime
    1. 停止计时器
    2. 删除ch管道

**kv.waitForMergeReqProcess**

- 参数：**●**mReq MergeReq：merge请求 **●**index int：日志条目索引
- 返回值：**●**Err：错误类型
- 执行过程
    1. 根据index得到对应的process结果管道ch
    2. 开启计时器
    - 若首先等待到ch管道中有process结果processedResult被取出
        1. 若检测到processedResult与mReq不一致（processResult.Type ≠ MergeReqCmd或processedResult.GID ≠ mReq.GID或processedResult.ConfigNum ≠ mReq.ConfigNum或processedResult.Shard ≠ mReq.Shard），将返回的错误类型定为ErrWrongLeader；反之，将返回的错误类型定为processedResult.Err
    - 若首先等待到计时器过时
        1. 将返回的错误类型定为ErrOvertime
    1. 停止计时器
    2. 删除ch管道

**kv.waitForDeleteReqProcess**

- 参数：**●**dReq DeleteReq：delete请求 **●**index int：日志条目索引
- 返回值：**●**Err：错误类型
- 执行过程
    1. 根据index得到对应的process结果管道ch
    2. 开启计时器
    - 若首先等待到ch管道中有process结果processedResult被取出
        1. 若检测到processedResult与dReq不一致（processResult.Type ≠ DeleteReqCmd或processedResult.ConfigNum ≠ dReq.ConfigNum或processedResult.Shard ≠ dReq.Shard），将返回的错误类型定为ErrWrongLeader；反之，将返回的错误类型定为processedResult.Err
    - 若首先等待到计时器过时
        1. 将返回的错误类型定为ErrOvertime
    1. 停止计时器
    2. 删除ch管道

### 4类Handler

**Get RPC Handler (kv.Get)**

- 参数：**●**args *****GetArgs：输入参数 **●**reply *****GetReply：返回参数
- 无返回值
- 执行过程
    1. 准备好需要process的Op
    2. 调用kv.prepareForOpProcess准备process操作
    3. 若kv.prepareForOpProcess返回的Err值prepareErr ≠ OK，令reply.Err = prepareErr，直接返回
    4. 调用kv.waitForOpProcess等待操作被process，令reply.Err及reply.Value分别等于kv.waitForOpProcess返回的错误类型及value值

**Put & Append RPC Handler (kv.PutAppend)**

- 参数：**●**args *****PutAppendArgs：输入参数 **●**reply *****PutAppendReply：返回参数
- 无返回值
- 执行过程
    1. 准备好需要process的Op
    2. 调用kv.prepareForOpProcess准备process操作
    3. 若kv.prepareForOpProcess返回的Err值prepareErr ≠ OK，令reply.Err = prepareErr，直接返回
    4. 调用kv.waitForOpProcess等待操作被process，令reply.Err等于kv.waitForOpProcess返回的错误类型

**MergeShardDatas RPC Handler (kv.MergeShardDatas)**

- 参数：**●**args *****MergeShardDatasArgs：输入参数 **●**reply *****MergeShardDatasReply：返回参数
- 无返回值
- 执行过程
    1. 准备好需要process的merge请求
    2. 调用kv.prepareForMergeReqProcess准备process merge请求
    3. 若kv.prepareForMergeReqProcess返回的Err值prepareErr ≠ OK，令reply.Err = prepareErr，直接返回
    4. 调用kv.waitForMergeReqProcess等待merge请求被process，令reply.Err等于kv.waitForMergeReqProcess返回的错误类型

**DeleteShardDatas Process Handler (kv.DeleteShardDatas)**

- 参数：**●**args *****DeleteShardDatasArgs：输入参数 **●**reply *****DeleteShardDatasReply：返回参数
- 无返回值
- 执行过程
    1. 准备好需要process的delete请求
    2. 调用kv.prepareForDeleteReqProcess准备process delete请求
    3. 若kv.prepareForDeleteReqProcess返回的Err值prepareErr ≠ OK，令reply.Err = prepareErr，直接返回
    4. 调用kv.waitForDeleteReqProcess等待delete请求被process，令reply.Err等于kv.waitForDeleteReqProcess返回的错误类型

### 3个Ticker

**总指挥：kv.processor**

- 功能：控制快照、操作、配置、请求被process的过程
- 循环过程
    1. 从applyCh管道中取出消息
    - 若取出的消息为快照消息
        1. 根据快照数据恢复kv.clientId2executedOpId、kv.kvStore、kv.curConfig、kv.inShards、kv.outShards
        2. 若kv.curConfig被更新为新配置，将kv.checkCurConfigTime更新为time.Now()，若同时kv.newConfig管道中无数据，插入true作为新配置标志（在kv.mu临界区外进行）
    - 若取出的消息为操作消息
        - 直接跳到步骤4的情况（记shard为op.Key对应的分片编号）
            - kv.curConfig.Shards[shard] ≠ kv.gid（说明当前配置中RG不含shard）
                - 令opResult.Err = ErrWrongGroup
            - shard在inShards中（说明RG需要shard但shard还没到）
                - 令opResult.Err = ErrNotArrivedShard
        1. 令opResult.Err = OK，若操作类型为GetV，或操作类型为PutKV / AppendKV且根据kv.clientId2executedOpId可知操作之前未被execute过，execute操作并更新kv.clientId2executedOpId
            1. 若操作类型为GetV，execute操作时，根据op.Key从kv.kvStore中获得opResult.Value
            2. 若操作类型为PutKV / AppendKV，execute操作时，根据op.Key-op.Value更新kv.kvStore
        2. 将操作process结果插入其日志条目索引所对应的process结果管道中（表示操作已被process）
        3. 若kv.maxraftstate > 0且当前Raft层状态信息长度大于kv.maxraftstate，调用kv.rf.Snapshot截取快照
    - 若取出的消息为配置消息
        - 直接跳到步骤6的情况
            - newConfig.Num ≤ kv.curConfig.Num（说明新配置已被intall过）
        1. install新配置（令kv.curConfig = *newConfig）
        - 若kv.curConfig.Num == 1
            1. 使服务器所在RG中的分片生效：令kv.kvStore.ShardDatas[shard] = make(ShardData)
        - 若kv.curConfig.Num ≠ 1
            1. 根据newConfig安装前服务器的配置（prevConfig）更新kv.inShards及kv.outShards
                1. 若同时满足①kv.curConfig.Shards[shard] == kv.gid（说明当前配置中RG需要该分片） ②prevConfig.Shards[shard] ≠ kv.gid（说明先前配置中RG不含该分片） ③shard在kv.kvStore中的配置编号小于kv.curConfig.Num（说明RG需要的该分片还没到）：将shard对应的InShardInfo插入kv.inShards
                2. 若同时满足①kv.curConfig.Shards[shard] ≠ kv.gid（说明当前配置中RG不需要该分片） ②prevConfig.Shards[shard] == kv.gid（说明先前配置中RG含有该分片）：将shard对应的OutShardInfo（其中ConfigNum被置为kv.curConfig.Num）插入kv.outShards
        1. 对于不在kv.inShards且不在kv.outShards中的shard，将其在kv.kvStore中的配置编号更新为kv.curConfig.Num
        2. 将kv.checkCurConfigTime更新为time.Now()，若kv.newConfig管道中无数据，插入true作为新配置标志（在kv.mu临界区外进行）
        3. 若kv.maxraftstate > 0且当前Raft层状态信息长度大于kv.maxraftstate，调用kv.rf.Snapshot截取快照
    - 若取出的消息为merge请求消息
        - 直接跳到步骤6的情况
            - mReq.ConfigNum < kv.curConfig.Num（说明待merge的分片过时）
                - 令mReqResult.Err = ErrOutdatedShard
            - mReq.ConfigNum不大于mReq.Shard在kv.kvStore中的配置编号（说明分片已被merge）
                - 令mReqResult.Err = ErrRepeatedShard
        1. merge mReq.Shard（将mReq.ShardData深拷贝至kv.kvStore.ShardDatas[mReq.Shard]）
        2. 将其在kv.kvStore中的配置编号更新为mReq.ConfigNum
        3. 根据mReq.ClientId2ExecutedOpId更新kv.clientId2executedOpId
        4. 在kv.inShards中删除mReq.Shard
        5. 将merge请求process结果插入其日志条目索引所对应的process结果管道中（表示merge请求已被process）
    - 若取出的消息为delete请求消息
        - 直接跳到步骤5的情况
            - dReq.ConfigNum不大于dReq.Shard在kv.kvStore中的配置编号（说明分片已被delete或被更高配置编号的分片覆盖）
                - 令dReqResult.Err = ErrRepeatedShard
        1. delete dReq.Shard（令kv.kvStore.ShardDatas[dReq.Shard] = nil）
        2. 将其在kv.kvStore中的配置编号更新为dReq.ConfigNum
        3. 在kv.outShards中删除dReq.Shard
        4. 将delete请求process结果插入其日志条目索引所对应的process结果管道中（表示delete请求已被process）

**配置获取：kv.configPoller**

- 功能：控制新配置获取的过程
- 循环过程
    1. 休息固定长度（ConfigPollPeriod）的时段
    - 重新开始循环过程的情况
        - 服务器非leader
    1. 若kv.inShards非空且当前时刻距kv.checkCurConfigTime超过StartNopTimeout，将kv.checkCurConfigTime更新为time.Now()，并执行kv.rf.Start(Nop{})
    2. 调用kv.mck.Query(kv.curConfg.Num+1)获取新配置newConfig
    - 重新开始循环过程的情况
        - newConfig.Num ≠ kv.curConfig.Num+1（未能成功获取kv.curConfig的下一配置）
        - newConfig.Num ≤ kv.curConfig.Num（newConfig已被install过）
        - kv.inShards非空（当前配置所需的分片未全部到达）
    1. 执行kv.rf.Start(newConfig)

**分片迁移：kv.shardMigrant**

- 功能：控制分片迁移的过程
- 循环过程
    1. 休息固定长度（ShardMigratePeriod）的时段（若rf.newConfig管道中有新配置标志，提前结束休息）
    - 重新开始循环过程的情况
        - 服务器非leader
        - kv.outShards为空
    1. 对于kv.outShards中的各个分片，依次调用kv.migrateShard方法执行单个分片迁移过程
- 单个分片迁移过程：kv.migrateShard
    - 参数：**●**shard int：分片编号 **●**info OutShardInfo：
    - 无返回值
    1. 生成MergeShardDatas RPC的输入参数（其中ShardData及ClientId2ExecutedOpId应为kv.kvStore.ShardDatas[shard]及kv.clientId2ExecutedOpId的深拷贝副本）
    2. **启动单独的协程**，传入目标RG包含的服务器列表（info.Servers）及MergeShardDatas RPC的输入参数
    3. 调用kv.GetTargetLeader确定targetLeader
    4. 准备好MergeShardDatas RPC的空返回参数，向targetLeader发送MergeShardDatas RPC请求
    5. 后置处理
        1. 若未收到targetLeader的ACK，调用kv.UpdateTargetLeader更换targetLeader，回到步骤4
        2. 若收到targetLeader的ACK且reply.Err == OK，跳到步骤6
        3. 若收到targetLeader的ACK且reply.Err == ErrWrongLeader，调用kv.UpdateTargetLeader更换targetLeader，回到步骤4
        4. 若收到targetLeader的ACK且reply.Err == ErrOutdatedShard，跳到步骤6
        5. 若收到targetLeader的ACK且reply.Err == ErrRepeatedShard，跳到步骤6
        6. 若收到targetLeader的ACK且reply.Err == ErrOvertime，调用kv.UpdateTargetLeader更换targetLeader，回到步骤4
    6. 生成DeleteShardDatas Process的输入参数（其中ConfigNum = mArgs.ConfigNum, Shard = mArgs.Shard）并准备好其空返回参数，调用kv.DeleteShardDatas发起delete分片请求并等待其被处理

## 注意点

- 注意两个深拷贝的时机
    - kv.shardMigrant中生成MergeShardDatas RPC的输入参数时，ShardData及ClientId2ExecutedOpId应为kv.kvStore.ShardDatas[shard]及kv.clientId2ExecutedOpId的深拷贝副本，防止Service层的修改与Raft层持久化日志时的读取发生竞争
    - kv.processor中merge mReq.Shard时应将mReq.ShardData深拷贝至kv.kvStore.ShardDatas[mReq.Shard]，防止同一RG中不同服务器对迁入的同一分片发生竞争
- 注意分片迁移过程中接收端应支持merge超前分片，以提高GC效率
    - 例如RG101先处于5号配置，RG100还处于4号配置，在5号配置下，RG101需将1号分片迁至RG100，此时RG100虽处于4号配置，但可先merge RG101迁入的对应5号配置的1号分片（5 > 4）；然后RG101可在成功收到RG100的ACK时发起1号分片的delete请求并等待其被处理
    - 若接收端不支持merge超前分片，发送端delete该分片的过程将延后，导致TestChallenge1Delete会出现失败
- 注意通过空命令解决Service层与Raft层冲突的问题
    - 若不引入空命令，存在以下情况：S101-1作为leader在日志条目索引为339的位置添加了merge 5号分片的merge请求命令，成功process了该请求，并向5号分片的发送者返回ACK（发送者因认为迁移分片的过程成功了而不会重新发送）；随后S101-2成为新的leader，却因为索引为339的merge请求term落后于rf.currentTerm而无法将其主动commit，造成5号分片的请求始终无法得到响应
    - 通过kv.checkCurConfigTime进行控制，在kv.configPoller中若检测到当前配置所需的分片长时间内未全部到达，通过向日志中添加空命令来被动commit其之前的日志条目
- 注意kv.shardMigrant中应定期检查是否有需要迁移的分片并迁移
    - 若仅在rf.newConfig管道中有新配置标志时开始检查需要迁移的分片并迁移，存在以下情况：S101-1 install了新配置，在kv.shardMigrant的循环过程恰好开始推进之前，突然失去了原有的leader身份，导致该循环过程被跳过；S101-1又立刻成为leader并一直保持，但kv.shardMigrant的循环过程始终无rf.newConfig中的新配置标志驱动推进，需要对外迁移的1号分片无法对外迁移，RG100等不到应迁入的1号分片，造成1号分片的请求始终无法得到响应
    - 应在固定长度（ShardMigratePeriod）的时段后检查需要迁移的分片并迁移，而非仅由rf.newConfig管道中的新配置标志驱动迁移过程，保证最终kv.outShards中的分片均可完成迁移