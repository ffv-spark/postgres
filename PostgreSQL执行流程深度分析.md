# PostgreSQL程序执行流程 - 深度分析

**分析深度：** 非常详尽
**分析日期：** 2025-11-06
**版本：** 基于PostgreSQL开发分支

---

## 1. 服务器启动流程

### 1.1 入口点：src/backend/main/main.c

`src/backend/main/main.c`中的`main()`函数（第70-237行）是所有PostgreSQL服务器进程的通用入口点。所有服务器进程（postmaster、backend、standalone、bootstrap）都从这里开始执行。

```
main(int argc, char *argv[])
├─ 平台特定初始化 (startup_hacks)
├─ 进程状态显示设置 (save_ps_display_args)
├─ 内存和错误系统初始化
│  ├─ MyProcPid = getpid()
│  ├─ MemoryContextInit()
│  └─ set_stack_base()
├─ 区域设置初始化 (LC_COLLATE, LC_CTYPE, LC_MESSAGES等)
├─ 标准选项解析 (--help, --version, --describe-config)
├─ Root权限检查 (check_root)
└─ 根据第一个参数分发到适当的主函数：
   ├─ DISPATCH_CHECK → BootstrapModeMain()
   ├─ DISPATCH_BOOT → BootstrapModeMain()
   ├─ DISPATCH_FORKCHILD → SubPostmasterMain()
   ├─ DISPATCH_DESCRIBE_CONFIG → GucInfoMain()
   ├─ DISPATCH_SINGLE → PostgresSingleUserMain()
   └─ DISPATCH_POSTMASTER → PostmasterMain() [默认]
```

**关键变量：**
- `progname`：从argv[0]获取的程序名
- `dispatch_option`：决定运行哪种模式
- `MyProcPid`：当前进程的进程ID
- `reached_main`：跟踪是否到达main()的标志

### 1.2 Postmaster初始化：src/backend/postmaster/postmaster.c

`PostmasterMain()`函数（从第493行开始）是数据库服务器的核心初始化例程。

#### 1.2.1 初始化序列

```
PostmasterMain(int argc, char *argv[])
├─ 阶段1：进程设置
│  ├─ InitProcessGlobals()
│  ├─ PostmasterPid = MyProcPid
│  ├─ IsPostmasterEnvironment = true
│  └─ 初始化Win32信号（如果在Windows上）
│
├─ 阶段2：内存上下文设置
│  ├─ umask(PG_MODE_MASK_OWNER) - 设置严格权限
│  ├─ PostmasterContext = AllocSetContextCreate(TopMemoryContext, ...)
│  └─ MemoryContextSwitchTo(PostmasterContext)
│
├─ 阶段3：安装路径设置
│  └─ getInstallationPaths(argv[0])
│
├─ 阶段4：信号处理器设置
│  ├─ pqinitmask() - 初始化信号掩码
│  ├─ sigprocmask(SIG_SETMASK, &BlockSig, NULL) - 阻塞信号
│  ├─ pqsignal(SIGHUP, handle_pm_reload_request_signal)
│  ├─ pqsignal(SIGTERM/SIGINT/SIGQUIT, handle_pm_shutdown_request_signal)
│  ├─ pqsignal(SIGUSR1, handle_pm_pmsignal_signal)
│  ├─ pqsignal(SIGCHLD, handle_pm_child_exit_signal)
│  ├─ InitializeWaitEventSupport()
│  ├─ InitProcessLocalLatch()
│  ├─ 忽略SIGTTIN、SIGTTOU（终端信号）
│  ├─ 忽略SIGXFSZ（磁盘满）
│  └─ sigprocmask(SIG_SETMASK, &UnBlockSig, NULL) - 解除信号阻塞
│
├─ 阶段5：GUC选项初始化
│  └─ InitializeGUCOptions()
│
├─ 阶段6：命令行选项解析
│  ├─ 解析-B（shared_buffers）
│  ├─ 解析-D（data directory）
│  ├─ 解析-p（port）
│  ├─ 解析-h（listen addresses）
│  ├─ 解析-k（socket directory）
│  ├─ 解析其他配置选项
│  └─ 重置getopt()状态（optind = 1）
│
├─ 阶段7：配置文件加载
│  ├─ SelectConfigFiles(userDoption, progname)
│  ├─ 加载postgresql.conf
│  └─ 加载recovery.conf（如果需要）
│
├─ 阶段8：数据目录验证
│  ├─ checkDataDir()
│  ├─ checkControlFile()
│  └─ ChangeToDataDir()
│
├─ 阶段9：配置验证
│  ├─ 验证superuser_reserved_connections + reserved_connections < max_connections
│  ├─ 验证WAL归档设置
│  ├─ 验证WAL流式传输（max_wal_senders）
│  ├─ CheckDateTokenTables()
│  └─ 检查无效的GUC组合
│
└─ 阶段10：服务器主循环
   └─ ServerLoop() - 主事件循环
```

#### 1.2.2 共享内存初始化

共享内存设置在`ServerLoop()`和相关函数中进行：

```
共享内存设置：
├─ InitShmemAccess() - 初始化共享内存访问
├─ CreateOrAttachShmemStructs() - 创建/附加共享内存结构
│  ├─ PGSharedMemoryCreate()
│  │  ├─ 分配共享内存段
│  │  ├─ 根据shared_buffers、max_connections等设置大小
│  │  └─ 请求共享内存结构布局
│  ├─ CreateProcArray() - 创建进程数组
│  ├─ CreateLockFile() - 创建共享锁结构
│  ├─ CreateBufferPool() - 创建共享缓冲池
│  │  ├─ 分配缓冲区
│  │  ├─ 初始化缓冲区描述符
│  │  └─ 初始化缓冲区管理结构
│  ├─ CreateXLogControlData() - 初始化WAL控制结构
│  ├─ CreateSubTransactionData() - 初始化子事务结构
│  ├─ CreateMultiXactData() - 初始化多事务结构
│  ├─ CreateCLogControlData() - 初始化事务状态日志
│  ├─ CreateDistributedTransactionStateData() - 用于分布式事务
│  └─ CreateShmemIndexes() - 在共享内存中创建哈希表
```

### 1.3 Postmaster状态机

Postmaster使用状态机来管理服务器状态：

```
typedef enum PMState {
    PM_INIT,                      // Postmaster初始化中
    PM_STARTUP,                   // 等待启动进程
    PM_RECOVERY,                  // 归档恢复模式
    PM_HOT_STANDBY,              // 热备模式
    PM_RUN,                       // 正常运行状态
    PM_STOP_BACKENDS,            // 停止剩余后端
    PM_WAIT_BACKENDS,            // 等待后端退出
    PM_WAIT_XLOG_SHUTDOWN,       // 等待检查点器关闭检查点
    PM_WAIT_XLOG_ARCHIVAL,       // 等待归档器和WAL发送器
    PM_WAIT_IO_WORKERS,          // 等待I/O工作进程
    PM_WAIT_CHECKPOINTER,        // 等待检查点器关闭
    PM_WAIT_DEAD_END,            // 等待死端子进程
    PM_NO_CHILDREN                // 所有重要子进程已退出
} PMState;
```

### 1.4 后台进程启动

`ServerLoop()`函数启动后台进程：

```
LaunchMissingBackgroundProcesses()
├─ 启动进程 (StartupPMChild)
│  ├─ 读取控制文件
│  ├─ 如果需要执行WAL恢复
│  ├─ 建立recovery_ready状态
│  └─ 转换到PM_RUN状态
├─ 后台写进程 (BgWriterPMChild)
│  ├─ 扫描缓冲池查找脏页
│  ├─ 将页面写入磁盘
│  └─ 减轻正常后端的写工作负载
├─ 检查点进程 (CheckpointerPMChild)
│  ├─ 执行周期性检查点
│  ├─ 创建一致性恢复点
│  └─ 管理检查点元数据
├─ WAL写进程 (WalWriterPMChild)
│  ├─ 将WAL缓冲区写入磁盘
│  ├─ 管理WAL刷新
│  └─ 处理同步复制等待
├─ WAL接收器 (WalReceiverPMChild) [如果启用了复制]
│  ├─ 从主服务器接收WAL
│  ├─ 将WAL写入本地磁盘
│  └─ 管理备用恢复
├─ 自动清理启动器 (AutoVacLauncherPMChild)
│  ├─ 监控表的vacuum需求
│  ├─ 生成autovacuum工作进程
│  └─ 管理vacuum调度
├─ WAL归档器 (PgArchPMChild) [如果启用了归档]
│  ├─ 将WAL文件复制到归档位置
│  ├─ 管理WAL归档目录
│  └─ 处理归档错误
├─ 系统日志记录器 (SysLoggerPMChild) [如果记录到文件]
│  ├─ 管理日志文件轮换
│  ├─ 收集日志消息
│  └─ 处理stderr重定向
├─ 逻辑复制启动器 (LogicalLauncherPMChild) [如果启用了逻辑复制]
│  ├─ 管理逻辑复制工作进程
│  ├─ 应用来自发布者的更改
│  └─ 处理订阅管理
└─ 后台工作进程（自定义）
   ├─ 用户定义的后台工作进程
   ├─ 基于扩展的后台作业
   └─ 自定义监控/维护任务
```

### 1.5 连接接受循环

```
ServerLoop()
├─ 循环：主事件循环
│  ├─ 检查postmaster信号（SIGTERM、SIGHUP等）
│  ├─ 检查子进程退出
│  ├─ 启动缺失的后台进程
│  ├─ WaitEventSetWait() - 在监听套接字上等待事件
│  │  └─ 基于超时：100ms
│  ├─ 处理就绪事件：
│  │  ├─ 如果监听套接字就绪：
│  │  │  ├─ 接受客户端连接
│  │  │  └─ BackendStartup()
│  │  │     ├─ 检查是否接受连接（canAcceptConnections）
│  │  │     ├─ Fork新后端进程
│  │  │     │  └─ 设置调度选项为DISPATCH_FORKCHILD
│  │  │     └─ 父进程：添加到子进程列表
│  │  └─ 如果postmaster管道就绪：
│  │     └─ 处理父进程消息
│  └─ 处理postmaster状态转换
└─ 收到关闭信号时退出循环
```

---

## 2. 客户端连接流程

### 2.1 连接请求处理

当客户端连接时，postmaster的`BackendStartup()`函数（在postmaster.c中）处理连接：

```
BackendStartup(ClientSocket *client_sock)
├─ 检查连接接受状态
│  └─ canAcceptConnections(BACKEND_TYPE_NORMAL)
│     ├─ CAC_OK - 接受正常连接
│     ├─ CAC_WAITBACKEND - 等待后端退出
│     └─ CAC_SHUTDOWN - 拒绝连接
│
├─ Fork新子进程
│  ├─ fork() → 子进程继承套接字
│  ├─ 父进程（Postmaster）：
│  │  ├─ 关闭子进程侧套接字
│  │  ├─ 将PMChild条目添加到ActiveChildList
│  │  ├─ 增加后端计数
│  │  └─ 继续接受连接
│  │
│  └─ 子进程（后端进程）：
│     ├─ 关闭postmaster监听套接字
│     ├─ 设置调度选项为DISPATCH_FORKCHILD
│     └─ 调用PostgresMain()（或在EXEC_BACKEND上调用SubPostmasterMain()）
```

### 2.2 认证过程

后端进程在`PostgresMain()`（tcop/postgres.c）中以认证开始执行：

```
PostgresMain(int argc, char *argv[], const char *username)
├─ 初始化阶段
│  ├─ InitPostgres() - 初始化后端
│  │  ├─ BaseInit()
│  │  │  ├─ InitProcessGlobals()
│  │  │  ├─ InitBufferPoolAccess()
│  │  │  │  ├─ AdjustNBuffers() - 设置后端缓冲区计数
│  │  │  │  └─ InitLocalBufferPool()
│  │  │  ├─ InitProcessAccessInfo()
│  │  │  ├─ InitStorageEngineBackend()
│  │  │  └─ InitAccessMethods()
│  │  ├─ InitMultiXactState()
│  │  ├─ InitUnloggedRelationSync()
│  │  ├─ InitPostgresRelationCache() - 加载relcache
│  │  ├─ InitTempTableNamespace() - 创建临时模式
│  │  ├─ InitializeMaxBackendId()
│  │  └─ EmitInvalidationMessage() - 处理SI消息
│  │
│  ├─ ClientAuthentication(Port *port)
│  │  ├─ 从pg_hba.conf获取认证信息
│  │  ├─ 设置AuthenticationTimeout（通常为60秒）
│  │  ├─ 将连接与pg_hba.conf规则匹配
│  │  │  ├─ 检查网络/主机
│  │  │  ├─ 检查数据库
│  │  │  ├─ 检查用户
│  │  │  └─ 获取认证方法
│  │  ├─ 调用认证方法处理程序
│  │  │  ├─ AUTH_REQ_OK → 客户端已批准
│  │  │  ├─ AUTH_REQ_PASSWORD → 发送密码挑战
│  │  │  ├─ AUTH_REQ_MD5 → 发送MD5密码挑战
│  │  │  ├─ AUTH_REQ_SCRAM_SHA_256 → SCRAM认证
│  │  │  ├─ AUTH_REQ_GSS → GSSAPI认证
│  │  │  ├─ AUTH_REQ_SSPI → Windows SSPI认证
│  │  │  ├─ AUTH_REQ_CERT → SSL证书认证
│  │  │  ├─ AUTH_REQ_SASL → SASL认证
│  │  │  └─ AUTH_REQ_LDAP → LDAP认证
│  │  ├─ 从客户端读取认证响应
│  │  ├─ 验证凭据
│  │  │  ├─ 检查用户在pg_authid中是否存在
│  │  │  ├─ 验证密码哈希
│  │  │  └─ 检查用户未被禁用
│  │  └─ 返回认证结果
│  │
│  └─ SetSessionUserId(userId) - 设置已认证用户
│
├─ 会话初始化阶段
│  ├─ 创建会话内存上下文
│  ├─ 为会话初始化GUC变量
│  ├─ InitCatalogCache() - 加载系统目录
│  ├─ QueryCancelPending = false
│  ├─ InterruptPending = false
│  └─ ProcSignalInit()
│
└─ 主查询循环（见第3节）
```

### 2.3 后端进程创建数据流

```
后端进程生命周期：
┌─────────────────────────────────────────┐
│ Postmaster在套接字上接收连接             │
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│ BackendStartup(ClientSocket)            │
│ ├─ 检查是否可以接受连接                  │
│ └─ 调用fork()创建子进程                  │
└──────────────────┬──────────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
        ▼ (父进程)            ▼ (子进程)
   Postmaster            PostgresMain()
   ├─ 跟踪子进程        ├─ InitPostgres()
   ├─ 更新计数器        ├─ ClientAuthentication()
   └─ 继续循环          ├─ 会话初始化
                        └─ 查询处理循环
```

### 2.4 会话初始化

```
会话初始化序列：
├─ 内存上下文设置
│  ├─ 创建TopMemoryContext（如果不存在）
│  ├─ 创建MessageContext（用于处理消息）
│  ├─ 创建PostmasterContext（如果在postmaster中）
│  ├─ 创建CacheContext（用于缓存）
│  └─ 创建PortalContext（用于查询门户）
│
├─ 关系缓存设置
│  ├─ 加载系统目录缓存
│  ├─ 为以下内容初始化关系描述符：
│  │  ├─ pg_class
│  │  ├─ pg_attribute
│  │  ├─ pg_index
│  │  ├─ pg_constraint
│  │  └─ 其他系统表
│  └─ 注册缓存失效函数
│
├─ 锁和事务设置
│  ├─ 初始化LOCALLOCK哈希表
│  ├─ 初始化事务状态
│  ├─ 设置默认隔离级别
│  └─ 初始化快照管理器
│
└─ 连接特定设置
   ├─ 设置CurrentUserId = 已认证用户
   ├─ 设置CurrentSchemaId
   ├─ 初始化search_path
   └─ 从ALTER SYSTEM/SET设置会话GUC参数
```

---

## 3. 查询处理流程

### 3.1 查询接收

`PostgresMain()`中的主查询循环从客户端读取查询：

```
postgres.c中的主查询处理循环：
├─ 循环：while (!ignore_till_sync)
│  │
│  ├─ start_xact_command()
│  │  ├─ 如果尚未启动则启动事务
│  │  └─ 设置命令状态
│  │
│  ├─ ReadCommand(inBuf) - 从客户端读取下一个命令
│  │  ├─ 检查是交互式（stdin）还是套接字后端
│  │  │
│  │  ├─ SocketBackend(inBuf)
│  │  │  ├─ HOLD_CANCEL_INTERRUPTS()
│  │  │  ├─ pq_startmsgread()
│  │  │  ├─ pq_getbyte() - 读取消息类型
│  │  │  │  消息类型：
│  │  │  │  ├─ 'Q' (PqMsg_Query) - 简单查询
│  │  │  │  ├─ 'P' (PqMsg_Parse) - 解析语句
│  │  │  │  ├─ 'B' (PqMsg_Bind) - 绑定参数
│  │  │  │  ├─ 'D' (PqMsg_Describe) - 描述语句
│  │  │  │  ├─ 'E' (PqMsg_Execute) - 执行预处理语句
│  │  │  │  ├─ 'S' (PqMsg_Sync) - 同步（扩展协议）
│  │  │  │  ├─ 'C' (PqMsg_Close) - 关闭预处理语句
│  │  │  │  ├─ 'H' (PqMsg_CopyResponse) - 复制数据
│  │  │  │  ├─ 'd' (PqMsg_CopyData) - 复制数据
│  │  │  │  ├─ 'c' (PqMsg_CopyDone) - 复制完成
│  │  │  │  ├─ 'X' (PqMsg_Terminate) - 断开连接
│  │  │  │  ├─ 'F' (PqMsg_FunctionCall) - 快速路径函数
│  │  │  │  └─ 'f' (PqMsg_CopyFail) - 复制错误
│  │  │  ├─ pq_getmessage() - 根据长度读取消息体
│  │  │  └─ pq_endmsgread() - 标记消息结束
│  │  │
│  │  └─ InteractiveBackend(inBuf) [如果是交互式]
│  │     ├─ 打印"backend> "提示符
│  │     ├─ 从stdin读取直到换行
│  │     └─ 返回PqMsg_Query
│  │
│  └─ 根据消息类型处理命令
```

### 3.2 查询解析器阶段（SQL → AST）

解析器将原始SQL文本转换为抽象语法树：

```
查询解析流程：

输入：SQL文本 "SELECT * FROM users WHERE id = 1;"

├─ pg_parse_query(sourceText) [tcop/postgres.c]
│  └─ raw_parser(sourceText) [parser/parser.c]
│     ├─ lex_scan_setup() - 初始化词法分析器
│     ├─ base_yyparse() - Bison解析器主函数
│     │  ├─ 词法分析器标记输入：
│     │  │  ├─ SELECT标记
│     │  │  ├─ *（星号）
│     │  │  ├─ FROM
│     │  │  ├─ 标识符"users"
│     │  │  ├─ WHERE
│     │  │  ├─ 标识符"id"
│     │  │  ├─ =运算符
│     │  │  └─ 数字常量1
│     │  │
│     │  └─ 解析器递归构建AST：
│     │     SelectStmt
│     │     ├─ targetList: [ResTarget(val: A_Star)]
│     │     ├─ fromClause: [RangeVar(relname: "users")]
│     │     └─ whereClause: A_Expr(
│     │        ├─ name: "="
│     │        ├─ lexpr: ColumnRef(name: "id")
│     │        └─ rexpr: A_Const(val: 1)
│     │        )
│     └─ lex_scan_cleanup()
│
└─ 返回：RawStmt
   ├─ stmt: SelectStmt（树）
   └─ stmt_location, stmt_len（文本位置）
```

**解析器数据结构：**

解析器操作以下关键节点类型：
- `SelectStmt` - SELECT语句
- `InsertStmt` - INSERT语句
- `UpdateStmt` - UPDATE语句
- `DeleteStmt` - DELETE语句
- `FuncCall` - 函数调用
- `Expr`及相关表达式节点
- `RangeVar` - 表引用

### 3.3 查询分析器/重写器阶段（语义分析）

分析器将原始解析树转换为Query对象并处理视图/规则：

```
查询分析和重写：

原始AST → parse_analyze_fixedparams() [parser/analyze.c]
│
├─ transformTopLevelStmt(pstate, RawStmt)
│  └─ transformStmt(pstate, stmt_node)
│     ├─ transformSelectStmt() [对于SELECT]
│     │  ├─ transformFromClause() - 处理FROM子句
│     │  │  ├─ 解析关系名称
│     │  │  ├─ 检查表/模式权限（ACLCHECK_SELECT）
│     │  │  ├─ 加载关系定义
│     │  │  ├─ 将RTE（RangeTblEntry）添加到pstate->p_rtable
│     │  │  └─ 处理JOIN
│     │  │
│     │  ├─ transformTargetList() - 处理SELECT列表
│     │  │  ├─ 解析列引用
│     │  │  ├─ 展开*为实际列
│     │  │  ├─ 类型强制表达式
│     │  │  └─ 为每个输出列创建ResTarget
│     │  │
│     │  ├─ transformWhereClause() - 处理WHERE
│     │  │  ├─ 递归transformExpr()
│     │  │  ├─ 谓词的类型强制
│     │  │  └─ 函数名称解析
│     │  │
│     │  ├─ transformGroupByClause() - 处理GROUP BY
│     │  ├─ transformHavingClause() - 处理HAVING
│     │  ├─ transformSortClause() - 处理ORDER BY
│     │  ├─ transformLimitClause() - 处理LIMIT
│     │  └─ 返回Query节点
│     │
│     ├─ transformInsertStmt() [对于INSERT]
│     │  ├─ 解析目标表
│     │  ├─ 检查INSERT权限
│     │  ├─ 将列列表与值匹配
│     │  ├─ 为缺失列应用默认值
│     │  ├─ 类型强制值
│     │  └─ 处理ON CONFLICT子句
│     │
│     ├─ transformUpdateStmt() [对于UPDATE]
│     │  ├─ 解析目标表
│     │  ├─ 检查UPDATE权限
│     │  ├─ 解析SET子句中的列引用
│     │  ├─ 新值的类型强制
│     │  ├─ 处理WHERE子句
│     │  └─ 处理FROM子句
│     │
│     └─ transformDeleteStmt() [对于DELETE]
│        ├─ 解析目标表
│        ├─ 检查DELETE权限
│        ├─ 处理WHERE子句
│        └─ 处理USING子句
│
├─ 处理视图规则（rewrite.c）
│  ├─ 如果表是视图：
│  │  ├─ 查找操作的规则
│  │  ├─ 应用查询重写规则
│  │  │  ├─ 插入规则
│  │  │  ├─ 更新规则
│  │  │  ├─ 删除规则
│  │  │  └─ 选择规则
│  │  └─ 用规则查询替换原始查询
│  │
│  └─ 如果表有规则系统：
│     └─ 应用适当的规则
│
└─ 返回：Query（已分析的查询树）
   ├─ commandType（CMD_SELECT、CMD_INSERT等）
   ├─ rtable（范围表 - 关系列表）
   ├─ targetList（输出列）
   ├─ jointree（WHERE和FROM组合）
   ├─ groupClause、havingClause、sortClause
   └─ 其他元数据（需要的权限检查等）
```

### 3.4 查询优化器阶段（AST → 执行计划）

优化器将Query转换为优化的执行计划：

```
查询规划：

Query → planner(Query *parse, const char *query_string, ...)
│                                         [optimizer/planner.c]
│
├─ standard_planner(root, parse, ...)
│  │
│  ├─ PlannerGlobal结构创建
│  │  ├─ 分配PlannerGlobal
│  │  ├─ 初始化CTE
│  │  └─ 设置并行信息
│  │
│  ├─ relation_parse_tree_to_relids(parse) - 识别关系
│  │  ├─ 构建关系OID列表
│  │  └─ 初始化关系统计信息
│  │
│  ├─ PlannerInfo（root）创建
│  │  ├─ 为每个基本关系初始化RelOptInfo
│  │  ├─ 计算表统计信息（行数、页数）
│  │  ├─ 计算索引成本
│  │  └─ 为连接条件构建等价类
│  │
│  ├─ 预处理阶段
│  │  ├─ preprocess_expression() - 简化表达式
│  │  ├─ preprocess_qual_conditions() - 处理WHERE
│  │  ├─ 为谓词构建EquivalenceClass
│  │  └─ 识别连接顺序依赖关系
│  │
│  ├─ 计划生成阶段
│  │  ├─ 对于每个命令类型（SELECT、INSERT等）：
│  │  │
│  │  ├─ SELECT规划：
│  │  │  ├─ join_search_one_level() - 构建连接树
│  │  │  │  ├─ 生成所有可能的连接顺序
│  │  │  │  ├─ 为每个连接路径计算成本：
│  │  │  │  │  ├─ SeqScan成本 = 页数 / seq_page_cost
│  │  │  │  │  ├─ IndexScan成本 = 随机I/O成本 + CPU
│  │  │  │  │  ├─ BitmapScan成本 = 索引 + 堆扫描
│  │  │  │  │  ├─ NestedLoop成本 = outer_rows * inner_cost
│  │  │  │  │  ├─ HashJoin成本 = build_hash + probe
│  │  │  │  │  └─ MergeJoin成本 = sort + merge
│  │  │  │  ├─ 修剪高成本路径（keep_useful_pathlist）
│  │  │  │  └─ 为每个关系集返回最佳路径
│  │  │  │
│  │  │  ├─ 最终路径选择
│  │  │  │  ├─ 为以下内容选择最佳路径：
│  │  │  │  │  ├─ 返回的行（用于LIMIT）
│  │  │  │  │  ├─ ORDER BY要求
│  │  │  │  │  └─ GROUP BY要求
│  │  │  │  ├─ Create_plan() - 将路径转换为计划节点
│  │  │  │  └─ 如果需要附加物理排序节点
│  │  │  │
│  │  │  └─ 规划后步骤：
│  │  │     ├─ 为子查询添加子计划
│  │  │     ├─ 添加CTE
│  │  │     ├─ 为不相关子查询添加initplans
│  │  │     └─ 标记可并行化节点
│  │  │
│  │  ├─ INSERT规划：
│  │  │  ├─ 规划SELECT（如果是INSERT ... SELECT）
│  │  │  ├─ 在顶部添加ModifyTable节点
│  │  │  └─ 检查ON CONFLICT逻辑
│  │  │
│  │  ├─ UPDATE规划：
│  │  │  ├─ 使用WHERE规划目标表的扫描
│  │  │  ├─ 添加ModifyTable节点
│  │  │  ├─ 设置更新表达式和targetlist
│  │  │  └─ 检查触发器要求
│  │  │
│  │  └─ DELETE规划：
│  │     ├─ 使用WHERE规划目标表的扫描
│  │     ├─ 添加ModifyTable节点
│  │     └─ 检查触发器要求
│  │
│  ├─ 计划最终化
│  │  ├─ create_plan() - 构建最终计划树
│  │  ├─ 优化子计划
│  │  ├─ 传播约束
│  │  ├─ 估算计划成本
│  │  └─ 计划并行化选项
│  │
│  └─ 返回PlannedStmt
│     ├─ commandType
│     ├─ planTree（计划树的根）
│     ├─ rtable（范围表）
│     ├─ result_relations
│     ├─ subplans
│     ├─ initPlan
│     ├─ targetList
│     └─ 成本估算
│
└─ 输出：PlannedStmt（执行计划）

SELECT * FROM users WHERE id = 1的示例计划树：

PlannedStmt
└─ planTree: SeqScan
   ├─ scanRelationOid: users表OID
   ├─ filter: id = 1（在执行期间应用）
   └─ targetList: [id, name, email, ...]
```

### 3.5 执行器阶段（计划执行）

执行器执行计划树以检索/修改数据：

```
查询执行：

ProcessQuery(query_desc) [tcop/postgres.c]
│
├─ CreateQueryDesc() - 之前已完成
│  ├─ query_desc.plannedstmt = plan
│  ├─ query_desc.snapshot = active_snapshot
│  └─ query_desc.sourceText = query_string
│
└─ PortalStart/PortalRun (portal.c)
   │
   ├─ ExecutorStart(queryDesc, options) [executor/execMain.c]
   │  │
   │  ├─ CreateExecutorState()
   │  │  ├─ 创建EState（执行器状态）结构
   │  │  ├─ es_query_cxt - 查询内存上下文
   │  │  ├─ es_epq_active - 急切计划队列状态
   │  │  ├─ 初始化结果关系（用于UPDATE/INSERT/DELETE）
   │  │  └─ 设置触发器上下文
   │  │
   │  ├─ InitPlan(queryDesc, eflags)
   │  │  ├─ ExecInitNode(planTree, estate)
   │  │  │  ├─ 递归自下而上初始化计划节点
   │  │  │  │
   │  │  │  ├─ 对于SeqScan：
   │  │  │  │  ├─ 使用关系管理器打开关系
   │  │  │  │  ├─ 初始化表扫描描述符
   │  │  │  │  │  └─ table_beginscan() [tableam.h]
   │  │  │  │  ├─ 分配结果槽
   │  │  │  │  └─ 返回SeqScanState
   │  │  │  │
   │  │  │  ├─ 对于IndexScan：
   │  │  │  │  ├─ 打开基本关系
   │  │  │  │  ├─ 打开索引
   │  │  │  │  ├─ 从计划构建索引扫描键
   │  │  │  │  ├─ 初始化索引扫描描述符
   │  │  │  │  │  └─ index_beginscan() [indexam.h]
   │  │  │  │  └─ 分配结果槽
   │  │  │  │
   │  │  │  ├─ 对于NestedLoopJoin：
   │  │  │  │  ├─ 初始化外部计划
   │  │  │  │  ├─ 初始化内部计划
   │  │  │  │  ├─ 初始化连接过滤器
   │  │  │  │  └─ 分配结果槽
   │  │  │  │
   │  │  │  ├─ 对于HashJoin：
   │  │  │  │  ├─ 初始化外部计划
   │  │  │  │  ├─ 初始化内部计划
   │  │  │  │  ├─ ExecHashTableCreate() - 创建哈希表
   │  │  │  │  ├─ ExecHashTableBuild() - 从内部构建哈希表
   │  │  │  │  └─ 分配结果槽
   │  │  │  │
   │  │  │  ├─ 对于Aggregate：
   │  │  │  │  ├─ 初始化聚合状态
   │  │  │  │  ├─ 初始化聚合函数
   │  │  │  │  ├─ 创建聚合缓冲区
   │  │  │  │  └─ 如果有GROUP BY则初始化分组聚合状态
   │  │  │  │
   │  │  │  ├─ 对于Sort：
   │  │  │  │  ├─ 初始化子节点
   │  │  │  │  ├─ 创建排序描述符
   │  │  │  │  └─ 分配排序缓冲区
   │  │  │  │
   │  │  │  ├─ 对于Limit：
   │  │  │  │  ├─ 初始化子节点
   │  │  │  │  └─ 初始化count/offset
   │  │  │  │
   │  │  │  ├─ 对于ModifyTable（INSERT/UPDATE/DELETE）：
   │  │  │  │  ├─ 初始化目标关系
   │  │  │  │  ├─ 检查权限
   │  │  │  │  ├─ 初始化结果关系
   │  │  │  │  ├─ 设置触发器上下文
   │  │  │  │  ├─ 初始化子扫描计划
   │  │  │  │  └─ 设置约束检查
   │  │  │  │
   │  │  │  └─ [对所有计划节点类型递归]
   │  │  │
   │  │  └─ 为UPDATE/INSERT/DELETE设置后触发上下文
   │  │
   │  ├─ 为事务隔离注册快照
   │  ├─ 如果需要设置查询检测
   │  └─ 返回（queryDesc现在已准备好执行）
   │
   ├─ ExecutorRun(queryDesc, direction, count) [executor/execMain.c]
   │  │
   │  ├─ standard_ExecutorRun(queryDesc, direction, count)
   │  │  │
   │  │  └─ ExecutePlan(queryDesc, ...)
   │  │     │
   │  │     └─ 循环：从计划获取元组
   │  │        │
   │  │        ├─ ExecProcNode(planstate) - 从计划获取下一个元组
   │  │        │  ├─ 情况SeqScan：
   │  │        │  │  ├─ table_scan_getnextslot() [tableam.h]
   │  │        │  │  │  ├─ ReadBuffer(heaprel, pageno)
   │  │        │  │  │  │  └─ 将页面加载到缓冲区缓存 [storage/bufmgr.c]
   │  │        │  │  │  │     ├─ 在缓冲区哈希表中查找
   │  │        │  │  │  │     ├─ 如果未找到，驱逐LRU页面
   │  │        │  │  │  │     ├─ 从磁盘读取
   │  │        │  │  │  │     └─ Pin缓冲区（增加引用计数）
   │  │        │  │  │  ├─ 检查元组可见性
   │  │        │  │  │  │  └─ HeapTupleSatisfiesMVCC(tuple, snapshot, buffer)
   │  │        │  │  │  │     ├─ 检查XMIN（插入事务）
   │  │        │  │  │  │     ├─ 检查XMAX（删除事务）
   │  │        │  │  │  │     ├─ 检查XID是否在活动快照中
   │  │        │  │  │  │     └─ 如果可见返回true，否则返回false
   │  │        │  │  │  ├─ ReleaseBuffer()
   │  │        │  │  │  └─ 如果可见，填充槽并返回
   │  │        │  │  └─ 继续直到扫描所有元组
   │  │        │  │
   │  │        │  ├─ 情况IndexScan：
   │  │        │  │  ├─ index_getnext() [indexam.h]
   │  │        │  │  │  ├─ 扫描索引查找匹配元组
   │  │        │  │  │  ├─ 从索引返回TID
   │  │        │  │  │  └─ 计数为迭代
   │  │        │  │  ├─ heap_fetch() [heapam.h]
   │  │        │  │  │  ├─ ReadBuffer(heap_rel, block_from_tid)
   │  │        │  │  │  ├─ OffsetNumber offset = ItemPointerGetOffsetNumber(tid)
   │  │        │  │  │  ├─ 从页面的offset获取元组
   │  │        │  │  │  └─ 检查可见性
   │  │        │  │  └─ 用元组填充槽
   │  │        │  │
   │  │        │  ├─ 情况NestedLoopJoin：
   │  │        │  │  ├─ 从外部计划获取元组
   │  │        │  │  ├─ 循环：与内部计划元组匹配
   │  │        │  │  │  ├─ 重置内部扫描
   │  │        │  │  │  ├─ 从内部获取第一个元组
   │  │        │  │  │  ├─ 循环：检查每个内部元组的连接条件
   │  │        │  │  │  │  ├─ ExecQual(join_filter, context)
   │  │        │  │  │  │  │  ├─ 评估表达式
   │  │        │  │  │  │  │  ├─ 对于每个列访问：
   │  │        │  │  │  │  │  │  ├─ 槽访问（已在内存中）
   │  │        │  │  │  │  │  │  ├─ 如果需要进行类型强制
   │  │        │  │  │  │  │  │  └─ 运算符评估
   │  │        │  │  │  │  │  └─ 返回布尔结果
   │  │        │  │  │  │  ├─ 如果匹配，创建结果元组并返回
   │  │        │  │  │  │  └─ 否则继续
   │  │        │  │  │  └─ 获取下一个内部元组
   │  │        │  │  └─ 当内部耗尽时，获取下一个外部元组
   │  │        │  │
   │  │        │  ├─ 情况HashJoin：
   │  │        │  │  ├─ 用外部元组探测哈希表
   │  │        │  │  ├─ 对于每个匹配的哈希桶条目：
   │  │        │  │  │  ├─ 检查连接条件
   │  │        │  │  │  └─ 如果匹配，返回结果元组
   │  │        │  │  └─ 否则获取下一个外部元组
   │  │        │  │
   │  │        │  ├─ 情况Aggregate：
   │  │        │  │  ├─ 累积聚合值
   │  │        │  │  ├─ 对于每个输入行：
   │  │        │  │  │  ├─ 调用聚合推进函数
   │  │        │  │  │  │  └─ 例如，aggregate_sum(state, value)
   │  │        │  │  │  └─ 更新状态
   │  │        │  │  ├─ 如果有GROUP BY：
   │  │        │  │  │  ├─ 使用聚合哈希表
   │  │        │  │  │  ├─ 键 = 组值
   │  │        │  │  │  └─ 值 = 聚合状态
   │  │        │  │  └─ 完成时返回最终聚合
   │  │        │  │
   │  │        │  ├─ 情况Sort：
   │  │        │  │  ├─ 获取所有输入元组
   │  │        │  │  ├─ 使用qsort或tuplesort排序
   │  │        │  │  │  └─ 根据ORDER BY使用比较函数
   │  │        │  │  └─ 按排序顺序返回元组
   │  │        │  │
   │  │        │  └─ [对所有节点类型以此类推]
   │  │        │
   │  │        ├─ ExecProject(targetlist, slot)
   │  │        │  ├─ 从计划元组创建输出元组
   │  │        │  ├─ 评估计算列
   │  │        │  ├─ 应用投影和强制
   │  │        │  └─ 返回结果槽
   │  │        │
   │  │        ├─ 通过DestReceiver将元组发送给客户端
   │  │        │  ├─ receiveSlot(slot)
   │  │        │  │  ├─ 转换为线格式（文本/二进制）
   │  │        │  │  ├─ 发送'D'（DataRow）消息
   │  │        │  │  │  ├─ 消息类型：D
   │  │        │  │  │  ├─ 消息长度
   │  │        │  │  │  ├─ 列数
   │  │        │  │  │  └─ 对于每列：
   │  │        │  │  │     ├─ 列长度（或-1表示NULL）
   │  │        │  │  │     └─ 列值字节
   │  │        │  │  └─ 在发送缓冲区中排队
   │  │        │  │
   │  │        │  └─ pq_flush() - 缓冲区满时刷新到网络
   │  │        │
   │  │        ├─ 检查中断信号
   │  │        │  ├─ CHECK_FOR_INTERRUPTS()
   │  │        │  ├─ 处理SIGTERM、SIGINT
   │  │        │  ├─ 如果请求取消查询
   │  │        │  └─ 回滚和清理
   │  │        │
   │  │        └─ 如果达到计数或扫描结束，退出循环
   │  │
   │  └─ 返回（已获取所有请求的元组）
   │
   ├─ ExecutorFinish(queryDesc) [executor/execMain.c]
   │  ├─ 对于没有GROUP BY的聚合：
   │  │  ├─ 最终化聚合函数
   │  │  │  └─ aggregate_final_func(state) → 结果值
   │  │  └─ 返回最终聚合行
   │  │
   │  ├─ 对于UPDATE/INSERT/DELETE：
   │  │  ├─ 执行延迟的before/after触发器
   │  │  │  └─ 对于每个延迟触发器：
   │  │  │     ├─ 调用触发器函数
   │  │  │     └─ 执行触发器SQL
   │  │  │
   │  │  ├─ 刷新约束检查
   │  │  └─ 执行AFTER触发器
   │  │
   │  └─ 清理执行器状态
   │
   └─ ExecutorEnd(queryDesc) [executor/execMain.c]
      ├─ ExecEndPlan(planstate, estate)
      │  └─ 递归关闭所有计划节点
      │     ├─ 关闭扫描关系
      │     ├─ 关闭索引
      │     ├─ 释放哈希表
      │     ├─ 释放排序缓冲区
      │     └─ 释放分配的结构
      │
      ├─ 释放EState
      ├─ 注销快照
      └─ 清理执行器内存
```

### 3.6 结果返回给客户端

```
向客户端发送结果：

对于SELECT查询：
├─ 向客户端发送ReadyForQuery消息
│  ├─ 'Z'消息类型
│  ├─ 状态：'I'（空闲）、'T'（事务中）、'E'（错误）
│  └─ 在所有行之后发送
│
└─ 结果行的格式：
   ├─ RowDescription消息（首先发送）：
   │  ├─ 消息类型：T
   │  ├─ 字段计数
   │  └─ 对于每个字段：
   │     ├─ 列名
   │     ├─ 表OID
   │     ├─ 列号
   │     ├─ 类型OID
   │     ├─ 类型大小
   │     ├─ 类型修饰符
   │     └─ 格式代码（0=文本，1=二进制）
   │
   └─ DataRow消息（对于每个结果行）：
      ├─ 消息类型：D
      ├─ 字段计数
      └─ 对于每列：
         ├─ 长度（或-1表示NULL）
         └─ 输出格式中的值（文本或二进制）

对于UPDATE/INSERT/DELETE：
├─ 返回行计数："INSERT 0 5"（oid和计数）
└─ 如果有RETURNING子句：
   ├─ 发送RowDescription
   ├─ 为每个返回的行发送DataRow
   └─ 发送CommandComplete
```

---

## 4. 事务处理

### 4.1 事务启动/提交/回滚

位于`src/backend/access/transam/xact.c`：

```
事务生命周期：

start_xact_command() [tcop/postgres.c]
├─ 如果不在事务中：
│  ├─ BeginTransactionBlock()
│  │  ├─ 设置blockState = TBLOCK_INPROGRESS
│  │  ├─ 设置TransState = TRANS_START
│  │  └─ 在第一个查询之前不做实际工作
│  │
│  └─ StartTransactionCommand() [xact.c]
│     ├─ start_transaction() [xact.c]
│     │  ├─ 如果需要分配TransactionId
│     │  │  └─ GetNewTransactionId(isSubTransaction)
│     │  │     ├─ 从共享内存读取xidgen变量
│     │  │     ├─ 分配下一个XID
│     │  │     └─ 更新共享xidgen
│     │  │
│     │  ├─ 记录为进行中
│     │  │  └─ PGPROC->xid = MyTransactionId
│     │  │
│     │  ├─ 初始化事务快照
│     │  │  └─ GetSnapshotData(&SnapshotData)
│     │  │     ├─ 从PGPROC数组读取活动XID
│     │  │     ├─ 查找全局xmin（最旧的活动XID）
│     │  │     ├─ 查找xmax（要分配的下一个）
│     │  │     └─ 标记进行中的事务
│     │  │
│     │  ├─ 初始化事务上下文
│     │  │  └─ 创建TransactionStateData
│     │  │     ├─ fullTransactionId = {epoch, xid}
│     │  │     ├─ subTransactionId = 1
│     │  │     ├─ state = TRANS_INPROGRESS
│     │  │     └─ blockState = TBLOCK_INPROGRESS
│     │  │
│     │  └─ 如果需要在WAL中记录事务启动
│     │     └─ LogCurrentTransactionState()
│     │
│     └─ 向其他后端发送InvalidationMessage
│        └─ Signal()其他后端关于新事务
│
├─ 在事务中：设置xact_started = true
└─ 为此命令设置CommandId
   └─ GetCurrentCommandId(canAssign)


finish_xact_command() [tcop/postgres.c]
├─ CommitTransactionCommand() [xact.c]
│  └─ CommitTransaction()
│     ├─ 预提交阶段：
│     │  ├─ 运行BEFORE COMMIT触发器
│     │  └─ 刷新PREPARE语句
│     │
│     ├─ 提交阶段：
│     │  ├─ 记录WAL提交记录
│     │  │  └─ LogXactAbortRecord()
│     │  │     ├─ 创建XLOG_XACT_COMMIT记录
│     │  │     ├─ 包括时间戳
│     │  │     ├─ 包括事务XID
│     │  │     └─ 包括提交信息
│     │  │
│     │  ├─ 调用wal_writer刷新WAL
│     │  │  └─ XLogFlush(currentLogEndPos)
│     │  │     ├─ 将缓冲的WAL写入磁盘
│     │  │     ├─ 如果synchronous_commit = on则fsync()
│     │  │     └─ 更新LSN（日志序列号）
│     │  │
│     │  ├─ 将事务标记为已提交
│     │  │  └─ TransactionIdCommitTree(xid, nchildren, children)
│     │  │     ├─ 写入pg_xact
│     │  │     │  └─ TransactionIdSetStatus(xid, TRANSACTION_COMMITTED)
│     │  │     └─ 写入子事务XID
│     │  │        └─ TransactionIdSetStatus(child_xid, TRANSACTION_COMMITTED)
│     │  │
│     │  ├─ 从PGPROC活动列表中删除
│     │  │  └─ PGPROC->xid = InvalidTransactionId
│     │  │
│     │  └─ 如果需要刷新缓存
│     │
│     ├─ 后提交阶段：
│     │  ├─ 执行AFTER COMMIT触发器
│     │  ├─ 使关系缓存条目失效
│     │  │  └─ CardinalitySet() - 更新规划器基数信息
│     │  ├─ 释放锁
│     │  │  └─ LockReleaseAll(DEFAULT_LOCKMETHOD)
│     │  ├─ 关闭门户
│     │  ├─ 释放事务内存
│     │  └─ 向复制发送提交消息
│     │
│     └─ 更新事务计数器
│        └─ 增加PostgreSQL的内部统计信息
│
└─ 返回空闲状态

---

ROLLBACK过程：

RollbackTransactionCommand()
└─ RollbackTransaction()
   ├─ 预中止阶段：
   │  ├─ 在WAL中记录中止记录
   │  │  └─ XactLogAbortRecord()
   │  │     ├─ XLOG_XACT_ABORT记录
   │  │     └─ 包括事务XID
   │  │
   │  ├─ 将事务标记为已中止
   │  │  └─ TransactionIdAbortTree(xid, nchildren)
   │  │     └─ TransactionIdSetStatus(xid, TRANSACTION_ABORTED)
   │  │
   │  └─ 刷新WAL
   │     └─ XLogFlush()
   │
   ├─ 中止阶段：
   │  ├─ 执行BEFORE ROLLBACK触发器
   │  ├─ 回滚保存点
   │  ├─ 撤销DML操作：
   │  │  ├─ 对于INSERT：删除插入的行
   │  │  ├─ 对于UPDATE：恢复以前的版本
   │  │  └─ 对于DELETE：恢复删除的行
   │  │
   │  ├─ 释放锁
   │  │  └─ LockReleaseAll()
   │  │     ├─ 首先是本地锁
   │  │     ├─ 然后是共享锁
   │  │     └─ 通知等待的进程
   │  │
   │  └─ 清理状态
   │
   ├─ 后中止阶段：
   │  ├─ 执行AFTER ROLLBACK触发器
   │  ├─ 重置事务状态
   │  ├─ 使快照失效
   │  └─ 释放事务内存
   │
   └─ 返回空闲状态
```

### 4.2 子事务（SAVEPOINT）

```
SAVEPOINT管理：

SAVEPOINT sp_name
└─ BeginInternalSubTransaction(NULL)
   ├─ 创建新的TransactionStateData
   │  └─ parent = 以前的TransactionState
   │
   ├─ 分配SubTransactionId
   │  └─ ++currentSubTransactionId
   │
   ├─ 创建子事务上下文
   │  └─ 为子事务创建内存上下文
   │
   ├─ 在WAL中记录（可选）
   │  └─ XLogStartSubTransaction(subxid)
   │
   └─ 推送到事务状态堆栈

RELEASE SAVEPOINT sp_name
└─ ReleaseCurrentSubTransaction()
   ├─ 验证保存点仍然存在
   ├─ 提交子事务更改
   │  └─ 将更改合并到父事务
   ├─ 在WAL中记录释放
   └─ 从事务状态堆栈弹出

ROLLBACK TO SAVEPOINT sp_name
└─ RollbackAndReleaseCurrentSubTransaction()
   ├─ 中止子事务操作
   ├─ 撤销DML更改
   ├─ 从保存点之前恢复状态
   ├─ 在WAL中记录中止
   └─ 从事务状态堆栈弹出
```

### 4.3 MVCC实现

PostgreSQL中的MVCC（多版本并发控制）允许读者和写者共存：

```
MVCC核心概念：

事务ID (XID)：
├─ 在事务启动时分配的32位标识符
├─ 在服务器生命周期中递增
├─ 存储在元组头中：
│  ├─ t_infomask：提交状态位
│  ├─ t_xmin：插入事务ID
│  ├─ t_xmax：删除/更新事务ID
│  └─ t_cid：命令ID
│
└─ Epoch：
   ├─ 32位epoch计数器
   ├─ 跟踪XID环绕
   └─ 与XID结合用于完整事务ID


元组可见性规则：

HeapTupleSatisfiesMVCC(tuple, snapshot)
├─ 检查插入事务 (t_xmin)：
│  ├─ 如果 t_xmin == my_xid：
│  │  └─ 我插入的 → 可见
│  ├─ 否则如果 t_xmin 在 snapshot.xip（活动列表）中：
│  │  └─ 插入进行中 → 不可见
│  ├─ 否则如果 t_xmin < snapshot.xmin：
│  │  └─ 插入肯定已提交 → 检查删除
│  ├─ 否则如果 t_xmin >= snapshot.xmax：
│  │  └─ 插入肯定未启动 → 不可见
│  └─ 否则（t_xmin在xmin和xmax之间）：
│     └─ 检查pg_xact状态
│
└─ 检查删除事务 (t_xmax)：
   ├─ 如果 t_xmax == InvalidXid：
   │  └─ 未删除 → 可见
   ├─ 否则如果 t_xmax == my_xid：
   │  └─ 我删除的 → 不可见
   ├─ 否则如果 t_xmax 在 snapshot.xip 中：
   │  └─ 删除进行中 → 可见
   ├─ 否则如果 t_xmax < snapshot.xmin：
   │  └─ 删除肯定已提交 → 不可见
   ├─ 否则如果 t_xmax >= snapshot.xmax：
   │  └─ 删除肯定未启动 → 可见
   └─ 否则（t_xmax在xmin和xmax之间）：
      └─ 检查pg_xact状态


快照结构：

Snapshot {
    SnapshotType: SNAPSHOT_MVCC, SNAPSHOT_SELF, SNAPSHOT_DIRTY, SNAPSHOT_TOAST
    xmin: transaction.in_progress中最小的XID
    xmax: 看到的最高XID + 1
    xip[]: 活动事务XID数组
    xcnt: 活动XID数量
    xip_base: 用于压缩的基础XID（PG13+）
    subxip[]: 活动子事务XID数组（如果有）
    subxcnt: 活动子xid数量
    suboverflowed: 子xid列表是否不完整
    takenDuringRecovery: 在崩溃恢复期间获取
}

快照创建：

GetSnapshotData() [src/backend/access/transam/procarray.c]
├─ 锁定PGPROC数组
├─ 从PGPROC读取所有活动事务状态
├─ 查找：
│  ├─ xmin = 最小活动XID
│  ├─ xmax = 最高分配的XID + 1
│  └─ xip[] = 所有活动XID的数组
├─ 对xip[]排序以进行二分搜索
├─ 解锁PGPROC数组
└─ 返回Snapshot

示例快照状态：
获取快照之前：
  PGPROC[0]: xid=100, running
  PGPROC[1]: xid=102, running
  PGPROC[2]: xid=103, running (subtransaction)
  LastAssignedXid = 103

GetSnapshotData()之后的快照：
  xmin = 100（最旧活动）
  xmax = 104（要分配的下一个）
  xip[] = {100, 102, 103}
  xcnt = 3

现在对于元组可见性：
  xmin=95  → 可见（比xmin旧）
  xmin=100 → 不可见（在xip中）
  xmin=101 → 可见（不在xip中，>= xmin）
  xmin=103 → 不可见（在xip中）
  xmin=104 → 不可见（>= xmax）
```

### 4.4 WAL（预写日志）

```
WAL结构：

WAL文件：
├─ 位置：$PGDATA/pg_wal/
├─ 文件名：XXXXXXXXXXXXXXXX（16个十六进制数字 = 8字节XLogFileId + 8字节XLogRecPtr）
├─ 大小：默认16 MB（wal_segment_size）
└─ 组织：
   ├─ 页面0：文件头
   ├─ 页面1-N：WAL记录
   └─ 每个页面：8KB（XLOG_BLCKSZ）

WAL记录结构：

XLogRecord {
    xl_tot_len: 记录总长度
    xl_xid: 事务ID
    xl_prev: 指向前一条记录的指针
    xl_info: 记录类型（插入、删除等）
    xl_rmid: 资源管理器ID（堆、索引等）
    xl_data_len: 数据部分长度
    ...数据...
}

WAL记录类型（资源管理器）：
├─ RM_XACT_ID: 事务记录
│  ├─ XLOG_XACT_COMMIT
│  ├─ XLOG_XACT_ABORT
│  ├─ XLOG_XACT_PREPARE
│  └─ XLOG_XACT_ASSIGNMENT
├─ RM_HEAP_ID: 堆表记录
│  ├─ XLOG_HEAP_INSERT
│  ├─ XLOG_HEAP_UPDATE
│  ├─ XLOG_HEAP_DELETE
│  ├─ XLOG_HEAP_HOT_UPDATE
│  └─ XLOG_HEAP_TRUNCATE
├─ RM_BTREE_ID: B-tree索引记录
│  ├─ XLOG_BTREE_INSERT_LEAF
│  ├─ XLOG_BTREE_INSERT_UPPER
│  ├─ XLOG_BTREE_SPLIT_L
│  └─ 等
└─ [不同访问方法的其他资源管理器]

WAL插入和刷新：

插入操作：
1. LogNewPage()或LogRecordData()
   ├─ 获取WALInsertLock
   ├─ 写入共享内存中的WAL缓冲区
   ├─ 更新currentLogEndPos
   └─ 释放WALInsertLock

2. XLogFlush(requestPtr)
   ├─ 获取WALFlushLock
   ├─ 如果synchronous_commit：
   │  ├─ fsync() WAL文件到磁盘
   │  └─ 更新lastFlushedPtr
   └─ 释放WALFlushLock

同步提交级别：
├─ off: 缓冲区写入后立即返回
├─ local: 本地fsync()后返回
├─ remote_write: 副本写入磁盘后返回
├─ remote_apply: 副本应用后返回
└─ on: 与remote_apply相同（默认）

WAL格式：
页面头（32字节）| 记录 | 记录 | ... | 尾部（可选）

每个页面开始于：
├─ 魔数（XLOG_PAGE_MAGIC）
├─ 页面头信息
├─ 文件中第一页的XLogFileHeader
└─ 记录跟随
```

---

## 5. 存储访问流程

### 5.1 缓冲池交互

缓冲池是PostgreSQL的磁盘页面内存缓存：

```
缓冲池架构：

共享内存缓冲池：
├─ 分配：shared_buffers参数（默认128MB）
├─ 页面大小：8KB（BLCKSZ）
├─ 缓冲区数量：shared_buffers / 8192
│
├─ 数据结构：共享内存中的BufferDesc数组
│  ├─ 每个缓冲区一个BufferDesc
│  └─ 每个BufferDesc包含：
│     ├─ tag: RelFileLocator {spcOid, dbOid, relOid, segNo}
│     ├─ freeNext, freePrev: 空闲列表链接
│     ├─ data: 指向缓冲区页面数据的指针
│     ├─ refcount: Pin计数
│     ├─ usageCount: LRU使用计数器
│     ├─ flags: 脏、I/O进行中等
│     └─ spinlock: 保护头修改
│
├─ 缓冲区哈希表 (buf_table.c)
│  ├─ 将tag映射到BufferDesc的dynahash表
│  ├─ 分区以提高可扩展性
│  └─ 由BufMappingLock保护
│
└─ 空闲列表 (freelist.c)
   ├─ 未使用缓冲区列表
   ├─ 由buffer_strategy_lock保护
   └─ 用于缓冲区替换

缓冲区替换策略（时钟扫描）：

nextVictimBuffer → [B0] [B1] [B2] [B3] [B4] ...
                     ↑ (时钟指针循环通过)

算法：
1. 锁定buffer_strategy_lock
2. 在nextVictimBuffer处选择缓冲区
3. 推进nextVictimBuffer
4. 解锁buffer_strategy_lock
5. 如果缓冲区被pin或usageCount > 0：
   ├─ 减少usageCount
   └─ 转到步骤1（尝试下一个缓冲区）
6. 如果缓冲区脏：
   ├─ 写入磁盘（通常由后台写进程完成）
7. Pin缓冲区并使用
```

### 5.2 页面访问

```
ReadBuffer() [storage/bufmgr.c]
├─ 调用者提供：
│  ├─ rel: Relation对象
│  ├─ blockNum: 要读取的块号
│  └─ strategy: 缓冲区环策略（可选）
│
├─ ReadBufferExtended(rel, fork, blockNum, mode, strategy)
│  │
│  ├─ 计算RelFileLocator标签
│  │  └─ tag {spcOid, dbOid, relOid, segNo}
│  │
│  ├─ LockBufMappingPartition(tag)
│  │  └─ 为此标签的分区获取BufMappingLock
│  │
│  ├─ buf_table.c: BufTableLookup(tag)
│  │  ├─ 哈希表查找
│  │  └─ 检查页面是否已在缓存中
│  │
│  ├─ 如果找到：
│  │  ├─ PinBuffer(bufHdr)
│  │  │  ├─ 增加引用计数
│  │  │  ├─ 增加使用计数
│  │  │  └─ 更新LRU信息
│  │  ├─ UnlockBufMappingPartition()
│  │  └─ 返回缓冲区
│  │
│  └─ 如果未找到（缓存未命中）：
│     ├─ UnlockBufMappingPartition()
│     ├─ StrategyGetBuffer(strategy) - 获取牺牲缓冲区
│     │  ├─ 使用时钟扫描算法
│     │  ├─ 如果脏则驱逐（写入磁盘）
│     │  └─ 返回未使用的缓冲区
│     │
│     ├─ LockBufMappingPartition(tag) - 重新获取锁
│     ├─ BufTableInsert() - 添加到哈希表
│     │  └─ 更新缓冲区标签
│     │
│     ├─ UnlockBufMappingPartition()
│     ├─ ReadBuffer_Disk()
│     │  ├─ mdopen() - 打开关系fork
│     │  ├─ mdread() - 从磁盘读取块
│     │  │  └─ lseek() + read()系统调用
│     │  └─ 标记页面为已加载
│     │
│     └─ 返回缓冲区

ReleaseBuffer(buffer) [storage/bufmgr.c]
├─ 获取BufferDesc
├─ UnpinBuffer(bufHdr)
│  ├─ 减少引用计数
│  ├─ 如果引用计数 == 1并且有人等待清理锁：
│  │  └─ 通知条件变量
│  └─ 释放自旋锁
└─ 标记缓冲区为已释放

缓冲区访问流程：

缓冲区Pin（所有权）：
├─ 访问前必须pin缓冲区
├─ 可以在同一缓冲区上持有多个pin
├─ 不能跨事务边界持有pin
└─ Pin保护缓冲区免于驱逐

缓冲区Lock（内容访问）：
├─ BUFFER_LOCK_SHARE（读访问）
│  └─ 允许多个读者
├─ BUFFER_LOCK_EXCLUSIVE（写访问）
│  └─ 需要独占访问
└─ 在获取锁之前必须持有pin

典型访问模式：
1. ReadBuffer(rel, blockno) → pin缓冲区
2. LockBuffer(buf, BUFFER_LOCK_SHARE) → 锁定以供读取
3. 访问缓冲区数据：(HeapTupleHeader) BufferGetPage(buf)
4. UnlockBuffer(buf) → 解锁
5. 继续访问数据（仍持有pin）
6. ReleaseBuffer(buf) → unpin缓冲区

事务边界：
└─ 所有pin必须在事务结束时释放
   └─ 在EndTransaction中自动调用
```

### 5.3 锁获取

PostgreSQL中的锁分为多层：

```
锁层次结构：

1. 自旋锁（非常短期）
   ├─ 用于：缓冲区头、锁管理器分区
   ├─ 持有时间：仅几条指令
   ├─ 忙等待机制
   └─ 无死锁检测

2. 轻量级锁（LWLocks）
   ├─ 用于：共享内存结构保护
   ├─ 模式：共享、独占
   ├─ 模式：在信号量上阻塞
   └─ 无死锁检测

   常见LWLocks：
   ├─ BufMappingLock: 保护缓冲区映射哈希表
   ├─ BufFreelistLock: 保护空闲缓冲区列表
   ├─ LockMgrLock: 保护锁管理器结构
   ├─ WALInsertLock: 保护WAL插入
   ├─ WALFlushLock: 保护WAL刷新
   └─ [约100个其他LWLocks]

3. 重量级锁（常规锁）
   ├─ 用于：表/关系/元组锁定
   ├─ 类型：AccessShareLock、RowExclusiveLock、ExclusiveLock等
   ├─ 模式：显式冲突检查
   └─ 功能：死锁检测、自动释放

   锁类型：
   ├─ AccessShareLock: SELECT（最低）
   ├─ RowShareLock: SELECT FOR SHARE
   ├─ RowExclusiveLock: INSERT、UPDATE、DELETE
   ├─ ShareUpdateExclusiveLock: VACUUM、ANALYZE
   ├─ ShareLock: CREATE INDEX
   ├─ ShareRowExclusiveLock:
   ├─ ExclusiveLock: LOCK TABLE
   └─ AccessExclusiveLock: ALTER TABLE、DROP（最高）

关系锁获取：

LockRelationOid(relOid, lockMode)
├─ 在LOCK哈希表中查找锁
├─ 检查与现有锁的冲突
├─ 如果无冲突：
│  ├─ 立即授予锁
│  └─ 在PROCLOCK结构中记录
├─ 如果有冲突：
│  ├─ 添加到等待队列
│  ├─ 设置死锁检测警报
│  ├─ 在信号量上休眠
│  └─ 授予锁或检测到死锁时唤醒
└─ 返回锁

锁冲突矩阵：
[这里会有一个详细的锁冲突矩阵表，但由于限制，这里简化表示]
```

### 5.4 元组可见性检查

元组可见性是MVCC的核心：

```
元组可见性检查流程：

HeapTupleSatisfiesMVCC(htup, snapshot, buffer)
├─ 获取元组头：t_data = htup->t_data
│  ├─ t_infomask: 状态位
│  │  ├─ HEAP_XMIN_COMMITTED: Xmin已提交
│  │  ├─ HEAP_XMIN_INVALID: Xmin无效
│  │  ├─ HEAP_XMAX_COMMITTED: Xmax已提交
│  │  ├─ HEAP_XMAX_INVALID: Xmax无效
│  │  ├─ HEAP_XMAX_IS_MULTI: Xmax是multixact
│  │  ├─ HEAP_XMAX_IS_LOCKED_ONLY: Xmax是锁，不是删除
│  │  └─ [其他标志]
│  ├─ t_xmin: 插入事务ID
│  └─ t_xmax: 删除事务ID或锁xid
│
├─ 检查插入状态：
│  ├─ 原始Xmin: HeapTupleHeaderGetRawXmin(tuple)
│  ├─ 步骤1：确定插入是否已提交
│  │  ├─ 如果设置了HEAP_XMIN_COMMITTED位：
│  │  │  └─ 是，已提交
│  │  ├─ 否则如果设置了HEAP_XMIN_INVALID位：
│  │  │  └─ 否，已中止
│  │  ├─ 否则如果Xmin == my_xid：
│  │  │  └─ 是，我仍在此事务中
│  │  ├─ 否则如果TransactionIdIsInProgress(Xmin)：
│  │  │  └─ 否，仍在进行中
│  │  ├─ 否则如果XidInMVCCSnapshot(Xmin, snapshot)：
│  │  │  └─ 否，获取快照时活动
│  │  ├─ 否则如果TransactionIdDidCommit(Xmin)：
│  │  │  ├─ 是，检查pg_xact状态
│  │  │  └─ 设置HEAP_XMIN_COMMITTED提示位
│  │  └─ 否则：
│  │     ├─ 否，假设已中止/崩溃
│  │     └─ 设置HEAP_XMIN_INVALID提示位
│  │
│  └─ 如果插入未提交：
│     └─ 返回：不可见
│
├─ 检查删除状态：
│  ├─ 原始Xmax: HeapTupleHeaderGetRawXmax(tuple)
│  ├─ 步骤1：确定元组是否已删除
│  │  ├─ 如果设置了HEAP_XMAX_INVALID位：
│  │  │  └─ 否，未删除
│  │  ├─ 否则如果设置了HEAP_XMAX_IS_LOCKED_ONLY：
│  │  │  └─ 否，仅锁定，未删除
│  │  ├─ 否则如果Xmax == my_xid：
│  │  │  └─ 是，我删除的
│  │  ├─ 否则如果TransactionIdIsInProgress(Xmax)：
│  │  │  └─ 否，删除进行中
│  │  ├─ 否则如果XidInMVCCSnapshot(Xmax, snapshot)：
│  │  │  └─ 否，获取快照时删除活动
│  │  ├─ 否则如果TransactionIdDidCommit(Xmax)：
│  │  │  ├─ 是，已提交删除
│  │  │  └─ 设置HEAP_XMAX_COMMITTED提示位
│  │  └─ 否则：
│  │     └─ 否，假设已中止/崩溃
│  │
│  └─ 如果已删除且删除已提交：
│     └─ 返回：不可见
│
└─ 返回：可见

提示位：
├─ 避免重复pg_xact查找的优化
├─ 一旦知道事务状态，在元组上设置提示位
├─ SetHintBits(tuple, buffer, infomask, xid)
│  ├─ 检查设置是否安全：
│  │  ├─ 对于提交位：检查WAL是否已刷新
│  │  └─ 对于中止位：总是安全的
│  ├─ tuple->t_infomask |= infomask
│  └─ MarkBufferDirtyHint(buffer, true)
└─ 下次访问元组时，位已设置

快照函数：
├─ XidInMVCCSnapshot(xid, snapshot)
│  ├─ 在snapshot.xip[]中二分搜索
│  └─ 如果xid在活动列表中返回true
│
├─ TransactionIdIsInProgress(xid)
│  ├─ 直接检查PGPROC数组
│  └─ 如果xid主动运行则为true
│
└─ TransactionIdDidCommit(xid)
   ├─ 检查pg_xact日志
   └─ 如果已提交则为true
```

---

## 6. 控制流程图（文本格式）

### 6.1 服务器启动序列图

```
┌─────────────────────────────────────────────────────────────────┐
│ PostgreSQL服务器启动流程                                          │
└─────────────────────────────────────────────────────────────────┘

         启动postgres
                │
                ▼
    ┌───────────────────────┐
    │ main() [main.c]       │
    │ - 检查root            │
    │ - 初始化内存          │
    │ - 初始化区域设置      │
    └───────────┬───────────┘
                │
                ▼
    ┌───────────────────────┐
    │ PostmasterMain()      │
    │ [postmaster.c]        │
    │ - 初始化全局变量      │
    │ - 加载配置            │
    │ - 创建共享内存        │
    │ - 设置信号            │
    └───────────┬───────────┘
                │
                ▼
    ┌───────────────────────┐
    │ ServerLoop()          │
    │ 主事件循环            │
    │ - 接受连接            │
    │ - 管理子进程          │
    │ - 状态机              │
    └───────────┬───────────┘
                │
     ┌──────────┼──────────┐
     │          │          │
     ▼          ▼          ▼
 启动进程   后台进程   连接处理
  ├─ 重做   ├─后台写   ├─认证
  ├─ 初始化 ├─检查点   ├─后端
  └─ 就绪   └─其他     └─循环
```

### 6.2 查询处理流水线

```
┌─────────────────────────────────────────────────────────────────┐
│ SQL查询处理流水线                                                 │
└─────────────────────────────────────────────────────────────────┘

输入：SELECT * FROM users WHERE age > 18;

    ┌──────────────────────────────────┐
    │ 解析阶段                          │
    │ 字符串 → 抽象语法树               │
    └────────────┬─────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │ 输出：RawStmt                             │
    │ ├─ SelectStmt节点                        │
    │ ├─ targetList（列）                      │
    │ ├─ fromClause（表）                      │
    │ └─ whereClause（谓词）                   │
    └────────────┬──────────────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │ 分析阶段                                  │
    │ AST → 语义分析 + 重写                     │
    └────────────┬──────────────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │ 输出：Query                               │
    │ ├─ commandType (CMD_SELECT)              │
    │ ├─ rtable（带OID的范围表）                │
    │ ├─ targetList（已解析的列）               │
    │ ├─ jointree（已处理的WHERE）              │
    │ └─ 需要的权限                             │
    └────────────┬──────────────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │ 规划阶段                                   │
    │ Query → 优化的执行计划                     │
    │ ├─ 基于成本的优化                          │
    │ ├─ 路径枚举                                │
    │ ├─ 连接排序                                │
    │ └─ 索引选择                                │
    └────────────┬──────────────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │ 输出：PlannedStmt                         │
    │ ├─ planTree（执行计划树）                  │
    │ ├─ relcache（关系信息）                    │
    │ └─ 成本估算                                │
    │                                            │
    │ 示例planTree：                             │
    │     Limit（如果有LIMIT子句）               │
    │       └─ Sort（如果有ORDER BY）           │
    │           └─ SeqScan on users             │
    │               └─ Filter (age > 18)        │
    └────────────┬──────────────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │ 执行阶段                                   │
    │ Plan → 结果                                │
    │ ├─ ExecutorStart()                        │
    │ ├─ ExecutorRun()                          │
    │ │  ├─ 打开表/索引                          │
    │ │  ├─ 扫描页面                             │
    │ │  ├─ 检查元组可见性（MVCC）               │
    │ │  ├─ 应用过滤器                           │
    │ │  ├─ 如果需要排序                         │
    │ │  └─ 返回行                               │
    │ ├─ ExecutorFinish()                       │
    │ └─ ExecutorEnd()                          │
    └────────────┬──────────────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │ 输出：结果行                               │
    │ ├─ RowDescription消息                     │
    │ ├─ DataRow消息                            │
    │ └─ CommandComplete消息                    │
    └──────────────────────────────────────────────┘
```

### 6.3 连接建立流程

```
┌──────────────────────────────────────────────────────┐
│ 客户端连接建立                                        │
└──────────────────────────────────────────────────────┘

客户端                          Postmaster

  │                                │
  ├─ TCP连接 ─────────────────────>│
  │                                │
  │                         Accept()调用
  │                                │
  │                           fork()
  │                           ├─ 子进程（后端）
  │                           └─ 父进程（Postmaster）
  │                                │
  │<────── 欢迎消息 ────────────────┤ 后端
  │        (启动参数)               │
  │                                │
  │─ 启动消息 ─────────────────────>│
  │ (用户、密码、数据库)             │
  │                                │
  │                         验证用户
  │                         ClientAuthentication()
  │                                │
  │                         检查pg_hba.conf
  │                         获取认证方法
  │                                │
  │<──── 认证请求 ──────────────────┤
  │ (密码、MD5、SCRAM等)            │
  │                                │
  │─ 认证响应 ─────────────────────>│
  │ (密码/凭据)                     │
  │                                │
  │                         验证凭据
  │                         检查pg_authid
  │                         检查权限
  │                                │
  │<──── 认证成功 ──────────────────┤
  │ (或错误)                        │
  │                                │
  │                         InitPostgres()
  │                         加载relcache
  │                         创建临时模式
  │                                │
  │<──── ReadyForQuery ─────────────┤
  │ (状态 = 'I'表示空闲)            │
  │                                │
  │─ 查询 ────────────────────────>│
  │                                │
  │                         解析 → 分析
  │                         规划 → 执行
  │                                │
  │<─── RowDescription ─────────────┤
  │<─── DataRow ────────────────────┤
  │<─── DataRow ────────────────────┤
  │<─── CommandComplete ────────────┤
  │<─── ReadyForQuery ──────────────┤
  │                                │
```

---

## 7. 关键数据结构

### 7.1 核心执行结构

```c
// 查询描述 - 在执行阶段传递
typedef struct QueryDesc {
    char       *sourceText;        // SQL文本
    CachedPlan *cplan;            // 缓存的计划
    PlannedStmt *plannedstmt;     // 执行计划
    SnapShot    snapshot;         // MVCC快照
    SnapShot    crosscheck_snapshot;
    CommandDest dest;             // 输出目的地
    ParamListInfo params;         // 绑定参数
    QueryEnvironment *queryEnv;
    Instrument *instrument_options;
    EState     *estate;           // 执行器状态（由ExecutorStart设置）
    PlanState  *planstate;        // 计划状态树（由ExecutorStart设置）
    bool        already_executed;
} QueryDesc;

// 执行器状态 - 主执行上下文
typedef struct EState {
    NodeTag     type;
    ScanDirection es_direction;
    Snapshot    es_snapshot;
    Snapshot    es_crosscheck_snapshot;
    List       *es_range_table;
    Index       es_result_relation_info;
    List       *es_result_relations;
    List       *es_root_result_relations;
    List       *es_opened_result_relations;
    List       *es_append_rel_target_lists;
    List       *es_subplanstates;
    List       *es_auxmodify_tables;
    Relation   *es_relations;
    List       *es_rowmarks;
    PlannedStmt *es_plannedstmt;
    const char *es_sourceText;
    JitExprContext *es_jit_context;
    int         es_jit_flags;
    MemoryContext es_query_cxt;
    List       *es_tupleTable;
    ParamListInfo es_param_list_info;
    ParamExecData *es_param_exec_vals;
    QueryEnvironment *es_queryEnv;
    CommandId   es_output_cid;
    int         es_processed;
    Oid         es_lastoid;
    int         es_top_eflags;
    bool        es_output_cid_valid;
    bool        es_in_tuple_routing;
    struct AfterTriggersData *es_ar_result_relations;
    List       *es_agg_nodes;
} EState;

// 计划状态 - 节点特定的执行状态
typedef struct PlanState {
    NodeTag     type;
    Plan       *plan;
    EState     *state;
    ExecProcNodeMtd ExecProcNode;
    PlanState  *lefttree;
    PlanState  *righttree;
    List       *initPlan;
    List       *subPlan;
    ExprContext *ps_ExprContext;
    ProjectionInfo *ps_ProjInfo;
    bool        ps_TupFromTlist;
    ResultRelInfo *ps_ResultRelInfo;
    List       *ps_qual;
    struct instrumentation *instrument;
    struct WorkerInstrumentation *worker_instrument;
    bool        worker_jit_instrument;
    void       *ps_ExprContext_cache;
} PlanState;

// 计划节点（表示操作的树节点）
typedef struct Plan {
    NodeTag     type;
    Cost        startup_cost;
    Cost        total_cost;
    double      rows;
    int         parallel_aware;
    int         parallel_safe;
    int         plan_node_id;
    List       *initPlan;
    List       *targetlist;
    List       *qual;
    List       *lefttree;
    List       *righttree;
    List       *initPlan;
    Bitmapset  *extParam;
    Bitmapset  *allParam;
} Plan;
```

### 7.2 MVCC相关结构

```c
// 快照 - 定义可见的事务集
typedef struct SnapshotData {
    SnapshotType snapshot_type;
    TransactionId xmin;           // 最旧的活动XID
    TransactionId xmax;           // 要分配的下一个XID
    TransactionId *xip;           // 活动XID
    uint32      xcnt;             // 活动XID数量
    TransactionId *subxip;        // 活动子事务XID
    int32       subxcnt;          // 子事务XID数量
    bool        suboverflowed;    // 子事务太多无法跟踪
    bool        takenDuringRecovery;
    bool        copied;
    CommandId   curcid;           // 命令级可见性的命令ID
    uint32      speculativeToken;
    XLogRecPtr  lsn;              // 获取快照时的WAL位置
    struct SnapshotData *active_count_delta;
} SnapshotData;

// 事务状态
typedef struct TransactionStateData {
    FullTransactionId fullTransactionId;
    SubTransactionId subTransactionId;
    char       *name;
    int         savepointLevel;
    TransState  state;
    TBlockState blockState;
    int         nestingLevel;
    int         gucNestLevel;
    MemoryContext curTransactionContext;
    ResourceOwner curTransactionOwner;
    MemoryContext priorContext;
    TransactionId *childXids;
    int         nChildXids;
    int         maxChildXids;
} TransactionStateData;

// 元组头 - 每个元组存储的元数据
typedef struct HeapTupleHeaderData {
    union {
        HeapTupleFields t_heap;
        DatumTupleFields t_datum;
    } t_choice;
    ItemPointerData t_ctid;      // 元组或其更新的位置
    uint16      t_infomask2;     // 标志
    uint16      t_infomask;      // 提交/锁状态标志
    uint8       t_hoff;          // 到用户数据的偏移
    bits8       t_bits[FLEXIBLE_ARRAY_MEMBER];
} HeapTupleHeaderData;
```

### 7.3 缓冲池结构

```c
// 缓冲区描述符
typedef struct BufferDesc {
    BufferTag   tag;            // 页面标识
    int         freeNext;       // 空闲列表中的下一个空闲缓冲区
    int         freePrev;       // 前一个空闲缓冲区
    uint32      refcount;       // Pin计数
    uint32      usageCount;     // LRU使用计数器
    BufFlags    flags;          // 脏、I/O进行中等
    uint16      wait_backend_pid;
    int         io_in_progress_lock;
    spin_delay_status delayStatus;
    BufferAccessStrategy ring;  // 缓冲区环（如果有）
    LocalBufferDesc *local_buf; // 本地缓冲区描述符（如果是本地）
} BufferDesc;

// 缓冲区标签 - 页面标识符
typedef struct BufferTag {
    RelFileLocator rlocator;    // 关系文件标识符
    ForkNumber  forkNum;        // 关系fork
    BlockNumber blockNum;       // 块号
} BufferTag;

// 本地缓冲池（每个后端）
typedef struct {
    Block       buffer;         // 实际页面数据
    BufferTag   tag;            // 页面标识符
    int         flags;          // Pin计数、脏等
    VacuumPhase vacuumPhase;    // 用于VACUUM操作
    bits8       extra;          // 额外标志
} LocalBufferDesc;
```

---

## 8. 性能考虑

### 8.1 查询优化技术

1. **索引选择**：规划器根据成本选择最佳索引
2. **连接排序**：优化器确定最优的表连接序列
3. **谓词下推**：将过滤器推送到最早的执行点
4. **路径修剪**：消除高成本路径
5. **并行执行**：在有益时标记计划以使用并行工作进程

### 8.2 MVCC优化

1. **提示位**：避免重复的事务状态查找
2. **快照缓存**：在多个操作中重用快照
3. **Xmin范围跟踪**：高效清理旧元组版本
4. **延迟冻结**：推迟昂贵的冻结操作
5. **并发Vacuum**：不阻塞查询的Vacuum

### 8.3 缓冲池优化

1. **缓冲区环策略**：为顺序扫描提供专用缓冲区
2. **时钟扫描算法**：公平的LRU替换
3. **分区锁定**：减少缓冲区访问时的争用
4. **环缓冲区**：防止大型操作驱逐缓存

---

## 总结

PostgreSQL的执行模型包括：

1. **服务器启动**：初始化、配置加载、共享内存设置
2. **连接流程**：套接字接受、认证、会话初始化
3. **查询处理**：解析 → 分析 → 规划 → 执行流水线
4. **事务管理**：MVCC快照、隔离级别、WAL日志
5. **存储访问**：缓冲池、页面I/O、锁管理、元组可见性

每个组件都经过精心设计，以在并发负载下实现性能和正确性。

---

**文档版本**: 1.0
**分析日期**: 2025-11-06
**PostgreSQL版本**: 开发版（基于最新master分支）
**代码库路径**: /home/user/postgres
