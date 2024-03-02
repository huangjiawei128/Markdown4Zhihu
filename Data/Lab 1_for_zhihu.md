# Lab 1

## 整体思路

### Worker向Coordinator索要任务

- 涉及的函数/方法：AskForTask、call
- 流程
    1. 通过call函数联系Coordinator调用Coordinator.DistributeTask方法分配任务

### Coordinator向Worker分配任务

- 涉及的函数/方法：Coordinator.DistributeTask、Coordinator.StartTask、Coordinator.AllMapTasksDone、Coordinator.ProduceReduceTasks、Coordinator.AllReduceTasksDone
- 流程
    - 若Coordinator.Phase为MapPhase
        - 若Coordinator.MapChan中有可分配的任务
            1. 从Coordinator.MapChan中取出任务并调用StartTask方法启动任务（更新该任务在Coordinator.MapTaskInfoMap中的信息）
            2. 返回这一种类必为MapTask的任务
        - 若Coordinator.MapChan中无可分配的任务
            1. 调用Coordinator.cond.Wait方法使协程陷入等待状态直到Coordinator.MapChan中有可分配的任务或所有map任务均处于Done状态（调用AllMapTasksDone方法判断）
            2. 若所有map任务均处于Done状态，并且此时**Coordinator.Phase仍为MapPhase**，调用ProduceReduceTasks方法生产所有reduce任务并将Coordinator.Phase置为ReducePhase
            3. 重新执行分配过程
    - 若Coordinator.Phase为ReducePhase
        - 若Coordinator.ReduceChan中有可分配的任务
            1. 从Coordinator.ReduceChan中取出任务并调用StartTask方法启动任务（更新该任务在Coordinator.ReduceTaskInfoMap中的信息）
            2. 返回这一种类必为ReduceTask的任务
        - 若Coordinator.ReduceChan中无可分配的任务
            1. 调用Coordinator.cond.Wait方法使协程陷入等待状态直到Coordinator.ReduceChan中有可分配的任务或所有reduce任务均处于Done状态（调用AllReduceTasksDone方法判断）
            2. 若所有reduce任务均处于Done状态，并且此时**Coordinator.Phase仍为ReducePhase**，将Coordinator.Phase置为EndPhase
            3. 重新执行分配过程
    - 若Coordinator.Phase为EndPhase
        1. 返回种类为EndTask的任务

### Worker执行任务

- 涉及的函数/方法：DoMapTask、DoReduceTask
- 流程
    - 对于map任务，调用DoMapTask函数
        1. 将（文件名，文件内容）利用map函数映射至K-V对列表
        2. 将K-V对列表根据Key值划分为N组（N等于reduce任务数）
        3. 将每组K-V对保存至文件mr-i-j中（i表示任务id，j表示用于第j个reduce任务，0≤j<N）
    - 对于reduce任务，调用DoReduceTask函数
        1. 对于第j个reduce任务（0≤j<N，N等于reduce任务数），读取所有mr-i-j文件中的K-V对（i表示map任务id），并将其按照Key值排序
        2. 将各个（Key值，Value值列表）利用reduce函数映射至单个K-V对并保存至临时文件mr-tmp-j中（reduce函数执行时可能遇到crash情况）
        3. 将临时文件更名为mr-out-j，即为最终reduce任务输出的文件

### Worker告知Coordinator任务完成

- 涉及的函数/方法：InformTaskDone、call
- 流程
    1. 通过call函数联系Coordinator调用Coordinator.DealTaskDone方法处理完成的任务

### Coordinator处理Worker完成的任务

- 涉及的函数/方法：Coordinator.DealTaskDone、Coordinator.EndTask
- 流程
    1. 调动Coordinator.EndTask方法结束任务（更新该任务在Coordinator.MapTaskInfoMap或Coordinator.ReduceTaskInfoMap中的信息）
    2. 若任务成功结束，则调用Coordinator.cond.Broadcast方法唤醒所有等待的协程

### 监控并处理crash问题

- 涉及的函数/方法：Coordinator.DetectCrash
- 流程
    - 根据Coordinator.TaskInfoMap不断检查是否有状态为Doing且执行时间超过最大等待时长（5s）的任务，直到进入EndPhase阶段（每次检查间隔1s）
        - 若有此类任务，将其状态置为Idle，重新塞进Coordinator.MapChan或Coordinator.ReduceChan中等待分配，并调用Coordinator.cond.Broadcast方法唤醒所有等待的协程

### Worker运行的顶层逻辑

- 涉及的函数/方法：Worker
- 流程
    - 不断调用AskForTask函数向Coordinator索要任务（每次索要间隔1s）
        - 若任务类型为MapTask，先调用DoMapTask函数执行任务，再调用InformTaskDone函数告知Coordinator任务完成
        - 若任务类型为ReduceTask，先调用DoReduceTask函数执行任务，再调用InformTaskDone函数告知Coordinator任务完成
        - 若任务类型为EndTask，结束索要任务的过程

### Coordinator运行的顶层逻辑

- 涉及的函数/方法：MakeCoordinator、Coordinator.ProduceMapTasks、Coordinator.server、Coordinator.DetectCrash、Coordinator.Done
- 流程
    1. 最顶层（mrcoordinator.go的main函数）先调用MakeCoordinator函数初始化Coordinator
        1. 设置Coordinator成员变量值
        2. 调用Coordinator.ProduceMapTasks方法生产所有map任务
        3. 通过调用Coordinator.server方法开启http.Serve携程监听来自Worker的RPC请求，开启Coordinator.DetectCrash线程监控并处理crash情况的发生
    2. 最顶层不断调用Coordinator.Done方法检查是否已进入EndPhase阶段（每次调用间隔1s），若满足则结束程序

## 工程架构

### worker.go

- 数据结构
    - **KeyValue** struct：K-V（键-值）对
        - Key string：键
        - Value string：值
- 函数/方法
    - **Worker**：Worker运行的顶层逻辑
        - 参数：**●**mapf func(string, string) []KeyValue：map函数 **●**reducef func(string, []string) string：reduce函数
        - 无返回值
    - **AskForTask**：Worker向Coordinator索要Task
        - 无参数
        - 返回值：**●***Task：索要到的任务 **●**bool：索要任务是否成功
    - **InformTaskDone**：Worker告知Coordinator任务完成
        - 参数：**●**task *Task：完成的任务
        - 返回值：**●**bool：告知任务完成是否成功
    - **DoMapTask**：Worker执行Map任务
        - 参数：**●**mapf func(string, string) []KeyValue：map函数 **●**task *Task：所需执行的map任务
        - 无返回值
    - **DoReduceTask**：Worker执行reduce任务
        - 参数：**●**reducef func(string, []string) string：reduce函数 **●**task *Task：所需执行的reduce任务
        - 无返回值
    - **call**：Worker向Coordinator发送一个RPC请求
        - 参数：**●**rpcname string：需要Coordinator执行的方法名称 **●**args interface{}：输入参数 **●**reply interface{}：返回参数
        - 返回值：**●**bool：是否有错误发生

### coordinator.go

- 数据结构
    - **Phase** int：执行阶段
        - 取值：MapPhase、ReducePhase、EndPhase
    - **TaskState** int：任务状态
        - 取值：Idle、Doing、Done
    - **TaskInfo** struct：任务信息
        - State TaskState：任务状态
        - StartTime time.Time：任务开始时间
        - Ptr *Task：任务指针
    - **Coordinator** struct：协调者
        - mu sync.Mutex：互斥锁
        - cond *sync.Cond：条件变量
        - Phase Phase：执行阶段
        - MapperNum int：map任务数
        - ReducerNum int：reduce任务数
        - Filenames []string：原始文件名
        - MapChan chan *Task：存放map任务的管道
        - ReduceChan chan *Task：存放reduce任务的管道
        - NextTaskId int：下一个分配的任务id
        - MapTaskInfoMap map[int]*TaskInfo：map任务id → 任务信息
        - ReduceTaskInfoMap map[int]*TaskInfo：reduce任务id → 任务信息
- 函数/方法
    - **Coordinator.GetNextTaskId**：Coordinator生成下一个任务id
        - 无参数
        - 返回值：**●**int：下一个任务id
    - **Coordinator.GetTaskInfoMap**：Coordinator返回任务种类对应的任务信息map
        - 参数：**●**taskKind TaskKind：任务种类
        - 返回值：**●**map[int]*TaskInfo：任务信息map
    - **Coordinator.GetTaskPtrChan**：Coordinator返回任务种类对应的存放任务的管道
        - 参数：**●**taskKind TaskKind：任务种类
        - 返回值：**●**chan *Task：存放任务的管道
    - **Coordinator.StartTask**：Coordinator启动任务
        - 参数：**●**taskId int：任务id **●**taskKind TaskKind：任务种类
        - 返回值：**●**bool：启动任务是否成功
    - **Coordinator.EndTask**：Coordinator结束任务
        - 参数：**●**taskId int：任务id **●**taskKind TaskKind：任务种类
        - 返回值：**●**bool：结束任务是否成功
    - **Coordinator.AllMapTasksDone**：Coordinator判断所有map任务是否均处于Done状态
        - 无参数
        - 返回值：**●**bool：所有map任务是否均处于Done状态
    - **Coordinator.AllReduceTasksDone**：Coordinator判断所有reduce任务是否均处于Done状态
        - 无参数
        - 返回值：**●**bool：所有reduce任务是否均处于Done状态
    - **Coordinator.ProduceMapTasks**：Coordinator生产所有map任务
        - 无参数
        - 无返回值
    - **Coordinator.ProduceReduceTasks**：Coordinator生产所有reduce任务
        - 无参数
        - 无返回值
    - **Coordinator.DistributeTask**：Coordinator向Worker分配任务
        - 参数：**●**args *DistributeTaskArgs：输入参数 **●**reply *DistributeTaskReply：返回参数
        - 返回值：**●**error：错误信息
    - **Coordinator.DealTaskDone**：Coordinator处理Worker完成的任务
        - 参数：**●**args *DealTaskDoneArgs：输入参数 **●**reply *DealTaskDoneReply：返回参数
        - 返回值：**●**error：错误信息
    - **Coordinator.server**：Coordinator开启http.Serve线程监听来自Worker的RPC请求
        - 无参数
        - 无返回值
    - **Coordinator.DetectCrash**：Coordinator监控并处理crash情况
        - 无参数
        - 无返回值
    - **Coordinator.Done**：Coordinator判断是否处于EndPhase执行阶段
        - 无参数
        - 返回值：**●**bool：是否处于EndPhase执行阶段
    - **MakeCoordinator**：创建Coordinator
        - 参数：**●**files []string：原始文件名 **●**nReduce int：reduce任务数
        - 返回值：**●***Coordinator：协调者

### **rpc.go**

- 数据结构
    - **TaskKind** int：任务种类
        - 取值：MapTask、ReduceTask、EndTask
    - **Task** struct：任务
        - TaskId int：任务id
        - TaskKind TaskKind：任务种类
        - ReducerNum int：reduce任务数
        - Filenames []string：任务所涉及的文件名
    - **DistributeTaskArgs** struct：任务分配输入参数
    - **DistributeTaskReply** struct：任务分配返回参数
        - Task Task：分配的任务
    - **DealTaskDoneArgs** struct：任务完成处理输入参数
        - TaskId int：所需处理的任务id
    - **DealTaskDoneReply** struct：任务完成处理返回参数
- 函数/方法
    - **coordinatorSock**：生成唯一的UNIX域套接字名
        - 无参数
        - 返回值：**●**string：UNIX域套接字名