# PostgreSQL 程序执行流程说明和流程图

## 文档概述

本文档详细说明PostgreSQL数据库的程序执行流程，包括服务器启动、客户端连接、查询处理、事务管理和存储访问的完整流程。

---

## 目录

1. [服务器启动流程](#1-服务器启动流程)
2. [客户端连接流程](#2-客户端连接流程)
3. [查询处理流程](#3-查询处理流程)
4. [事务处理流程](#4-事务处理流程)
5. [存储访问流程](#5-存储访问流程)
6. [MVCC可见性检查](#6-mvcc可见性检查)
7. [后台进程协作](#7-后台进程协作)

---

## 1. 服务器启动流程

### 1.1 启动流程概览

```
┌─────────────────────────────────────────────────────────────┐
│                    服务器启动流程                              │
└─────────────────────────────────────────────────────────────┘

[操作系统启动PostgreSQL]
         │
         ↓
    main() 函数
    (src/backend/main/main.c)
         │
         ├─ 平台初始化
         ├─ 内存上下文初始化
         ├─ 区域设置初始化
         ├─ 命令行参数解析
         └─ 根据模式分发
         │
         ↓
  PostmasterMain()
  (src/backend/postmaster/postmaster.c)
         │
         ├─ [阶段1] 进程设置
         ├─ [阶段2] 内存上下文创建
         ├─ [阶段3] 安装路径设置
         ├─ [阶段4] 信号处理器设置
         ├─ [阶段5] GUC选项初始化
         ├─ [阶段6] 命令行选项解析
         ├─ [阶段7] 配置文件加载
         ├─ [阶段8] 数据目录验证
         ├─ [阶段9] 配置验证
         └─ [阶段10] 进入主循环
         │
         ↓
    ServerLoop()
         │
         ├─ 创建共享内存
         ├─ 初始化共享数据结构
         ├─ 启动后台进程
         └─ 监听客户端连接
```

### 1.2 详细启动阶段

#### 阶段1-4: 基础初始化

```
PostmasterMain() 初始化序列:

1. 进程设置
   ├─ InitProcessGlobals()
   ├─ 设置 PostmasterPid = MyProcPid
   ├─ IsPostmasterEnvironment = true
   └─ Windows信号初始化（如果需要）

2. 内存上下文
   ├─ umask(PG_MODE_MASK_OWNER)       # 设置严格权限
   ├─ 创建 PostmasterContext
   └─ 切换到 PostmasterContext

3. 安装路径
   └─ getInstallationPaths(argv[0])   # 确定程序路径

4. 信号处理器
   ├─ 阻塞所有信号
   ├─ SIGHUP  → handle_pm_reload_request_signal (重载配置)
   ├─ SIGTERM → handle_pm_shutdown_request_signal (关闭)
   ├─ SIGINT  → handle_pm_shutdown_request_signal (快速关闭)
   ├─ SIGQUIT → handle_pm_shutdown_request_signal (立即关闭)
   ├─ SIGUSR1 → handle_pm_pmsignal_signal (后台进程信号)
   ├─ SIGCHLD → handle_pm_child_exit_signal (子进程退出)
   └─ 解除信号阻塞
```

#### 阶段5-9: 配置和验证

```
配置加载和验证:

5. GUC初始化
   └─ InitializeGUCOptions()          # 初始化所有配置变量

6. 命令行解析
   ├─ -B shared_buffers               # 共享缓冲区大小
   ├─ -D data_directory               # 数据目录
   ├─ -p port                         # 监听端口
   ├─ -h listen_addresses             # 监听地址
   └─ 其他配置参数

7. 配置文件
   ├─ SelectConfigFiles()
   ├─ 加载 postgresql.conf
   ├─ 加载 postgresql.auto.conf
   └─ 加载 pg_hba.conf

8. 数据目录验证
   ├─ checkDataDir()                  # 检查目录存在和权限
   ├─ checkControlFile()              # 验证控制文件
   └─ ChangeToDataDir()               # 切换到数据目录

9. 配置验证
   ├─ 验证连接数限制
   ├─ 验证WAL设置
   ├─ 验证复制配置
   └─ 检查无效的配置组合
```

### 1.3 共享内存初始化

```
共享内存结构创建:

CreateOrAttachShmemStructs()
    │
    ├─ PGSharedMemoryCreate()
    │   ├─ 计算所需内存大小
    │   │   ├─ Buffer Pool: shared_buffers * 8KB
    │   │   ├─ Lock Table: max_connections * 锁结构
    │   │   ├─ Process Array: max_connections * 进程槽
    │   │   └─ 其他共享结构
    │   └─ 分配共享内存段
    │
    ├─ InitShmemAllocation()
    │   └─ 初始化共享内存分配器
    │
    ├─ CreateSharedMemoryAndSemaphores()
    │   ├─ 创建共享缓冲池
    │   │   ├─ Buffer Descriptors (BufferDesc数组)
    │   │   ├─ Buffer Blocks (实际数据页)
    │   │   └─ Buffer Hash Table (查找表)
    │   │
    │   ├─ 创建锁管理结构
    │   │   ├─ Lock Hash Table
    │   │   ├─ ProcLock Hash Table
    │   │   └─ LWLock Array
    │   │
    │   ├─ 创建进程数组 (PGPROC)
    │   │   └─ max_connections个槽位
    │   │
    │   ├─ 创建WAL控制结构
    │   │   ├─ XLogCtlData
    │   │   ├─ XLogCtlInsert
    │   │   └─ WAL Buffers
    │   │
    │   ├─ 创建事务日志 (CLOG)
    │   │
    │   └─ 创建统计信息缓冲区
    │
    └─ 创建信号量集合
```

### 1.4 后台进程启动

```
后台进程启动序列:

ServerLoop()
    │
    └─ LaunchMissingBackgroundProcesses()
        │
        ├─ [1] Startup Process (启动进程)
        │   ├─ 功能: WAL恢复和数据库一致性检查
        │   ├─ 读取控制文件
        │   ├─ 执行崩溃恢复
        │   ├─ 应用WAL记录
        │   └─ 完成后通知Postmaster进入PM_RUN状态
        │
        ├─ [2] Background Writer (后台写进程)
        │   ├─ 功能: 定期将脏页写入磁盘
        │   ├─ 扫描缓冲池
        │   ├─ 写出部分脏页
        │   └─ 减少检查点压力
        │
        ├─ [3] Checkpointer (检查点进程)
        │   ├─ 功能: 执行检查点操作
        │   ├─ 定期触发检查点
        │   ├─ 将所有脏页刷新到磁盘
        │   ├─ 更新控制文件
        │   └─ 创建一致性恢复点
        │
        ├─ [4] WAL Writer (WAL写进程)
        │   ├─ 功能: 将WAL缓冲区写入磁盘
        │   ├─ 定期刷新WAL
        │   ├─ 减少提交延迟
        │   └─ 支持异步提交
        │
        ├─ [5] Autovacuum Launcher (自动清理启动器)
        │   ├─ 功能: 管理自动VACUUM
        │   ├─ 监控表的死元组率
        │   ├─ 启动autovacuum worker
        │   └─ 更新统计信息
        │
        ├─ [6] Stats Collector (统计收集器)
        │   ├─ 功能: 收集数据库统计信息
        │   ├─ 收集表/索引访问统计
        │   ├─ 收集数据库活动统计
        │   └─ 写入统计文件
        │
        ├─ [7] WAL Archiver (WAL归档进程) [可选]
        │   ├─ 功能: 归档完成的WAL文件
        │   ├─ 复制WAL文件到归档位置
        │   └─ 支持PITR (时间点恢复)
        │
        └─ [8] Logical Replication Launcher [可选]
            ├─ 功能: 管理逻辑复制
            └─ 启动订阅worker进程
```

### 1.5 Postmaster状态机

```
Postmaster状态转换:

PM_INIT (初始化)
    │
    ↓
PM_STARTUP (等待启动进程)
    │
    ├─ 如果需要恢复 ──→ PM_RECOVERY
    │                      │
    │                      ├─ 如果是热备 ──→ PM_HOT_STANDBY
    │                      │                    │
    │                      └─ 恢复完成 ────────┘
    │                                           │
    └─────────────────────────────────────────┘
                                                │
                                                ↓
                                        PM_RUN (正常运行)
                                                │
                                                │ 收到关闭信号
                                                ↓
                                        PM_STOP_BACKENDS
                                                │
                                                ↓
                                        PM_WAIT_BACKENDS
                                                │
                                                ↓
                                        PM_SHUTDOWN
                                                │
                                                ↓
                                        PM_NO_CHILDREN
                                                │
                                                ↓
                                            [退出]
```

---

## 2. 客户端连接流程

### 2.1 连接建立流程

```
┌─────────────────────────────────────────────────────────────┐
│                    客户端连接流程                              │
└─────────────────────────────────────────────────────────────┘

[客户端应用]
     │
     │ 1. 建立TCP连接
     ↓
[Postmaster监听端口]
     │
     │ 2. accept() 接受连接
     ↓
  fork()
     │
     ├─────────────────────────────┬─────────────────────────┐
     │ (父进程)                     │ (子进程)                 │
     ↓                             ↓                         │
[Postmaster]                  [Backend Process]             │
继续监听                       │                            │
                              │ 3. 初始化子进程             │
                              ├─ 关闭监听socket             │
                              ├─ 初始化信号处理器            │
                              ├─ 设置进程标题               │
                              └─ 调用 PostgresMain()       │
                                   │                        │
                                   │ 4. 认证阶段            │
                                   ├─ 发送启动消息          │
                                   ├─ 接收认证参数          │
                                   ├─ 检查 pg_hba.conf     │
                                   ├─ 执行认证协议          │
                                   │  (密码/SCRAM/GSSAPI等) │
                                   ├─ 验证用户凭证          │
                                   └─ 认证成功/失败         │
                                        │                   │
                                        │ 5. 会话初始化     │
                                        ├─ 创建内存上下文    │
                                        ├─ 初始化relcache   │
                                        ├─ 初始化catcache   │
                                        ├─ 获取PGPROC槽     │
                                        ├─ 初始化锁表        │
                                        └─ 设置默认事务状态  │
                                             │              │
                                             ↓              │
                                      [进入查询处理循环]    │
                                             │              │
                                             ↓              │
                                      PostgresMain()       │
                                             │              │
                                             └──────────────┘
```

### 2.2 认证过程详解

```
认证流程:

PostgresMain() 认证阶段
    │
    ├─ 1. 发送启动消息
    │   └─ 发送服务器版本、编码等信息
    │
    ├─ 2. 接收启动包
    │   ├─ 读取客户端请求
    │   ├─ 提取数据库名
    │   ├─ 提取用户名
    │   └─ 提取连接参数
    │
    ├─ 3. 检查连接权限 (pg_hba.conf)
    │   ├─ 匹配主机地址
    │   ├─ 匹配数据库
    │   ├─ 匹配用户
    │   └─ 确定认证方法
    │       ├─ trust (信任)
    │       ├─ reject (拒绝)
    │       ├─ password (明文密码)
    │       ├─ md5 (MD5密码)
    │       ├─ scram-sha-256 (SCRAM)
    │       ├─ gss (GSSAPI/Kerberos)
    │       ├─ sspi (Windows SSPI)
    │       ├─ peer (Unix peer)
    │       ├─ ident (Ident)
    │       └─ cert (SSL证书)
    │
    ├─ 4. 执行认证协议
    │   │
    │   ├─ [如果是password/md5]
    │   │   ├─ 请求密码
    │   │   ├─ 接收密码
    │   │   ├─ 查询pg_authid
    │   │   ├─ 验证密码哈希
    │   │   └─ 返回结果
    │   │
    │   ├─ [如果是scram-sha-256]
    │   │   ├─ 发送SCRAM挑战
    │   │   ├─ 接收客户端响应
    │   │   ├─ 验证响应
    │   │   ├─ 发送服务器最终消息
    │   │   └─ 确认认证
    │   │
    │   └─ [如果是gss/sspi]
    │       ├─ GSSAPI握手
    │       ├─ 票据验证
    │       └─ 建立安全上下文
    │
    ├─ 5. 权限检查
    │   ├─ 检查用户是否有登录权限
    │   ├─ 检查连接数限制
    │   ├─ 检查数据库访问权限
    │   └─ 应用角色设置
    │
    └─ 6. 认证结果
        ├─ 成功 → 发送AuthenticationOk
        └─ 失败 → 发送错误消息并关闭连接
```

### 2.3 会话初始化

```
会话初始化流程:

InitPostgres()
(src/backend/utils/init/postinit.c)
    │
    ├─ 1. 基础初始化
    │   ├─ InitProcessGlobals()
    │   ├─ 设置 MyDatabaseId
    │   ├─ 设置 MyProc
    │   └─ 初始化随机数生成器
    │
    ├─ 2. 内存上下文
    │   ├─ 创建 MessageContext
    │   ├─ 创建 QueryContext
    │   └─ 创建 TransactionContext
    │
    ├─ 3. 系统目录访问初始化
    │   ├─ RelationCacheInitialize()
    │   │   └─ 初始化关系缓存
    │   └─ InitCatalogCache()
    │       └─ 初始化系统目录缓存
    │
    ├─ 4. 进程槽分配
    │   ├─ InitProcess()
    │   ├─ 从PGPROC数组获取槽位
    │   ├─ 初始化 MyProc
    │   └─ 注册到进程数组
    │
    ├─ 5. 锁管理器初始化
    │   └─ InitLockTable()
    │
    ├─ 6. 数据库连接
    │   ├─ 打开目标数据库
    │   ├─ 加载数据库设置
    │   └─ 设置搜索路径
    │
    ├─ 7. 事务状态初始化
    │   ├─ InitCurrentTransactionState()
    │   └─ 设置为 TRANS_DEFAULT 状态
    │
    └─ 8. 其他初始化
        ├─ 初始化临时模式
        ├─ 加载共享库
        └─ 应用角色设置
```

---

## 3. 查询处理流程

### 3.1 查询处理总体流程

```
┌──────────────────────────────────────────────────────────────┐
│                      查询处理流程                              │
└──────────────────────────────────────────────────────────────┘

[客户端发送SQL]
      │
      ↓
PostgresMain() 主循环
(src/backend/tcop/postgres.c)
      │
      ├─ ReadCommand()
      │   └─ 从客户端读取SQL字符串
      │
      ↓
exec_simple_query(query_string)
      │
      ├─────────────────────────────────────────────────────────┐
      │                                                         │
      │ [阶段1: 解析 - Parser]                                  │
      ├─ pg_parse_query(query_string)                          │
      │   (src/backend/parser/)                                │
      │   │                                                     │
      │   ├─ raw_parser(query_string)                          │
      │   │   ├─ scanner_init(query_string)  [scan.l]          │
      │   │   │   └─ 词法分析: SQL → Tokens                    │
      │   │   │                                                 │
      │   │   └─ base_yyparse()  [gram.y]                      │
      │   │       └─ 语法分析: Tokens → Raw Parse Tree         │
      │   │                                                     │
      │   └─ 返回: List<RawStmt>                               │
      │                                                         │
      ↓                                                         │
      │ [阶段2: 分析 - Analyzer]                                │
      ├─ parse_analyze(raw_parse_tree)                         │
      │   (src/backend/parser/analyze.c)                       │
      │   │                                                     │
      │   ├─ transformStmt(raw_stmt)                           │
      │   │   ├─ 类型检查                                       │
      │   │   ├─ 名称解析 (表名、列名、函数名)                  │
      │   │   ├─ 展开 * 为具体列                               │
      │   │   ├─ 权限检查                                       │
      │   │   └─ 表达式转换                                     │
      │   │                                                     │
      │   └─ 返回: Query Tree                                  │
      │                                                         │
      ↓                                                         │
      │ [阶段3: 重写 - Rewriter]                                │
      ├─ pg_rewrite_query(query)                               │
      │   (src/backend/rewrite/)                               │
      │   │                                                     │
      │   ├─ QueryRewrite(query)                               │
      │   │   ├─ 应用规则 (Rules)                              │
      │   │   ├─ 展开视图 (Views)                              │
      │   │   ├─ 处理ON INSERT/UPDATE/DELETE规则               │
      │   │   ├─ 处理RETURNING子句                             │
      │   │   └─ 安全屏障视图处理                               │
      │   │                                                     │
      │   └─ 返回: List<Query> (可能多个查询)                   │
      │                                                         │
      ↓                                                         │
      │ [阶段4: 规划 - Planner/Optimizer]                       │
      ├─ pg_plan_queries(queries)                              │
      │   (src/backend/optimizer/plan/planner.c)               │
      │   │                                                     │
      │   └─ 对每个Query:                                      │
      │       │                                                 │
      │       ├─ planner(query)                                │
      │       │   │                                             │
      │       │   ├─ subquery_planner()                        │
      │       │   │   ├─ 子查询优化                            │
      │       │   │   ├─ 表达式预处理                          │
      │       │   │   └─ 常量折叠                              │
      │       │   │                                             │
      │       │   ├─ grouping_planner()                        │
      │       │   │   │                                         │
      │       │   │   ├─ [路径生成]                            │
      │       │   │   │   query_planner()                      │
      │       │   │   │   ├─ 生成扫描路径                      │
      │       │   │   │   │   ├─ SeqScan (顺序扫描)            │
      │       │   │   │   │   ├─ IndexScan (索引扫描)          │
      │       │   │   │   │   ├─ IndexOnlyScan                │
      │       │   │   │   │   ├─ BitmapScan                   │
      │       │   │   │   │   └─ TidScan                      │
      │       │   │   │   │                                     │
      │       │   │   │   ├─ 生成连接路径                      │
      │       │   │   │   │   ├─ NestedLoop (嵌套循环)         │
      │       │   │   │   │   ├─ HashJoin (哈希连接)           │
      │       │   │   │   │   └─ MergeJoin (归并连接)          │
      │       │   │   │   │                                     │
      │       │   │   │   └─ 成本估算                          │
      │       │   │   │       ├─ cost_seqscan()               │
      │       │   │   │       ├─ cost_index()                 │
      │       │   │   │       └─ 选择最低成本路径              │
      │       │   │   │                                         │
      │       │   │   ├─ [聚合/分组处理]                        │
      │       │   │   ├─ [排序处理]                            │
      │       │   │   └─ [LIMIT处理]                           │
      │       │   │                                             │
      │       │   └─ create_plan(best_path)                    │
      │       │       └─ 将Path转换为Plan树                    │
      │       │                                                 │
      │       └─ 返回: PlannedStmt (执行计划)                   │
      │                                                         │
      ↓                                                         │
      │ [阶段5: 执行 - Executor]                                │
      └─ PortalRun(portal)                                     │
          (src/backend/executor/execMain.c)                    │
          │                                                     │
          ├─ ExecutorStart(queryDesc)                          │
          │   ├─ 初始化执行器状态 (EState)                      │
          │   ├─ 初始化计划节点                                 │
          │   ├─ 打开关系                                       │
          │   └─ 获取锁                                         │
          │                                                     │
          ├─ ExecutorRun(queryDesc)                            │
          │   │                                                 │
          │   └─ 循环获取元组:                                  │
          │       │                                             │
          │       ├─ ExecProcNode(planstate)                   │
          │       │   │                                         │
          │       │   ├─ [根据节点类型调用相应函数]             │
          │       │   │   ├─ ExecSeqScan()                     │
          │       │   │   ├─ ExecIndexScan()                   │
          │       │   │   ├─ ExecNestLoop()                    │
          │       │   │   ├─ ExecHashJoin()                    │
          │       │   │   ├─ ExecAgg()                         │
          │       │   │   └─ ...                               │
          │       │   │                                         │
          │       │   ├─ 从存储层获取数据                       │
          │       │   ├─ MVCC可见性检查                         │
          │       │   ├─ 应用过滤条件                           │
          │       │   └─ 返回元组                               │
          │       │                                             │
          │       └─ 发送元组到客户端                           │
          │                                                     │
          ├─ ExecutorFinish(queryDesc)                         │
          │   └─ 完成触发器等                                   │
          │                                                     │
          └─ ExecutorEnd(queryDesc)                            │
              ├─ 清理执行器状态                                 │
              ├─ 关闭关系                                       │
              └─ 释放资源                                       │
                                                                │
                                                                ↓
                                                        [结果返回客户端]
```

### 3.2 解析阶段详解

```
解析阶段 (Parser):

输入: "SELECT * FROM users WHERE age > 18;"
     │
     ↓
[词法分析 - scan.l]
     │
     ├─ 扫描字符串
     ├─ 识别关键字: SELECT, FROM, WHERE
     ├─ 识别标识符: users, age
     ├─ 识别操作符: *, >, ;
     └─ 识别常量: 18
     │
     └─→ Token流: [SELECT] [*] [FROM] [users] [WHERE] [age] [>] [18] [;]
          │
          ↓
[语法分析 - gram.y (Bison)]
     │
     ├─ 应用语法规则
     ├─ 构建语法树
     └─ 验证语法正确性
     │
     └─→ Raw Parse Tree:
          SelectStmt
          ├─ targetList: [ResTarget("*")]
          ├─ fromClause: [RangeVar("users")]
          └─ whereClause: [A_Expr(">", "age", 18)]
```

### 3.3 分析阶段详解

```
分析阶段 (Analyzer):

输入: Raw Parse Tree
     │
     ↓
transformStmt()
     │
     ├─ [名称解析]
     │   ├─ 查找表 "users" → relid
     │   ├─ 查找列 "age" → attnum, type
     │   └─ 展开 * → id, name, age, email...
     │
     ├─ [类型检查]
     │   ├─ age的类型: INTEGER
     │   ├─ 常量18的类型: INTEGER
     │   └─ 操作符>: INTEGER × INTEGER → BOOLEAN
     │
     ├─ [权限检查]
     │   └─ 用户是否有SELECT权限
     │
     └─ [构建Query Tree]
         │
         └─→ Query
             ├─ commandType: CMD_SELECT
             ├─ rtable: [RangeTblEntry(users)]
             ├─ targetList:
             │   ├─ TargetEntry(id)
             │   ├─ TargetEntry(name)
             │   ├─ TargetEntry(age)
             │   └─ TargetEntry(email)
             └─ quals: [OpExpr(">", Var(age), Const(18))]
```

### 3.4 优化阶段详解

```
优化阶段 (Planner):

输入: Query Tree
     │
     ↓
planner()
     │
     ├─ [1. 路径生成]
     │   │
     │   ├─ 顺序扫描路径:
     │   │   SeqScan(users)
     │   │   Cost: 启动=0.00, 总计=100.00
     │   │
     │   ├─ 索引扫描路径 (如果有索引):
     │   │   IndexScan(users, age_idx)
     │   │   Cost: 启动=0.29, 总计=25.50
     │   │
     │   └─ 位图扫描路径 (如果合适):
     │       BitmapHeapScan(users)
     │       Cost: 启动=4.50, 总计=45.00
     │
     ├─ [2. 成本估算]
     │   │
     │   ├─ 估算行数 (selectivity)
     │   │   ├─ 表总行数: 1000
     │   │   ├─ WHERE age > 18 选择性: 0.7
     │   │   └─ 估计返回: 700行
     │   │
     │   ├─ 计算I/O成本
     │   │   ├─ 顺序扫描: 读取所有页面
     │   │   └─ 索引扫描: 索引页 + 部分堆页
     │   │
     │   └─ 计算CPU成本
     │       ├─ 元组处理成本
     │       └─ 表达式求值成本
     │
     ├─ [3. 选择最优路径]
     │   └─ 选择 IndexScan (成本最低)
     │
     └─ [4. 生成执行计划]
         │
         └─→ PlannedStmt
             └─ planTree:
                 IndexScan
                 ├─ relation: users
                 ├─ indexid: age_idx
                 ├─ indexqual: age > 18
                 └─ cost: 0.29..25.50
```

### 3.5 执行阶段详解

```
执行阶段 (Executor):

输入: PlannedStmt
     │
     ↓
ExecutorStart()
     │
     ├─ 创建 EState (执行器状态)
     ├─ 创建 QueryDesc
     ├─ 初始化 PlanState 树
     │   └─ IndexScanState
     │       ├─ 打开关系 users
     │       ├─ 打开索引 age_idx
     │       ├─ 准备扫描键
     │       └─ 获取锁
     │
     ↓
ExecutorRun()
     │
     └─ while (还有元组):
         │
         ├─ ExecIndexScan(scanstate)
         │   │
         │   ├─ [1] 索引扫描
         │   │   ├─ index_getnext()
         │   │   ├─ 在索引中查找 age > 18
         │   │   └─ 返回 TID (ItemPointer)
         │   │
         │   ├─ [2] 堆元组获取
         │   │   ├─ index_fetch_heap(TID)
         │   │   ├─ 从缓冲池获取页面
         │   │   └─ 提取元组
         │   │
         │   ├─ [3] MVCC可见性检查
         │   │   ├─ HeapTupleSatisfiesMVCC()
         │   │   ├─ 检查 xmin (插入事务)
         │   │   ├─ 检查 xmax (删除事务)
         │   │   └─ 与当前快照比较
         │   │       ├─ 可见 → 继续
         │   │       └─ 不可见 → 跳过
         │   │
         │   ├─ [4] 应用额外过滤条件
         │   │   └─ ExecQual(scanstate->ps.qual)
         │   │
         │   └─ [5] 投影
         │       └─ ExecProject()
         │           └─ 提取需要的列
         │
         └─ [6] 返回元组
             └─ printtup() → 发送给客户端
                                                                │
ExecutorFinish()                                               │
     └─ 触发器处理                                              │
                                                                │
ExecutorEnd()                                                  │
     ├─ 关闭索引                                                │
     ├─ 关闭关系                                                │
     ├─ 释放锁                                                  │
     └─ 清理内存                                                │
                                                                ↓
                                                        [查询完成]
```

---

## 4. 事务处理流程

### 4.1 事务生命周期

```
┌──────────────────────────────────────────────────────────────┐
│                      事务处理流程                              │
└──────────────────────────────────────────────────────────────┘

[客户端] BEGIN
    │
    ↓
start_xact_command()
(src/backend/access/transam/xact.c)
    │
    ├─ StartTransactionCommand()
    │   │
    │   ├─ 检查当前事务状态
    │   │
    │   └─ BeginTransactionBlock()
    │       │
    │       ├─ [分配事务ID (XID)]
    │       │   ├─ GetNewTransactionId()
    │       │   ├─ 从共享内存获取下一个XID
    │       │   └─ MyProc->xid = new_xid
    │       │
    │       ├─ [创建事务快照]
    │       │   ├─ GetSnapshotData()
    │       │   ├─ 记录活动事务列表
    │       │   ├─ xmin = 最老活动XID
    │       │   └─ xmax = 下一个XID
    │       │
    │       ├─ [初始化事务状态]
    │       │   ├─ 设置 TransactionState
    │       │   ├─ 设置隔离级别
    │       │   └─ 初始化资源跟踪器
    │       │
    │       └─ 设置 s->blockState = TBLOCK_BEGIN
    │
    ↓
[执行SQL语句]
    │
    ├─ 每个语句:
    │   ├─ 使用事务快照进行可见性检查
    │   ├─ 修改数据时设置元组xmin/xmax
    │   └─ 生成WAL记录
    │
    ↓
[客户端] COMMIT
    │
    ↓
CommitTransactionCommand()
    │
    ├─ [预提交阶段]
    │   ├─ PreCommit_Notify()
    │   ├─ PreCommit_CheckForSerializationFailure()
    │   ├─ PreCommit_on_commit_actions()
    │   └─ PreCommit_Portals()
    │
    ├─ [提交阶段]
    │   CommitTransaction()
    │   │
    │   ├─ [1. 记录提交WAL]
    │   │   ├─ XLogInsert(XLOG_XACT_COMMIT)
    │   │   ├─ 写入提交记录到WAL缓冲区
    │   │   └─ XLogFlush() - 刷新WAL到磁盘
    │   │       │
    │   │       └─ 如果 synchronous_commit=on:
    │   │           └─ fsync(wal_file)
    │   │
    │   ├─ [2. 更新CLOG]
    │   │   ├─ TransactionIdCommitTree()
    │   │   └─ 设置XID状态为COMMITTED
    │   │
    │   ├─ [3. 释放锁]
    │   │   └─ LockReleaseAll()
    │   │       ├─ 释放表锁
    │   │       ├─ 释放行锁
    │   │       └─ 释放Advisory锁
    │   │
    │   ├─ [4. 清理资源]
    │   │   ├─ ResourceOwnerRelease()
    │   │   ├─ 关闭游标
    │   │   ├─ 删除临时文件
    │   │   └─ 释放内存上下文
    │   │
    │   └─ [5. 清除XID]
    │       └─ MyProc->xid = InvalidTransactionId
    │
    └─ [后提交阶段]
        ├─ AtEOXact_RelationCache()
        ├─ AtEOXact_Inval()
        └─ 处理延迟触发器

[客户端] ROLLBACK
    │
    ↓
AbortTransaction()
    │
    ├─ [1. 记录回滚WAL]
    │   └─ XLogInsert(XLOG_XACT_ABORT)
    │
    ├─ [2. 更新CLOG]
    │   └─ 设置XID状态为ABORTED
    │
    ├─ [3. 释放锁]
    │   └─ LockReleaseAll()
    │
    ├─ [4. 撤销更改]
    │   └─ MVCC自动处理（元组不可见）
    │
    └─ [5. 清理资源]
        └─ ResourceOwnerRelease()
```

### 4.2 MVCC实现

```
MVCC (多版本并发控制):

元组结构:
┌─────────────────────────────────────┐
│     HeapTupleHeader                 │
├─────────────────────────────────────┤
│ t_xmin   : TransactionId (创建XID)  │
│ t_xmax   : TransactionId (删除XID)  │
│ t_cid    : CommandId                │
│ t_ctid   : ItemPointer (链指针)     │
│ t_infomask: 状态标志位               │
├─────────────────────────────────────┤
│ 实际数据...                          │
└─────────────────────────────────────┘

快照 (Snapshot):
┌─────────────────────────────────────┐
│ xmin: TransactionId                 │
│      最老的仍在运行的事务            │
│                                     │
│ xmax: TransactionId                 │
│      下一个将被分配的XID             │
│                                     │
│ xip[]: TransactionId[]              │
│      快照时刻的活动事务列表          │
│                                     │
│ xcnt: int                           │
│      活动事务数量                    │
└─────────────────────────────────────┘

可见性检查算法:

HeapTupleSatisfiesMVCC(tuple, snapshot)
    │
    ├─ [1] 检查元组是否被删除
    │   │
    │   └─ if (tuple->t_xmax != 0):
    │       ├─ 如果 xmax < snapshot->xmin:
    │       │   └─ 已提交删除 → 不可见
    │       │
    │       ├─ 如果 xmax >= snapshot->xmax:
    │       │   └─ 删除事务在快照后开始 → 可见
    │       │
    │       └─ 如果 xmax in snapshot->xip[]:
    │           └─ 删除事务活动中 → 可见
    │
    ├─ [2] 检查元组是否可见
    │   │
    │   └─ if (tuple->t_xmin != 0):
    │       ├─ 如果 xmin < snapshot->xmin:
    │       │   └─ 已提交插入 → 可见
    │       │
    │       ├─ 如果 xmin >= snapshot->xmax:
    │       │   └─ 插入事务在快照后开始 → 不可见
    │       │
    │       ├─ 如果 xmin == 当前XID:
    │       │   └─ 本事务插入 → 可见
    │       │
    │       └─ 如果 xmin in snapshot->xip[]:
    │           └─ 插入事务活动中 → 不可见
    │
    └─ 返回可见性结果

提示位 (Hint Bits):
    为了避免重复检查CLOG，元组头包含提示位:

    ├─ HEAP_XMIN_COMMITTED: xmin已提交
    ├─ HEAP_XMIN_INVALID: xmin已回滚
    ├─ HEAP_XMAX_COMMITTED: xmax已提交
    └─ HEAP_XMAX_INVALID: xmax已回滚
```

### 4.3 WAL日志

```
WAL (Write-Ahead Logging):

WAL记录结构:
┌─────────────────────────────────────┐
│     XLogRecord                      │
├─────────────────────────────────────┤
│ xl_tot_len: 记录总长度               │
│ xl_xid: 事务ID                      │
│ xl_prev: 前一条记录的LSN             │
│ xl_info: 记录类型和标志              │
│ xl_rmid: 资源管理器ID                │
│ xl_crc: CRC校验                     │
├─────────────────────────────────────┤
│ 数据块信息...                        │
│ 主数据...                            │
└─────────────────────────────────────┘

WAL写入流程:

[Backend修改数据]
    │
    ├─ XLogBeginInsert()
    ├─ XLogRegisterData(data)
    ├─ XLogRegisterBuffer(buffer)
    └─ XLogInsert(rmid, info)
        │
        ├─ [1] 分配WAL空间
        │   ├─ ReserveXLogInsertLocation()
        │   └─ 返回LSN (Log Sequence Number)
        │
        ├─ [2] 复制到WAL缓冲区
        │   └─ CopyXLogRecordToWAL()
        │
        ├─ [3] 标记页面为脏
        │   └─ MarkBufferDirty()
        │
        ├─ [4] 更新pg_lsn
        │   └─ PageSetLSN(page, lsn)
        │
        └─ [5] 提交时刷新
            └─ XLogFlush(lsn)
                │
                ├─ 通知WAL Writer
                ├─ 等待写入完成
                └─ fsync() [如果synchronous_commit=on]

WAL记录类型:
├─ XLOG_HEAP_INSERT: 插入元组
├─ XLOG_HEAP_DELETE: 删除元组
├─ XLOG_HEAP_UPDATE: 更新元组
├─ XLOG_HEAP_HOT_UPDATE: HOT更新
├─ XLOG_XACT_COMMIT: 事务提交
├─ XLOG_XACT_ABORT: 事务回滚
├─ XLOG_BTREE_INSERT: B-tree插入
└─ ...
```

---

## 5. 存储访问流程

### 5.1 缓冲池访问

```
┌──────────────────────────────────────────────────────────────┐
│                      缓冲池访问流程                            │
└──────────────────────────────────────────────────────────────┘

[Executor请求页面]
    │
    ↓
ReadBuffer(relation, blocknum)
(src/backend/storage/buffer/bufmgr.c)
    │
    ├─ [1] 计算缓冲区标签
    │   BufferTag = {rnode, forknum, blocknum}
    │
    ├─ [2] 在缓冲池中查找
    │   │
    │   └─ BufTableLookup(BufferTag)
    │       ├─ 计算哈希值
    │       ├─ 在哈希表中查找
    │       │
    │       ├─ [缓存命中]
    │       │   ├─ 找到 buffer_id
    │       │   ├─ PinBuffer(buffer_id)
    │       │   │   ├─ 增加引用计数
    │       │   │   └─ 更新usage_count
    │       │   └─ 返回 Buffer
    │       │
    │       └─ [缓存未命中]
    │           │
    │           ├─ [3] 选择牺牲缓冲区
    │           │   GetVictimBuffer()
    │           │   │
    │           │   ├─ 使用Clock-Sweep算法
    │           │   │   ├─ 遍历缓冲区环
    │           │   │   ├─ 跳过被pin的缓冲区
    │           │   │   ├─ 递减usage_count
    │           │   │   └─ 选择usage_count=0的缓冲区
    │           │   │
    │           │   ├─ 如果缓冲区是脏的:
    │           │   │   └─ FlushBuffer()
    │           │   │       ├─ 写入磁盘
    │           │   │       └─ 清除脏标志
    │           │   │
    │           │   └─ 返回空闲缓冲区
    │           │
    │           ├─ [4] 从磁盘读取
    │           │   │
    │           │   ├─ smgrread()
    │           │   │   ├─ 打开文件
    │           │   │   ├─ lseek(blocknum * BLCKSZ)
    │           │   │   └─ read(buffer, BLCKSZ)
    │           │   │
    │           │   └─ 复制到缓冲区
    │           │
    │           ├─ [5] 插入哈希表
    │           │   └─ BufTableInsert(BufferTag, buffer_id)
    │           │
    │           └─ [6] Pin缓冲区
    │               └─ PinBuffer(buffer_id)
    │
    └─ 返回 Buffer

缓冲池数据结构:

共享缓冲池:
┌───────────────────────────────────────────────────┐
│              Shared Buffer Pool                   │
├───────────────────────────────────────────────────┤
│                                                   │
│ Buffer Descriptors (BufferDesc[])                │
│ ┌─────────────┬─────────────┬─────────────┐      │
│ │ BufferDesc  │ BufferDesc  │ BufferDesc  │ ...  │
│ │   tag       │   tag       │   tag       │      │
│ │   buf_id    │   buf_id    │   buf_id    │      │
│ │   refcount  │   refcount  │   refcount  │      │
│ │   usage_cnt │   usage_cnt │   usage_cnt │      │
│ │   flags     │   flags     │   flags     │      │
│ └─────────────┴─────────────┴─────────────┘      │
│          │            │            │              │
│          ↓            ↓            ↓              │
│ Buffer Blocks (实际数据页)                        │
│ ┌─────────────┬─────────────┬─────────────┐      │
│ │   8KB Page  │   8KB Page  │   8KB Page  │ ...  │
│ └─────────────┴─────────────┴─────────────┘      │
│                                                   │
│ Buffer Hash Table (查找)                         │
│ ┌──────────────────────────────────────┐         │
│ │  tag → buffer_id 映射                 │         │
│ └──────────────────────────────────────┘         │
└───────────────────────────────────────────────────┘

Buffer Descriptor状态:
├─ flags:
│   ├─ BM_DIRTY: 页面已修改
│   ├─ BM_VALID: 数据有效
│   ├─ BM_IO_IN_PROGRESS: I/O进行中
│   └─ BM_JUST_DIRTIED: 刚被修改
│
├─ refcount: 引用计数(pin数量)
└─ usage_count: 使用频率计数
```

### 5.2 锁管理

```
PostgreSQL锁层次:

[1] Spinlock (自旋锁)
    ├─ 用途: 保护共享内存中的临界区
    ├─ 持有时间: 微秒级
    ├─ 实现: CPU原子指令 (TAS, CAS)
    └─ 示例: 缓冲区描述符锁

[2] LWLock (轻量级锁)
    ├─ 用途: 保护共享数据结构
    ├─ 持有时间: 毫秒级
    ├─ 模式: Shared / Exclusive
    ├─ 实现: 基于自旋锁 + 等待队列
    └─ 示例:
        ├─ BufferMappingLock
        ├─ WALInsertLock
        └─ LockMgrLock

[3] Heavy Lock (重量级锁)
    ├─ 用途: 表锁、行锁
    ├─ 持有时间: 事务级
    ├─ 模式: 8种锁模式
    ├─ 死锁检测: 支持
    └─ 示例:
        ├─ AccessShareLock (SELECT)
        ├─ RowExclusiveLock (UPDATE/DELETE)
        └─ AccessExclusiveLock (DROP TABLE)

重量级锁模式:

锁模式                     与自己冲突  与其他冲突
────────────────────────────────────────────
AccessShareLock (读)          N         E
RowShareLock                  N         E
RowExclusiveLock (写)         N         S,SE,E
ShareUpdateExclusiveLock      Y         S,SE,E,SU
ShareLock                     N         RE,SE,E,SU
ShareRowExclusiveLock         Y         S,RE,SE,E,SU
ExclusiveLock                 Y         AS,RS,RE,S,SE,E,SU
AccessExclusiveLock (DDL)     Y         ALL

锁获取流程:

LockAcquire(locktag, lockmode)
    │
    ├─ [1] 计算锁标签哈希
    │   └─ tag = {dbid, relid, blocknum/tuple}
    │
    ├─ [2] 获取LockMgrLock (分区锁)
    │   └─ LWLockAcquire(LOCK_PARTITION)
    │
    ├─ [3] 在锁表中查找
    │   │
    │   └─ hash_search(LockMethodLockHash, tag)
    │       │
    │       ├─ [锁已存在]
    │       │   │
    │       │   ├─ 检查冲突
    │       │   │   ├─ LockCheckConflicts()
    │       │   │   │   └─ 与已有锁模式比较
    │       │   │   │
    │       │   │   ├─ [无冲突]
    │       │   │   │   ├─ GrantLock()
    │       │   │   │   ├─ 增加锁计数
    │       │   │   │   └─ 返回成功
    │       │   │   │
    │       │   │   └─ [有冲突]
    │       │   │       ├─ WaitOnLock()
    │       │   │       │   ├─ 加入等待队列
    │       │   │       │   ├─ 释放LockMgrLock
    │       │   │       │   ├─ 睡眠等待
    │       │   │       │   └─ 被唤醒后重试
    │       │   │       │
    │       │   │       └─ 死锁检测
    │       │   │           └─ DeadLockCheck()
    │       │   │
    │       └─ [锁不存在]
    │           ├─ 创建新锁对象
    │           ├─ GrantLock()
    │           └─ 插入锁表
    │
    └─ [4] 释放LockMgrLock
        └─ LWLockRelease(LOCK_PARTITION)
```

### 5.3 页面访问和元组获取

```
页面结构:

┌─────────────────────────────────────────────────────┐
│                    Page (8KB)                       │
├─────────────────────────────────────────────────────┤
│ PageHeader (24 bytes)                               │
│   ├─ pd_lsn: LSN (WAL日志位置)                      │
│   ├─ pd_checksum: 校验和                            │
│   ├─ pd_flags: 标志                                 │
│   ├─ pd_lower: 空闲空间开始                         │
│   ├─ pd_upper: 空闲空间结束                         │
│   └─ pd_special: 特殊空间指针                       │
├─────────────────────────────────────────────────────┤
│ Item Pointers (行指针数组)                           │
│   [0] → (offset, length)                            │
│   [1] → (offset, length)                            │
│   [2] → (offset, length)                            │
│   ...                                               │
├─────────────────────────────────────────────────────┤
│ Free Space (空闲空间)                                │
├─────────────────────────────────────────────────────┤
│ Tuples (从后向前存储)                                │
│   ┌──────────────────┐                              │
│   │ HeapTupleHeader  │  ← 元组3                     │
│   │ + 数据           │                              │
│   ├──────────────────┤                              │
│   │ HeapTupleHeader  │  ← 元组2                     │
│   │ + 数据           │                              │
│   ├──────────────────┤                              │
│   │ HeapTupleHeader  │  ← 元组1                     │
│   │ + 数据           │                              │
│   └──────────────────┘                              │
└─────────────────────────────────────────────────────┘

元组获取流程:

heap_fetch(relation, snapshot, tid)
    │
    ├─ [1] 从TID获取页面号和偏移
    │   ├─ blocknum = ItemPointerGetBlockNumber(tid)
    │   └─ offnum = ItemPointerGetOffsetNumber(tid)
    │
    ├─ [2] 读取页面
    │   └─ buffer = ReadBuffer(relation, blocknum)
    │       └─ (经过缓冲池)
    │
    ├─ [3] 锁定缓冲区
    │   └─ LockBuffer(buffer, BUFFER_LOCK_SHARE)
    │
    ├─ [4] 获取页面指针
    │   └─ page = BufferGetPage(buffer)
    │
    ├─ [5] 获取行指针
    │   └─ lp = PageGetItemId(page, offnum)
    │
    ├─ [6] 获取元组指针
    │   └─ tuple = (HeapTupleHeader) PageGetItem(page, lp)
    │
    ├─ [7] 检查元组状态
    │   ├─ LP_UNUSED → 未使用
    │   ├─ LP_NORMAL → 正常
    │   ├─ LP_REDIRECT → 重定向(HOT)
    │   └─ LP_DEAD → 已死
    │
    ├─ [8] MVCC可见性检查
    │   └─ HeapTupleSatisfiesVisibility(tuple, snapshot)
    │       ├─ 检查xmin/xmax
    │       └─ 与快照比较
    │
    ├─ [9] 解锁缓冲区
    │   └─ LockBuffer(buffer, BUFFER_LOCK_UNLOCK)
    │
    └─ [10] 返回元组
        └─ (保持buffer pinned)
```

---

## 6. MVCC可见性检查

### 6.1 可见性判断详解

```
MVCC可见性检查算法:

HeapTupleSatisfiesMVCC(tuple, snapshot)
(src/backend/access/heap/heapam_visibility.c)

输入:
  - tuple: HeapTupleHeader
  - snapshot: 查询快照

算法:
┌────────────────────────────────────────────────┐
│ Step 1: 检查元组是否已被删除                    │
└────────────────────────────────────────────────┘
if (tuple->t_infomask & HEAP_XMAX_INVALID):
    # xmax无效，元组未被删除
    goto check_xmin

if (tuple->t_infomask & HEAP_XMAX_IS_MULTI):
    # xmax是MultiXact，复杂情况
    检查MultiXact成员的可见性

xmax = HeapTupleHeaderGetRawXmax(tuple)

if (TransactionIdIsCurrentTransactionId(xmax)):
    # 被当前事务删除
    if (tuple->t_infomask & HEAP_XMAX_COMMITTED):
        return False  # 不可见
    else:
        return False  # 不可见（在当前事务内已删除）

if (tuple->t_infomask & HEAP_XMAX_COMMITTED):
    # xmax已提交
    return False  # 元组已被删除，不可见

if (tuple->t_infomask & HEAP_XMAX_INVALID):
    # xmax已回滚
    goto check_xmin  # 检查xmin

if (TransactionIdIsInProgress(xmax)):
    # 删除事务正在进行
    goto check_xmin  # 元组对当前查询可见

if (TransactionIdDidCommit(xmax)):
    # xmax已提交（检查CLOG）
    SetHintBits(tuple, HEAP_XMAX_COMMITTED)
    return False  # 不可见

else:
    # xmax已回滚
    SetHintBits(tuple, HEAP_XMAX_INVALID)
    goto check_xmin

┌────────────────────────────────────────────────┐
│ Step 2: 检查元组插入是否可见                    │
└────────────────────────────────────────────────┘
check_xmin:

if (tuple->t_infomask & HEAP_XMIN_INVALID):
    # xmin无效（元组已回滚）
    return False

xmin = HeapTupleHeaderGetRawXmin(tuple)

if (TransactionIdIsCurrentTransactionId(xmin)):
    # 被当前事务插入
    if (tuple->t_infomask & HEAP_XMIN_COMMITTED):
        return True  # 可见
    elif (tuple->t_infomask & HEAP_XMIN_INVALID):
        return False  # 不可见
    else:
        return True  # 可见（在当前事务内插入）

if (tuple->t_infomask & HEAP_XMIN_COMMITTED):
    # xmin已提交
    return True  # 可见

if (tuple->t_infomask & HEAP_XMIN_INVALID):
    # xmin已回滚
    return False

if (TransactionIdIsInProgress(xmin)):
    # 插入事务正在进行
    return False  # 不可见

if (TransactionIdDidCommit(xmin)):
    # xmin已提交（检查CLOG）
    SetHintBits(tuple, HEAP_XMIN_COMMITTED)
    return True  # 可见

else:
    # xmin已回滚
    SetHintBits(tuple, HEAP_XMIN_INVALID)
    return False

┌────────────────────────────────────────────────┐
│ 辅助函数                                        │
└────────────────────────────────────────────────┘

XidInMVCCSnapshot(xid, snapshot):
    # 检查XID是否在快照中

    if (xid < snapshot->xmin):
        # XID在最老活动事务之前
        return False  # 已提交且对快照可见

    if (xid >= snapshot->xmax):
        # XID在快照之后
        return True  # 对快照不可见

    # 在xmin和xmax之间，检查活动列表
    for i in range(snapshot->xcnt):
        if (xid == snapshot->xip[i]):
            return True  # 在快照时活动

    return False  # 已提交且对快照可见
```

### 6.2 快照创建

```
快照创建流程:

GetSnapshotData(snapshot)
(src/backend/utils/time/snapmgr.c)
    │
    ├─ [1] 获取锁
    │   └─ LWLockAcquire(ProcArrayLock, LW_SHARED)
    │
    ├─ [2] 获取下一个XID
    │   └─ xmax = ReadNextTransactionId()
    │
    ├─ [3] 扫描进程数组
    │   │
    │   └─ for each proc in ProcArray:
    │       │
    │       ├─ 跳过自己
    │       ├─ 跳过没有XID的进程
    │       ├─ 跳过Prepare但未提交的事务
    │       │
    │       └─ xid = proc->xid
    │           │
    │           ├─ 添加到活动列表
    │           │   snapshot->xip[count++] = xid
    │           │
    │           └─ 更新xmin
    │               if (xid < xmin):
    │                   xmin = xid
    │
    ├─ [4] 设置快照字段
    │   ├─ snapshot->xmin = xmin
    │   ├─ snapshot->xmax = xmax
    │   ├─ snapshot->xcnt = count
    │   └─ snapshot->curcid = currentCommandId
    │
    └─ [5] 释放锁
        └─ LWLockRelease(ProcArrayLock)

快照类型:

1. MVCC Snapshot (常规查询)
   └─ 事务开始时创建，整个事务使用

2. Self Snapshot (DDL)
   └─ 只能看到当前事务的修改

3. Dirty Snapshot (系统目录)
   └─ 可以看到正在进行的事务

4. Historic Snapshot (逻辑复制)
   └─ 历史一致性快照
```

---

## 7. 后台进程协作

### 7.1 后台进程交互图

```
PostgreSQL后台进程协作:

┌─────────────────────────────────────────────────────┐
│                    Postmaster                       │
│                   (主进程)                           │
└──────┬──────────────────────────────────────────────┘
       │
       │ fork()
       │
       ├──→ [Backend Processes] (多个)
       │     ├─ 处理客户端查询
       │     ├─ 生成WAL记录
       │     ├─ 修改共享缓冲池
       │     └─ 获取/释放锁
       │          ↓ ↓ ↓
       │    ┌─────────────────────┐
       │    │  Shared Buffer Pool  │
       │    │  ┌───┬───┬───┬───┐  │
       │    │  │ P │ P │ P │ P │  │
       │    │  │ a │ a │ a │ a │  │
       │    │  │ g │ g │ g │ g │  │
       │    │  │ e │ e │ e │ e │  │
       │    │  └───┴───┴───┴───┘  │
       │    └─────────────────────┘
       │          ↑        ↓
       │          │        │
       ├──→ [Background Writer]
       │     ├─ 扫描脏页
       │     ├─ 写出部分脏页
       │     └─ 减轻检查点压力
       │          ↓
       ├──→ [Checkpointer]
       │     ├─ 定期检查点
       │     ├─ 写出所有脏页
       │     ├─ fsync数据文件
       │     └─ 更新控制文件
       │          ↓
       │    ┌─────────────────────┐
       │    │    Data Files        │
       │    │   (base/oid/relid)  │
       │    └─────────────────────┘
       │
       ├──→ [WAL Writer]
       │     ├─ 从WAL缓冲区读取
       │     ├─ 写入WAL文件
       │     └─ fsync WAL
       │          ↓
       │    ┌─────────────────────┐
       │    │   WAL Files          │
       │    │   (pg_wal/)         │
       │    └─────────────────────┘
       │          ↓
       ├──→ [WAL Archiver]
       │     ├─ 检测完成的WAL
       │     ├─ 调用archive_command
       │     └─ 复制到归档位置
       │          ↓
       │    ┌─────────────────────┐
       │    │  Archive Location   │
       │    └─────────────────────┘
       │
       ├──→ [Autovacuum Launcher]
       │     ├─ 监控统计信息
       │     ├─ 决定VACUUM目标
       │     └─ 启动Worker
       │          ↓
       ├──→ [Autovacuum Workers] (多个)
       │     ├─ VACUUM表
       │     ├─ 清理死元组
       │     └─ 更新FSM/VM
       │
       ├──→ [Stats Collector]
       │     ├─ 接收统计消息
       │     ├─ 聚合统计数据
       │     └─ 写入统计文件
       │          ↓
       │    ┌─────────────────────┐
       │    │  pg_stat/ files     │
       │    └─────────────────────┘
       │
       └──→ [Logical Repl Launcher]
             └─→ [Subscription Workers]
                 ├─ 连接到发布者
                 ├─ 接收逻辑变更
                 └─ 应用到本地

进程间通信:
  ├─ 共享内存: 缓冲池、锁表、统计信息
  ├─ 信号: SIGUSR1用于进程间通知
  ├─ Latch: 轻量级的等待/唤醒机制
  └─ 文件: WAL文件、数据文件
```

### 7.2 检查点流程

```
检查点 (Checkpoint) 流程:

触发条件:
  ├─ 定时触发 (checkpoint_timeout)
  ├─ WAL量触发 (max_wal_size)
  └─ 手动触发 (CHECKPOINT命令)

CreateCheckPoint()
(src/backend/access/transam/xlog.c)
    │
    ├─ [Phase 1: 准备]
    │   ├─ 记录检查点开始
    │   ├─ 获取CheckpointLock
    │   └─ 记录检查点起始WAL位置
    │
    ├─ [Phase 2: 刷新缓冲]
    │   │
    │   └─ BufferSync()
    │       │
    │       ├─ 扫描所有缓冲区
    │       ├─ 找出所有脏页
    │       ├─ 按(表空间, 关系, 块号)排序
    │       └─ 批量写出
    │           │
    │           └─ for each dirty buffer:
    │               ├─ FlushBuffer()
    │               │   ├─ smgrwrite()
    │               │   └─ 不fsync（延后）
    │               └─ 清除BM_DIRTY标志
    │
    ├─ [Phase 3: Sync数据文件]
    │   │
    │   └─ ProcessSyncRequests()
    │       │
    │       └─ for each data file:
    │           └─ fsync(fd)
    │
    ├─ [Phase 4: 写检查点WAL记录]
    │   │
    │   ├─ XLogBeginInsert()
    │   ├─ XLogRegisterData(checkpoint_record)
    │   │   ├─ nextXID
    │   │   ├─ nextOID
    │   │   ├─ redo位置
    │   │   └─ 时间戳
    │   └─ XLogInsert(RM_XLOG_ID, XLOG_CHECKPOINT)
    │
    ├─ [Phase 5: 刷新WAL]
    │   └─ XLogFlush(checkpoint_redo)
    │
    ├─ [Phase 6: 更新控制文件]
    │   │
    │   └─ UpdateControlFile()
    │       ├─ 写入新的检查点位置
    │       ├─ 写入状态信息
    │       └─ fsync(pg_control)
    │
    └─ [Phase 7: 清理]
        ├─ RemoveOldXlogFiles()  # 删除旧WAL
        ├─ 释放CheckpointLock
        └─ 记录检查点完成

检查点WAL记录:
┌─────────────────────────────────┐
│  XLOG_CHECKPOINT Record         │
├─────────────────────────────────┤
│ redo: WAL重做起始位置            │
│ nextXID: 下一个事务ID            │
│ nextOID: 下一个对象ID            │
│ time: 检查点时间                 │
│ oldestXID: 最老的XID            │
│ oldestActiveXID: 最老活动XID    │
│ ...                             │
└─────────────────────────────────┘
```

---

## 总结

本文档详细说明了PostgreSQL的程序执行流程，包括：

1. **服务器启动**：从main()到Postmaster，共享内存初始化，后台进程启动
2. **客户端连接**：连接建立、认证、会话初始化
3. **查询处理**：Parser → Analyzer → Rewriter → Planner → Executor完整流水线
4. **事务管理**：事务生命周期、MVCC实现、WAL日志
5. **存储访问**：缓冲池管理、锁管理、页面和元组访问
6. **MVCC可见性**：详细的可见性检查算法
7. **后台进程**：各后台进程的协作和检查点流程

这些流程共同构成了PostgreSQL高效、可靠的数据库系统。

---

**文档版本**: 1.0
**分析日期**: 2025-11-06
**PostgreSQL版本**: 开发版 (基于最新master分支)
**代码库路径**: /home/user/postgres
