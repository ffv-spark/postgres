# PostgreSQL Program Execution Flows - Comprehensive Analysis

**Analysis Depth:** Very Thorough  
**Analysis Date:** 2025-11-06  
**Version:** Based on PostgreSQL Development Branch

---

## 1. SERVER STARTUP FLOW

### 1.1 Entry Point: src/backend/main/main.c

The `main()` function in `src/backend/main/main.c` (lines 70-237) is the universal entry point for all PostgreSQL server processes. All server processes (postmaster, backend, standalone, bootstrap) begin execution here.

```
main(int argc, char *argv[])
├─ Platform-specific initialization (startup_hacks)
├─ Process status display setup (save_ps_display_args)
├─ Memory and error system initialization
│  ├─ MyProcPid = getpid()
│  ├─ MemoryContextInit()
│  └─ set_stack_base()
├─ Locale initialization (LC_COLLATE, LC_CTYPE, LC_MESSAGES, etc.)
├─ Standard option parsing (--help, --version, --describe-config)
├─ Root privilege check (check_root)
└─ Dispatch to appropriate main function based on first argument:
   ├─ DISPATCH_CHECK → BootstrapModeMain()
   ├─ DISPATCH_BOOT → BootstrapModeMain()
   ├─ DISPATCH_FORKCHILD → SubPostmasterMain()
   ├─ DISPATCH_DESCRIBE_CONFIG → GucInfoMain()
   ├─ DISPATCH_SINGLE → PostgresSingleUserMain()
   └─ DISPATCH_POSTMASTER → PostmasterMain() [DEFAULT]
```

**Key Variables:**
- `progname`: Program name from argv[0]
- `dispatch_option`: Determines which mode to run in
- `MyProcPid`: Process ID of current process
- `reached_main`: Flag to track if main() was reached

### 1.2 Postmaster Initialization: src/backend/postmaster/postmaster.c

The `PostmasterMain()` function (starting at line 493) is the core initialization routine for the database server.

#### 1.2.1 Initialization Sequence

```
PostmasterMain(int argc, char *argv[])
├─ STAGE 1: Process Setup
│  ├─ InitProcessGlobals()
│  ├─ PostmasterPid = MyProcPid
│  ├─ IsPostmasterEnvironment = true
│  └─ Initialize Win32 signals (if on Windows)
│
├─ STAGE 2: Memory Context Setup
│  ├─ umask(PG_MODE_MASK_OWNER) - Set restrictive permissions
│  ├─ PostmasterContext = AllocSetContextCreate(TopMemoryContext, ...)
│  └─ MemoryContextSwitchTo(PostmasterContext)
│
├─ STAGE 3: Installation Path Setup
│  └─ getInstallationPaths(argv[0])
│
├─ STAGE 4: Signal Handler Setup
│  ├─ pqinitmask() - Initialize signal mask
│  ├─ sigprocmask(SIG_SETMASK, &BlockSig, NULL) - Block signals
│  ├─ pqsignal(SIGHUP, handle_pm_reload_request_signal)
│  ├─ pqsignal(SIGTERM/SIGINT/SIGQUIT, handle_pm_shutdown_request_signal)
│  ├─ pqsignal(SIGUSR1, handle_pm_pmsignal_signal)
│  ├─ pqsignal(SIGCHLD, handle_pm_child_exit_signal)
│  ├─ InitializeWaitEventSupport()
│  ├─ InitProcessLocalLatch()
│  ├─ Ignore SIGTTIN, SIGTTOU (terminal signals)
│  ├─ Ignore SIGXFSZ (disk full)
│  └─ sigprocmask(SIG_SETMASK, &UnBlockSig, NULL) - Unblock signals
│
├─ STAGE 5: GUC Options Initialization
│  └─ InitializeGUCOptions()
│
├─ STAGE 6: Command Line Option Parsing
│  ├─ Parse -B (shared_buffers)
│  ├─ Parse -D (data directory)
│  ├─ Parse -p (port)
│  ├─ Parse -h (listen addresses)
│  ├─ Parse -k (socket directory)
│  ├─ Parse other configuration options
│  └─ Reset getopt() state (optind = 1)
│
├─ STAGE 7: Configuration File Loading
│  ├─ SelectConfigFiles(userDoption, progname)
│  ├─ Load postgresql.conf
│  └─ Load recovery.conf (if needed)
│
├─ STAGE 8: Data Directory Validation
│  ├─ checkDataDir()
│  ├─ checkControlFile()
│  └─ ChangeToDataDir()
│
├─ STAGE 9: Configuration Validation
│  ├─ Validate superuser_reserved_connections + reserved_connections < max_connections
│  ├─ Validate WAL archival settings
│  ├─ Validate WAL streaming (max_wal_senders)
│  ├─ CheckDateTokenTables()
│  └─ Check for invalid GUC combinations
│
└─ STAGE 10: Server Main Loop
   └─ ServerLoop() - Main event loop
```

#### 1.2.2 Shared Memory Initialization

The shared memory setup occurs in `ServerLoop()` and related functions:

```
Shared Memory Setup:
├─ InitShmemAccess() - Initialize shared memory access
├─ CreateOrAttachShmemStructs() - Create/attach shared memory structures
│  ├─ PGSharedMemoryCreate()
│  │  ├─ Allocate shared memory segment
│  │  ├─ Set size based on shared_buffers, max_connections, etc.
│  │  └─ Request layout of shared memory structures
│  ├─ CreateProcArray() - Create process array
│  ├─ CreateLockFile() - Create shared lock structures
│  ├─ CreateBufferPool() - Create shared buffer pool
│  │  ├─ Allocate buffers
│  │  ├─ Initialize buffer descriptors
│  │  └─ Initialize buffer management structures
│  ├─ CreateXLogControlData() - Initialize WAL control structures
│  ├─ CreateSubTransactionData() - Initialize subtransaction structures
│  ├─ CreateMultiXactData() - Initialize multi-transaction structures
│  ├─ CreateCLogControlData() - Initialize transaction status log
│  ├─ CreateDistributedTransactionStateData() - For distributed transactions
│  └─ CreateShmemIndexes() - Create hash tables in shared memory
```

### 1.3 Postmaster State Machine

The postmaster uses a state machine to manage server state:

```
typedef enum PMState {
    PM_INIT,                      // Postmaster initializing
    PM_STARTUP,                   // Waiting for startup process
    PM_RECOVERY,                  // Archive recovery mode
    PM_HOT_STANDBY,              // Hot standby mode
    PM_RUN,                       // Normal running state
    PM_STOP_BACKENDS,            // Stopping remaining backends
    PM_WAIT_BACKENDS,            // Waiting for backends to exit
    PM_WAIT_XLOG_SHUTDOWN,       // Waiting for checkpointer shutdown checkpoint
    PM_WAIT_XLOG_ARCHIVAL,       // Waiting for archiver and WAL senders
    PM_WAIT_IO_WORKERS,          // Waiting for I/O workers
    PM_WAIT_CHECKPOINTER,        // Waiting for checkpointer shutdown
    PM_WAIT_DEAD_END,            // Waiting for dead-end children
    PM_NO_CHILDREN                // All important children exited
} PMState;
```

### 1.4 Background Process Spawning

The `ServerLoop()` function spawns background processes:

```
LaunchMissingBackgroundProcesses()
├─ Startup Process (StartupPMChild)
│  ├─ Reads control file
│  ├─ Performs WAL recovery if needed
│  ├─ Establishes recovery_ready state
│  └─ Transitions to PM_RUN state
├─ Background Writer (BgWriterPMChild)
│  ├─ Scans buffer pool for dirty pages
│  ├─ Writes pages to disk
│  └─ Offloads write work from normal backends
├─ Checkpointer (CheckpointerPMChild)
│  ├─ Performs periodic checkpoints
│  ├─ Creates consistency points for recovery
│  └─ Manages checkpoint metadata
├─ WAL Writer (WalWriterPMChild)
│  ├─ Writes WAL buffers to disk
│  ├─ Manages WAL flushing
│  └─ Handles synchronous replication waits
├─ WAL Receiver (WalReceiverPMChild) [if replication enabled]
│  ├─ Receives WAL from primary
│  ├─ Writes WAL to local disk
│  └─ Manages standby recovery
├─ Autovacuum Launcher (AutoVacLauncherPMChild)
│  ├─ Monitors tables for vacuum needs
│  ├─ Spawns autovacuum workers
│  └─ Manages vacuum scheduling
├─ WAL Archiver (PgArchPMChild) [if archiving enabled]
│  ├─ Copies WAL files to archive location
│  ├─ Manages WAL archive directory
│  └─ Handles archive errors
├─ Syslogger (SysLoggerPMChild) [if logging to files]
│  ├─ Manages log file rotation
│  ├─ Collects log messages
│  └─ Handles stderr redirection
├─ Logical Replication Launcher (LogicalLauncherPMChild) [if logical replication enabled]
│  ├─ Manages logical replication workers
│  ├─ Applies changes from publishers
│  └─ Handles subscription management
└─ Background Workers (Custom)
   ├─ User-defined background worker processes
   ├─ Extension-based background jobs
   └─ Custom monitoring/maintenance tasks
```

### 1.5 Connection Acceptance Loop

```
ServerLoop()
├─ LOOP: Main event loop
│  ├─ Check for postmaster signals (SIGTERM, SIGHUP, etc.)
│  ├─ Check for child process exits
│  ├─ Launch missing background processes
│  ├─ WaitEventSetWait() - Wait for events on listen sockets
│  │  └─ Timeout-based: 100ms
│  ├─ Process ready events:
│  │  ├─ If listen socket ready:
│  │  │  ├─ Accept client connection
│  │  │  └─ BackendStartup()
│  │  │     ├─ Check if accepting connections (canAcceptConnections)
│  │  │     ├─ Fork new backend process
│  │  │     │  └─ Set dispatch option to DISPATCH_FORKCHILD
│  │  │     └─ Parent: Add to child list
│  │  └─ If postmaster pipe ready:
│  │     └─ Process parent process messages
│  └─ Handle postmaster state transitions
└─ Exit loop on shutdown signal
```

---

## 2. CLIENT CONNECTION FLOW

### 2.1 Connection Request Handling

When a client connects, the postmaster's `BackendStartup()` function (in postmaster.c) handles the connection:

```
BackendStartup(ClientSocket *client_sock)
├─ Check connection acceptance status
│  └─ canAcceptConnections(BACKEND_TYPE_NORMAL)
│     ├─ CAC_OK - Accept normal connections
│     ├─ CAC_WAITBACKEND - Wait for backends to exit
│     └─ CAC_SHUTDOWN - Reject connection
│
├─ Fork new child process
│  ├─ fork() → child process inherits socket
│  ├─ Parent (Postmaster):
│  │  ├─ Close child-side socket
│  │  ├─ Add PMChild entry to ActiveChildList
│  │  ├─ Increment backend count
│  │  └─ Continue accepting connections
│  │
│  └─ Child (Backend Process):
│     ├─ Close postmaster listen sockets
│     ├─ Set dispatch option to DISPATCH_FORKCHILD
│     └─ Call PostgresMain() [or SubPostmasterMain() on EXEC_BACKEND]
```

### 2.2 Authentication Process

The backend process begins execution with authentication in `PostgresMain()` (tcop/postgres.c):

```
PostgresMain(int argc, char *argv[], const char *username)
├─ INITIALIZATION PHASE
│  ├─ InitPostgres() - Initialize backend
│  │  ├─ BaseInit()
│  │  │  ├─ InitProcessGlobals()
│  │  │  ├─ InitBufferPoolAccess()
│  │  │  │  ├─ AdjustNBuffers() - Set backend buffer counts
│  │  │  │  └─ InitLocalBufferPool()
│  │  │  ├─ InitProcessAccessInfo()
│  │  │  ├─ InitStorageEngineBackend()
│  │  │  └─ InitAccessMethods()
│  │  ├─ InitMultiXactState()
│  │  ├─ InitUnloggedRelationSync()
│  │  ├─ InitPostgresRelationCache() - Load relcache
│  │  ├─ InitTempTableNamespace() - Create temp schema
│  │  ├─ InitializeMaxBackendId()
│  │  └─ EmitInvalidationMessage() - Process SI messages
│  │
│  ├─ ClientAuthentication(Port *port)
│  │  ├─ Get authentication info from pg_hba.conf
│  │  ├─ Set AuthenticationTimeout (typically 60 seconds)
│  │  ├─ Match connection against pg_hba.conf rules
│  │  │  ├─ Check network/host
│  │  │  ├─ Check database
│  │  │  ├─ Check user
│  │  │  └─ Get authentication method
│  │  ├─ Call authentication method handler
│  │  │  ├─ AUTH_REQ_OK → Client already approved
│  │  │  ├─ AUTH_REQ_PASSWORD → Send password challenge
│  │  │  ├─ AUTH_REQ_MD5 → Send MD5 password challenge
│  │  │  ├─ AUTH_REQ_SCRAM_SHA_256 → SCRAM authentication
│  │  │  ├─ AUTH_REQ_GSS → GSSAPI authentication
│  │  │  ├─ AUTH_REQ_SSPI → Windows SSPI authentication
│  │  │  ├─ AUTH_REQ_CERT → SSL certificate authentication
│  │  │  ├─ AUTH_REQ_SASL → SASL authentication
│  │  │  └─ AUTH_REQ_LDAP → LDAP authentication
│  │  ├─ Read authentication response from client
│  │  ├─ Verify credentials
│  │  │  ├─ Check user exists in pg_authid
│  │  │  ├─ Verify password hash
│  │  │  └─ Check user not disabled
│  │  └─ Return authentication result
│  │
│  └─ SetSessionUserId(userId) - Set authenticated user
│
├─ SESSION INITIALIZATION PHASE
│  ├─ Create session memory context
│  ├─ Initialize GUC variables for session
│  ├─ InitCatalogCache() - Load system catalogs
│  ├─ QueryCancelPending = false
│  ├─ InterruptPending = false
│  └─ ProcSignalInit()
│
└─ MAIN QUERY LOOP (see section 3)
```

### 2.3 Backend Process Creation Data Flow

```
Backend Process Lifecycle:
┌─────────────────────────────────────────┐
│ Postmaster receives connection on socket│
└──────────────────┬──────────────────────┘
                   │
                   ▼
┌─────────────────────────────────────────┐
│ BackendStartup(ClientSocket)            │
│ ├─ Check if can accept connections      │
│ └─ Call fork() to create child process  │
└──────────────────┬──────────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
        ▼ (Parent)            ▼ (Child)
   Postmaster            PostgresMain()
   ├─ Track child       ├─ InitPostgres()
   ├─ Update counters   ├─ ClientAuthentication()
   └─ Continue loop     ├─ Session initialization
                        └─ Query processing loop
```

### 2.4 Session Initialization

```
Session Initialization Sequence:
├─ Memory Context Setup
│  ├─ Create TopMemoryContext (if not exists)
│  ├─ Create MessageContext (for processing messages)
│  ├─ Create PostmasterContext (if in postmaster)
│  ├─ Create CacheContext (for caching)
│  └─ Create PortalContext (for query portals)
│
├─ Relation Cache Setup
│  ├─ Load system catalog cache
│  ├─ Initialize relation descriptors for:
│  │  ├─ pg_class
│  │  ├─ pg_attribute
│  │  ├─ pg_index
│  │  ├─ pg_constraint
│  │  └─ Other system tables
│  └─ Register cache invalidation functions
│
├─ Lock and Transaction Setup
│  ├─ Initialize LOCALLOCK hash table
│  ├─ Initialize transaction state
│  ├─ Set default isolation level
│  └─ Initialize snapshot manager
│
└─ Connection-Specific Setup
   ├─ Set CurrentUserId = authenticated user
   ├─ Set CurrentSchemaId
   ├─ Initialize search_path
   └─ Set session GUC parameters from ALTER SYSTEM/SET
```

---

## 3. QUERY PROCESSING FLOW

### 3.1 Query Reception

The main query loop in `PostgresMain()` reads queries from the client:

```
Main Query Processing Loop in postgres.c:
├─ LOOP: while (!ignore_till_sync)
│  │
│  ├─ start_xact_command()
│  │  ├─ Start transaction if not already started
│  │  └─ Set command state
│  │
│  ├─ ReadCommand(inBuf) - Read next command from client
│  │  ├─ Check if interactive (stdin) or socket backend
│  │  │
│  │  ├─ SocketBackend(inBuf)
│  │  │  ├─ HOLD_CANCEL_INTERRUPTS()
│  │  │  ├─ pq_startmsgread()
│  │  │  ├─ pq_getbyte() - Read message type
│  │  │  │  Message types:
│  │  │  │  ├─ 'Q' (PqMsg_Query) - Simple query
│  │  │  │  ├─ 'P' (PqMsg_Parse) - Parse statement
│  │  │  │  ├─ 'B' (PqMsg_Bind) - Bind parameters
│  │  │  │  ├─ 'D' (PqMsg_Describe) - Describe statement
│  │  │  │  ├─ 'E' (PqMsg_Execute) - Execute prepared statement
│  │  │  │  ├─ 'S' (PqMsg_Sync) - Sync (extended protocol)
│  │  │  │  ├─ 'C' (PqMsg_Close) - Close prepared statement
│  │  │  │  ├─ 'H' (PqMsg_CopyResponse) - Copy data
│  │  │  │  ├─ 'd' (PqMsg_CopyData) - Copy data
│  │  │  │  ├─ 'c' (PqMsg_CopyDone) - Copy done
│  │  │  │  ├─ 'X' (PqMsg_Terminate) - Disconnect
│  │  │  │  ├─ 'F' (PqMsg_FunctionCall) - Fast-path function
│  │  │  │  └─ 'f' (PqMsg_CopyFail) - Copy error
│  │  │  ├─ pq_getmessage() - Read message body based on length
│  │  │  └─ pq_endmsgread() - Mark end of message
│  │  │
│  │  └─ InteractiveBackend(inBuf) [if interactive]
│  │     ├─ Print "backend> " prompt
│  │     ├─ Read from stdin until newline
│  │     └─ Return PqMsg_Query
│  │
│  └─ Process command based on message type
```

### 3.2 Query Parser Stage (SQL → AST)

The parser converts raw SQL text into an abstract syntax tree:

```
Query Parsing Flow:

Input: SQL text "SELECT * FROM users WHERE id = 1;"

├─ pg_parse_query(sourceText) [tcop/postgres.c]
│  └─ raw_parser(sourceText) [parser/parser.c]
│     ├─ lex_scan_setup() - Initialize lexer
│     ├─ base_yyparse() - Bison parser main function
│     │  ├─ Lexer tokenizes input:
│     │  │  ├─ SELECT token
│     │  │  ├─ * (STAR)
│     │  │  ├─ FROM
│     │  │  ├─ identifier "users"
│     │  │  ├─ WHERE
│     │  │  ├─ identifier "id"
│     │  │  ├─ = operator
│     │  │  └─ numeric constant 1
│     │  │
│     │  └─ Parser builds AST recursively:
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
└─ Return: RawStmt
   ├─ stmt: SelectStmt (tree)
   └─ stmt_location, stmt_len (text locations)
```

**Parser Data Structures:**

The parser operates on these key node types:
- `SelectStmt` - SELECT statements
- `InsertStmt` - INSERT statements
- `UpdateStmt` - UPDATE statements
- `DeleteStmt` - DELETE statements
- `FuncCall` - Function calls
- `Expr` and related expression nodes
- `RangeVar` - Table references

### 3.3 Query Analyzer/Rewriter Stage (Semantic Analysis)

The analyzer transforms the raw parse tree into a Query object and handles views/rules:

```
Query Analysis and Rewriting:

Raw AST → parse_analyze_fixedparams() [parser/analyze.c]
│
├─ transformTopLevelStmt(pstate, RawStmt)
│  └─ transformStmt(pstate, stmt_node)
│     ├─ transformSelectStmt() [for SELECT]
│     │  ├─ transformFromClause() - Process FROM clause
│     │  │  ├─ Resolve relation names
│     │  │  ├─ Check table/schema permissions (ACLCHECK_SELECT)
│     │  │  ├─ Load relation definitions
│     │  │  ├─ Add RTEs (RangeTblEntry) to pstate->p_rtable
│     │  │  └─ Process JOINs
│     │  │
│     │  ├─ transformTargetList() - Process SELECT list
│     │  │  ├─ Resolve column references
│     │  │  ├─ Expand * to actual columns
│     │  │  ├─ Type coerce expressions
│     │  │  └─ Create ResTarget for each output column
│     │  │
│     │  ├─ transformWhereClause() - Process WHERE
│     │  │  ├─ transformExpr() recursively
│     │  │  ├─ Type coercion for predicates
│     │  │  └─ Function name resolution
│     │  │
│     │  ├─ transformGroupByClause() - Process GROUP BY
│     │  ├─ transformHavingClause() - Process HAVING
│     │  ├─ transformSortClause() - Process ORDER BY
│     │  ├─ transformLimitClause() - Process LIMIT
│     │  └─ Return Query node
│     │
│     ├─ transformInsertStmt() [for INSERT]
│     │  ├─ Resolve target table
│     │  ├─ Check INSERT permission
│     │  ├─ Match column list to values
│     │  ├─ Apply default values for missing columns
│     │  ├─ Type coerce values
│     │  └─ Process ON CONFLICT clause
│     │
│     ├─ transformUpdateStmt() [for UPDATE]
│     │  ├─ Resolve target table
│     │  ├─ Check UPDATE permission
│     │  ├─ Resolve column references in SET clause
│     │  ├─ Type coercion for new values
│     │  ├─ Process WHERE clause
│     │  └─ Process FROM clause
│     │
│     └─ transformDeleteStmt() [for DELETE]
│        ├─ Resolve target table
│        ├─ Check DELETE permission
│        ├─ Process WHERE clause
│        └─ Process USING clause
│
├─ HandleView Rules (rewrite.c)
│  ├─ If table is a view:
│  │  ├─ Find rules for operation
│  │  ├─ Apply query rewrite rules
│  │  │  ├─ Insert rules
│  │  │  ├─ Update rules
│  │  │  ├─ Delete rules
│  │  │  └─ Select rules
│  │  └─ Replace original query with rule query
│  │
│  └─ If table has rule system:
│     └─ Apply appropriate rules
│
└─ Return: Query (analyzed query tree)
   ├─ commandType (CMD_SELECT, CMD_INSERT, etc.)
   ├─ rtable (Range Table - list of relations)
   ├─ targetList (output columns)
   ├─ jointree (WHERE and FROM combined)
   ├─ groupClause, havingClause, sortClause
   └─ Other metadata (permission checks needed, etc.)
```

### 3.4 Query Optimizer Stage (AST → Execution Plan)

The optimizer converts the Query into an optimized execution plan:

```
Query Planning:

Query → planner(Query *parse, const char *query_string, ...)
│                                         [optimizer/planner.c]
│
├─ standard_planner(root, parse, ...)
│  │
│  ├─ PlannerGlobal Structure Creation
│  │  ├─ Allocate PlannerGlobal
│  │  ├─ Initialize CTEs
│  │  └─ Setup parallelism info
│  │
│  ├─ relation_parse_tree_to_relids(parse) - Identify relations
│  │  ├─ Build list of relation OIDs
│  │  └─ Initialize relation statistics
│  │
│  ├─ PlannerInfo (root) Creation
│  │  ├─ Initialize RelOptInfo for each base relation
│  │  ├─ Compute table statistics (rows, pages)
│  │  ├─ Calculate index costs
│  │  └─ Build equivalence classes for join conditions
│  │
│  ├─ Preprocessing Phase
│  │  ├─ preprocess_expression() - Simplify expressions
│  │  ├─ preprocess_qual_conditions() - Process WHERE
│  │  ├─ Build EquivalenceClass for predicates
│  │  └─ Identify join order dependencies
│  │
│  ├─ Plan Generation Phase
│  │  ├─ For each command type (SELECT, INSERT, etc.):
│  │  │
│  │  ├─ SELECT planning:
│  │  │  ├─ join_search_one_level() - Build join tree
│  │  │  │  ├─ Generate all possible join orderings
│  │  │  │  ├─ Cost each join path:
│  │  │  │  │  ├─ SeqScan cost = pages / seq_page_cost
│  │  │  │  │  ├─ IndexScan cost = random I/O cost + CPU
│  │  │  │  │  ├─ BitmapScan cost = index + heap scan
│  │  │  │  │  ├─ NestedLoop cost = outer_rows * inner_cost
│  │  │  │  │  ├─ HashJoin cost = build_hash + probe
│  │  │  │  │  └─ MergeJoin cost = sort + merge
│  │  │  │  ├─ Prune high-cost paths (keep_useful_pathlist)
│  │  │  │  └─ Return best Path for each relation set
│  │  │  │
│  │  │  ├─ Final path selection
│  │  │  │  ├─ Select best paths for:
│  │  │  │  │  ├─ Rows returned (for LIMIT)
│  │  │  │  │  ├─ ORDER BY requirements
│  │  │  │  │  └─ GROUP BY requirements
│  │  │  │  ├─ Create_plan() - Convert path to plan node
│  │  │  │  └─ Attach physical sort nodes if needed
│  │  │  │
│  │  │  └─ Post-planning steps:
│  │  │     ├─ Add subplans for subqueries
│  │  │     ├─ Add CTEs
│  │  │     ├─ Add initplans for uncorrelated subqueries
│  │  │     └─ Mark parallelizable nodes
│  │  │
│  │  ├─ INSERT planning:
│  │  │  ├─ Plan SELECT (if INSERT ... SELECT)
│  │  │  ├─ Add ModifyTable node on top
│  │  │  └─ Check ON CONFLICT logic
│  │  │
│  │  ├─ UPDATE planning:
│  │  │  ├─ Plan scan of target table with WHERE
│  │  │  ├─ Add ModifyTable node
│  │  │  ├─ Set update exprs and targetlist
│  │  │  └─ Check trigger requirements
│  │  │
│  │  └─ DELETE planning:
│  │     ├─ Plan scan of target table with WHERE
│  │     ├─ Add ModifyTable node
│  │     └─ Check trigger requirements
│  │
│  ├─ Plan Finalization
│  │  ├─ create_plan() - Build final Plan tree
│  │  ├─ Optimize subplans
│  │  ├─ Propagate constraints
│  │  ├─ Estimate plan costs
│  │  └─ Plan parallelization options
│  │
│  └─ Return PlannedStmt
│     ├─ commandType
│     ├─ planTree (root of plan tree)
│     ├─ rtable (range table)
│     ├─ result_relations
│     ├─ subplans
│     ├─ initPlan
│     ├─ targetList
│     └─ Cost estimates
│
└─ Output: PlannedStmt (execution plan)

Example Plan Tree for SELECT * FROM users WHERE id = 1:

PlannedStmt
└─ planTree: SeqScan
   ├─ scanRelationOid: users table OID
   ├─ filter: id = 1 (applied during execution)
   └─ targetList: [id, name, email, ...]
```

### 3.5 Executor Stage (Plan Execution)

The executor executes the plan tree to retrieve/modify data:

```
Query Execution:

ProcessQuery(query_desc) [tcop/postgres.c]
│
├─ CreateQueryDesc() - Already done before
│  ├─ query_desc.plannedstmt = plan
│  ├─ query_desc.snapshot = active_snapshot
│  └─ query_desc.sourceText = query_string
│
└─ PortalStart/PortalRun (portal.c)
   │
   ├─ ExecutorStart(queryDesc, options) [executor/execMain.c]
   │  │
   │  ├─ CreateExecutorState()
   │  │  ├─ Create EState (Executor State) structure
   │  │  ├─ es_query_cxt - Query memory context
   │  │  ├─ es_epq_active - Eager Plan Queue state
   │  │  ├─ Initialize result relations (for UPDATE/INSERT/DELETE)
   │  │  └─ Setup trigger context
   │  │
   │  ├─ InitPlan(queryDesc, eflags)
   │  │  ├─ ExecInitNode(planTree, estate)
   │  │  │  ├─ Recursively initialize plan nodes bottom-up
   │  │  │  │
   │  │  │  ├─ For SeqScan:
   │  │  │  │  ├─ Open relation using relation manager
   │  │  │  │  ├─ Initialize table scan descriptor
   │  │  │  │  │  └─ table_beginscan() [tableam.h]
   │  │  │  │  ├─ Allocate result slots
   │  │  │  │  └─ Return SeqScanState
│  │  │  │
│  │  │  ├─ For IndexScan:
│  │  │  │  ├─ Open base relation
│  │  │  │  ├─ Open index
│  │  │  │  ├─ Build index scan keys from plan
│  │  │  │  ├─ Initialize index scan descriptor
│  │  │  │  │  └─ index_beginscan() [indexam.h]
│  │  │  │  └─ Allocate result slots
│  │  │  │
│  │  │  ├─ For NestedLoopJoin:
│  │  │  │  ├─ Init outer plan
│  │  │  │  ├─ Init inner plan
│  │  │  │  ├─ Initialize join filter
│  │  │  │  └─ Allocate result slots
│  │  │  │
│  │  │  ├─ For HashJoin:
│  │  │  │  ├─ Init outer plan
│  │  │  │  ├─ Init inner plan
│  │  │  │  ├─ ExecHashTableCreate() - Create hash table
│  │  │  │  ├─ ExecHashTableBuild() - Build hashtable from inner
│  │  │  │  └─ Allocate result slots
│  │  │  │
│  │  │  ├─ For Aggregate:
│  │  │  │  ├─ Initialize aggregate state
│  │  │  │  ├─ Initialize aggregate functions
│  │  │  │  ├─ Create aggregate buffer
│  │  │  │  └─ Init grouped aggregate state if GROUP BY
│  │  │  │
│  │  │  ├─ For Sort:
│  │  │  │  ├─ Init child node
│  │  │  │  ├─ Create sort descriptor
│  │  │  │  └─ Allocate sort buffer
│  │  │  │
│  │  │  ├─ For Limit:
│  │  │  │  ├─ Init child node
│  │  │  │  └─ Initialize count/offset
│  │  │  │
│  │  │  ├─ For ModifyTable (INSERT/UPDATE/DELETE):
│  │  │  │  ├─ Initialize target relation
│  │  │  │  ├─ Check permissions
│  │  │  │  ├─ Initialize result relation
│  │  │  │  ├─ Setup trigger context
│  │  │  │  ├─ Initialize child scan plan
│  │  │  │  └─ Setup constraint checking
│  │  │  │
│  │  │  └─ [Recursively for all plan node types]
│  │  │
│  │  └─ Setup after-trigger context for UPDATE/INSERT/DELETE
│  │
│  ├─ Register snapshot for transaction isolation
│  ├─ Setup query instrumentation if needed
│  └─ Return (queryDesc now ready for execution)
│
├─ ExecutorRun(queryDesc, direction, count) [executor/execMain.c]
│  │
│  ├─ standard_ExecutorRun(queryDesc, direction, count)
│  │  │
│  │  └─ ExecutePlan(queryDesc, ...)
│  │     │
│  │     └─ LOOP: Fetch tuples from plan
│  │        │
│  │        ├─ ExecProcNode(planstate) - Get next tuple from plan
│  │        │  ├─ Case SeqScan:
│  │        │  │  ├─ table_scan_getnextslot() [tableam.h]
│  │        │  │  │  ├─ ReadBuffer(heaprel, pageno)
│  │        │  │  │  │  └─ Load page into buffer cache [storage/bufmgr.c]
│  │        │  │  │  │     ├─ Lookup in buffer hash table
│  │        │  │  │  │     ├─ If not found, evict LRU page
│  │        │  │  │  │     ├─ Read from disk
│  │        │  │  │  │     └─ Pin buffer (increment refcount)
│  │        │  │  │  ├─ Check tuple visibility
│  │        │  │  │  │  └─ HeapTupleSatisfiesMVCC(tuple, snapshot, buffer)
│  │        │  │  │  │     ├─ Check XMIN (insert transaction)
│  │        │  │  │  │     ├─ Check XMAX (delete transaction)
│  │        │  │  │  │     ├─ Check if XID is in active snapshot
│  │        │  │  │  │     └─ Return true if visible, false otherwise
│  │        │  │  │  ├─ ReleaseBuffer()
│  │        │  │  │  └─ If visible, fill slot and return
│  │        │  │  └─ Continue until all tuples scanned
│  │        │  │
│  │        │  ├─ Case IndexScan:
│  │        │  │  ├─ index_getnext() [indexam.h]
│  │        │  │  │  ├─ Scan index for matching tuples
│  │        │  │  │  ├─ Return TID from index
│  │        │  │  │  └─ Count as iteration
│  │        │  │  ├─ heap_fetch() [heapam.h]
│  │        │  │  │  ├─ ReadBuffer(heap_rel, block_from_tid)
│  │        │  │  │  ├─ OffsetNumber offset = ItemPointerGetOffsetNumber(tid)
│  │        │  │  │  ├─ Get tuple from page at offset
│  │        │  │  │  └─ Check visibility
│  │        │  │  └─ Fill slot with tuple
│  │        │  │
│  │        │  ├─ Case NestedLoopJoin:
│  │        │  │  ├─ Get tuple from outer plan
│  │        │  │  ├─ LOOP: Match with inner plan tuples
│  │        │  │  │  ├─ Reset inner scan
│  │        │  │  │  ├─ Get first tuple from inner
│  │        │  │  │  ├─ LOOP: Check join condition for each inner tuple
│  │        │  │  │  │  ├─ ExecQual(join_filter, context)
│  │        │  │  │  │  │  ├─ Evaluate expression
│  │        │  │  │  │  │  ├─ For each column access:
│  │        │  │  │  │  │  │  ├─ Slot access (already in memory)
│  │        │  │  │  │  │  │  ├─ Type coercion if needed
│  │        │  │  │  │  │  │  └─ Operator evaluation
│  │        │  │  │  │  │  └─ Return boolean result
│  │        │  │  │  │  ├─ If match, create result tuple and return
│  │        │  │  │  │  └─ Otherwise continue
│  │        │  │  │  └─ Get next inner tuple
│  │        │  │  └─ When inner exhausted, get next outer tuple
│  │        │  │
│  │        │  ├─ Case HashJoin:
│  │        │  │  ├─ Probe hash table with outer tuple
│  │        │  │  ├─ For each matching hash bucket entry:
│  │        │  │  │  ├─ Check join condition
│  │        │  │  │  └─ If match, return result tuple
│  │        │  │  └─ Otherwise get next outer tuple
│  │        │  │
│  │        │  ├─ Case Aggregate:
│  │        │  │  ├─ Accumulate aggregate values
│  │        │  │  ├─ For each input row:
│  │        │  │  │  ├─ Call aggregate advance function
│  │        │  │  │  │  └─ e.g., aggregate_sum(state, value)
│  │        │  │  │  └─ Update state
│  │        │  │  ├─ If GROUP BY:
│  │        │  │  │  ├─ Use aggregate hash table
│  │        │  │  │  ├─ Key = group values
│  │        │  │  │  └─ Value = aggregate state
│  │        │  │  └─ Return final aggregate when done
│  │        │  │
│  │        │  ├─ Case Sort:
│  │        │  │  ├─ Fetch all input tuples
│  │        │  │  ├─ Sort using qsort or tuplesort
│  │        │  │  │  └─ Use comparison function based on ORDER BY
│  │        │  │  └─ Return tuples in sorted order
│  │        │  │
│  │        │  └─ [And so on for all node types]
│  │        │
│  │        ├─ ExecProject(targetlist, slot)
│  │        │  ├─ Create output tuple from plan tuple
│  │        │  ├─ Evaluate computed columns
│  │        │  ├─ Apply projections and coercions
│  │        │  └─ Return result slot
│  │        │
│  │        ├─ Send tuple to client via DestReceiver
│  │        │  ├─ receiveSlot(slot)
│  │        │  │  ├─ Convert to wire format (text/binary)
│  │        │  │  ├─ Send 'D' (DataRow) message
│  │        │  │  │  ├─ message type: D
│  │        │  │  │  ├─ message length
│  │        │  │  │  ├─ number of columns
│  │        │  │  │  └─ for each column:
│  │        │  │  │     ├─ column length (or -1 for NULL)
│  │        │  │  │     └─ column value bytes
│  │        │  │  └─ Queue in send buffer
│  │        │  │
│  │        │  └─ pq_flush() - Flush to network when buffer full
│  │        │
│  │        ├─ Check for interrupt signals
│  │        │  ├─ CHECK_FOR_INTERRUPTS()
│  │        │  ├─ Process SIGTERM, SIGINT
│  │        │  ├─ Cancel query if requested
│  │        │  └─ Rollback and cleanup
│  │        │
│  │        └─ If count reached or end of scan, exit loop
│  │
│  └─ Return (all requested tuples fetched)
│
├─ ExecutorFinish(queryDesc) [executor/execMain.c]
│  ├─ For aggregates without GROUP BY:
│  │  ├─ Finalize aggregate function
│  │  │  └─ aggregate_final_func(state) → result value
│  │  └─ Return final aggregate row
│  │
│  ├─ For UPDATE/INSERT/DELETE:
│  │  ├─ Execute before/after triggers that are deferred
│  │  │  └─ For each deferred trigger:
│  │  │     ├─ Call trigger function
│  │  │     └─ Execute trigger SQL
│  │  │
│  │  ├─ Flush constraint checks
│  │  └─ Execute AFTER triggers
│  │
│  └─ Cleanup executor state
│
└─ ExecutorEnd(queryDesc) [executor/execMain.c]
   ├─ ExecEndPlan(planstate, estate)
   │  └─ Recursively close all plan nodes
   │     ├─ Close scan relations
   │     ├─ Close indexes
   │     ├─ Free hash tables
   │     ├─ Free sort buffers
   │     └─ Free allocated structures
   │
   ├─ Free EState
   ├─ Unregister snapshot
   └─ Clean up executor memory
```

### 3.6 Result Return to Client

```
Sending Results to Client:

For SELECT queries:
├─ Send ReadyForQuery message to client
│  ├─ 'Z' message type
│  ├─ Status: 'I' (idle), 'T' (transaction), 'E' (error)
│  └─ Sent after all rows
│
└─ Format of result rows:
   ├─ RowDescription message (sent first):
   │  ├─ Message type: T
   │  ├─ Field count
   │  └─ For each field:
   │     ├─ Column name
   │     ├─ Table OID
   │     ├─ Column number
   │     ├─ Type OID
   │     ├─ Type size
   │     ├─ Type modifier
   │     └─ Format code (0=text, 1=binary)
   │
   └─ DataRow messages (for each result row):
      ├─ Message type: D
      ├─ Field count
      └─ For each column:
         ├─ Length (or -1 for NULL)
         └─ Value in output format (text or binary)

For UPDATE/INSERT/DELETE:
├─ Return row count: "INSERT 0 5" (oid and count)
└─ If RETURNING clause:
   ├─ Send RowDescription
   ├─ Send DataRow for each returned row
   └─ Send CommandComplete
```

---

## 4. TRANSACTION PROCESSING

### 4.1 Transaction Start/Commit/Rollback

Located in `src/backend/access/transam/xact.c`:

```
Transaction Lifecycle:

start_xact_command() [tcop/postgres.c]
├─ If not in transaction:
│  ├─ BeginTransactionBlock()
│  │  ├─ Set blockState = TBLOCK_INPROGRESS
│  │  ├─ Set TransState = TRANS_START
│  │  └─ No actual work until first query
│  │
│  └─ StartTransactionCommand() [xact.c]
│     ├─ start_transaction() [xact.c]
│     │  ├─ Assign TransactionId if needed
│     │  │  └─ GetNewTransactionId(isSubTransaction)
│     │  │     ├─ Read xidgen variable from shared memory
│     │  │     ├─ Assign next XID
│     │  │     └─ Update shared xidgen
│     │  │
│     │  ├─ Record as in-progress
│     │  │  └─ PGPROC->xid = MyTransactionId
│     │  │
│     │  ├─ Initialize transaction snapshot
│     │  │  └─ GetSnapshotData(&SnapshotData)
│     │  │     ├─ Read active XIDs from PGPROC array
│     │  │     ├─ Find global xmin (oldest active XID)
│     │  │     ├─ Find xmax (next to assign)
│     │  │     └─ Mark transactions in progress
│     │  │
│     │  ├─ Initialize transaction context
│     │  │  └─ Create TransactionStateData
│     │  │     ├─ fullTransactionId = {epoch, xid}
│     │  │     ├─ subTransactionId = 1
│     │  │     ├─ state = TRANS_INPROGRESS
│     │  │     └─ blockState = TBLOCK_INPROGRESS
│     │  │
│     │  └─ Log transaction start in WAL (if needed)
│     │     └─ LogCurrentTransactionState()
│     │
│     └─ Emit InvalidationMessage to other backends
│        └─ Signal() other backends about new transaction
│
├─ In transaction: Set xact_started = true
└─ Set CommandId for this command
   └─ GetCurrentCommandId(canAssign)


finish_xact_command() [tcop/postgres.c]
├─ CommitTransactionCommand() [xact.c]
│  └─ CommitTransaction()
│     ├─ Pre-commit phase:
│     │  ├─ Run BEFORE COMMIT triggers
│     │  └─ Flush PREPARE statements
│     │
│     ├─ Commit phase:
│     │  ├─ Log WAL commit record
│     │  │  └─ LogXactAbortRecord()
│     │  │     ├─ Create XLOG_XACT_COMMIT record
│     │  │     ├─ Include timestamp
│     │  │     ├─ Include transaction XIDs
│     │  │     └─ Include commit info
│     │  │
│     │  ├─ Call wal_writer to flush WAL
│     │  │  └─ XLogFlush(currentLogEndPos)
│     │  │     ├─ Write buffered WAL to disk
│     │  │     ├─ fsync() if synchronous_commit = on
│     │  │     └─ Update LSN (Log Sequence Number)
│     │  │
│     │  ├─ Mark transaction as committed
│     │  │  └─ TransactionIdCommitTree(xid, nchildren, children)
│     │  │     ├─ Write to pg_xact
│     │  │     │  └─ TransactionIdSetStatus(xid, TRANSACTION_COMMITTED)
│     │  │     └─ Write subtransaction XIDs
│     │  │        └─ TransactionIdSetStatus(child_xid, TRANSACTION_COMMITTED)
│     │  │
│     │  ├─ Remove from PGPROC active list
│     │  │  └─ PGPROC->xid = InvalidTransactionId
│     │  │
│     │  └─ Flush caches if needed
│     │
│     ├─ Post-commit phase:
│     │  ├─ Execute AFTER COMMIT triggers
│     │  ├─ Invalidate relation cache entries
│     │  │  └─ CardinalitySet() - Update planner cardinality info
│     │  ├─ Release locks
│     │  │  └─ LockReleaseAll(DEFAULT_LOCKMETHOD)
│     │  ├─ Close portals
│     │  ├─ Free transaction memory
│     │  └─ Emit commit message to replication
│     │
│     └─ Update transaction counters
│        └─ Increment PostgreSQL's internal stats
│
└─ Return to idle state

---

ROLLBACK Process:

RollbackTransactionCommand()
└─ RollbackTransaction()
   ├─ Pre-abort phase:
   │  ├─ Log abort record in WAL
   │  │  └─ XactLogAbortRecord()
   │  │     ├─ XLOG_XACT_ABORT record
   │  │     └─ Include transaction XIDs
   │  │
   │  ├─ Mark transaction as aborted
   │  │  └─ TransactionIdAbortTree(xid, nchildren)
   │  │     └─ TransactionIdSetStatus(xid, TRANSACTION_ABORTED)
   │  │
   │  └─ Flush WAL
   │     └─ XLogFlush()
   │
   ├─ Abort phase:
   │  ├─ Execute BEFORE ROLLBACK triggers
   │  ├─ Rollback savepoints
   │  ├─ Undo DML operations:
   │  │  ├─ For INSERT: Remove inserted rows
   │  │  ├─ For UPDATE: Restore previous versions
   │  │  └─ For DELETE: Restore deleted rows
   │  │
   │  ├─ Release locks
   │  │  └─ LockReleaseAll()
   │  │     ├─ Local locks first
   │  │     ├─ Then shared locks
   │  │     └─ Signal waiting processes
   │  │
   │  └─ Cleanup state
   │
   ├─ Post-abort phase:
   │  ├─ Execute AFTER ROLLBACK triggers
   │  ├─ Reset transaction state
   │  ├─ Invalidate snapshots
   │  └─ Free transaction memory
   │
   └─ Return to idle state
```

### 4.2 Subtransactions (SAVEPOINT)

```
SAVEPOINT Management:

SAVEPOINT sp_name
└─ BeginInternalSubTransaction(NULL)
   ├─ Create new TransactionStateData
   │  └─ parent = previous TransactionState
   │
   ├─ Assign SubTransactionId
   │  └─ ++currentSubTransactionId
   │
   ├─ Create subtransaction context
   │  └─ Create memory context for subtransaction
   │
   ├─ Log in WAL (optional)
   │  └─ XLogStartSubTransaction(subxid)
   │
   └─ Push onto transaction state stack

RELEASE SAVEPOINT sp_name
└─ ReleaseCurrentSubTransaction()
   ├─ Validate savepoint still exists
   ├─ Commit subtransaction changes
   │  └─ Merge changes into parent transaction
   ├─ Log release in WAL
   └─ Pop from transaction state stack

ROLLBACK TO SAVEPOINT sp_name
└─ RollbackAndReleaseCurrentSubTransaction()
   ├─ Abort subtransaction operations
   ├─ Undo DML changes
   ├─ Restore state from before savepoint
   ├─ Log abort in WAL
   └─ Pop from transaction state stack
```

### 4.3 MVCC Implementation

MVCC (Multi-Version Concurrency Control) in PostgreSQL allows readers and writers to coexist:

```
MVCC Core Concepts:

Transaction ID (XID):
├─ 32-bit identifier assigned at transaction start
├─ Incremented across server lifetime
├─ Stored in tuple headers:
│  ├─ t_infomask: Commit status bits
│  ├─ t_xmin: Insert transaction ID
│  ├─ t_xmax: Delete/update transaction ID
│  └─ t_cid: Command ID
│
└─ Epoch:
   ├─ 32-bit epoch counter
   ├─ Tracks XID wraparound
   └─ Combined with XID for full transaction ID


Tuple Visibility Rules:

HeapTupleSatisfiesMVCC(tuple, snapshot)
├─ Check insert transaction (t_xmin):
│  ├─ If t_xmin == my_xid:
│  │  └─ I inserted it → VISIBLE
│  ├─ Else if t_xmin in snapshot.xip (active list):
│  │  └─ Insert is in-progress → NOT_VISIBLE
│  ├─ Else if t_xmin < snapshot.xmin:
│  │  └─ Insert definitely committed → Check delete
│  ├─ Else if t_xmin >= snapshot.xmax:
│  │  └─ Insert definitely not started → NOT_VISIBLE
│  └─ Else (t_xmin between xmin and xmax):
│     └─ Check pg_xact status
│
└─ Check delete transaction (t_xmax):
   ├─ If t_xmax == InvalidXid:
   │  └─ Not deleted → VISIBLE
   ├─ Else if t_xmax == my_xid:
   │  └─ I deleted it → NOT_VISIBLE
   ├─ Else if t_xmax in snapshot.xip:
   │  └─ Delete is in-progress → VISIBLE
   ├─ Else if t_xmax < snapshot.xmin:
   │  └─ Delete definitely committed → NOT_VISIBLE
   ├─ Else if t_xmax >= snapshot.xmax:
   │  └─ Delete definitely not started → VISIBLE
   └─ Else (t_xmax between xmin and xmax):
      └─ Check pg_xact status


Snapshot Structure:

Snapshot {
    SnapshotType: SNAPSHOT_MVCC, SNAPSHOT_SELF, SNAPSHOT_DIRTY, SNAPSHOT_TOAST
    xmin: Smallest XID in transaction.in_progress
    xmax: Highest XID + 1 seen
    xip[]: Array of active transaction XIDs
    xcnt: Number of active XIDs
    xip_base: Base XID for compression (PG13+)
    subxip[]: Array of active subtransaction XIDs (if any)
    subxcnt: Number of active subxids
    suboverflowed: Whether subxids list is incomplete
    takenDuringRecovery: Taken during crash recovery
}

Snapshot Creation:

GetSnapshotData() [src/backend/access/transam/procarray.c]
├─ Lock PGPROC array
├─ Read all active transaction states from PGPROC
├─ Find:
│  ├─ xmin = smallest active XID
│  ├─ xmax = highest assigned XID + 1
│  └─ xip[] = array of all active XIDs
├─ Sort xip[] for binary search
├─ Unlock PGPROC array
└─ Return Snapshot

Example Snapshot State:
Before snapshot taken:
  PGPROC[0]: xid=100, running
  PGPROC[1]: xid=102, running
  PGPROC[2]: xid=103, running (subtransaction)
  LastAssignedXid = 103

Snapshot after GetSnapshotData():
  xmin = 100 (oldest active)
  xmax = 104 (next to assign)
  xip[] = {100, 102, 103}
  xcnt = 3

Now for tuple visibility:
  xmin=95  → visible (older than xmin)
  xmin=100 → NOT visible (in xip)
  xmin=101 → visible (not in xip, >= xmin)
  xmin=103 → NOT visible (in xip)
  xmin=104 → NOT visible (>= xmax)
```

### 4.4 WAL (Write-Ahead Logging)

```
WAL Structure:

WAL Files:
├─ Location: $PGDATA/pg_wal/
├─ Filename: XXXXXXXXXXXXXXXX (16 hex digits = 8 bytes XLogFileId + 8 bytes XLogRecPtr)
├─ Size: 16 MB by default (wal_segment_size)
└─ Organization:
   ├─ Page 0: File header
   ├─ Pages 1-N: WAL records
   └─ Each page: 8KB (XLOG_BLCKSZ)

WAL Record Structure:

XLogRecord {
    xl_tot_len: Total record length
    xl_xid: Transaction ID
    xl_prev: Pointer to previous record
    xl_info: Type of record (insert, delete, etc.)
    xl_rmid: Resource manager ID (heap, index, etc.)
    xl_data_len: Length of data section
    ...data...
}

WAL Record Types (resource managers):
├─ RM_XACT_ID: Transaction records
│  ├─ XLOG_XACT_COMMIT
│  ├─ XLOG_XACT_ABORT
│  ├─ XLOG_XACT_PREPARE
│  └─ XLOG_XACT_ASSIGNMENT
├─ RM_HEAP_ID: Heap table records
│  ├─ XLOG_HEAP_INSERT
│  ├─ XLOG_HEAP_UPDATE
│  ├─ XLOG_HEAP_DELETE
│  ├─ XLOG_HEAP_HOT_UPDATE
│  └─ XLOG_HEAP_TRUNCATE
├─ RM_BTREE_ID: B-tree index records
│  ├─ XLOG_BTREE_INSERT_LEAF
│  ├─ XLOG_BTREE_INSERT_UPPER
│  ├─ XLOG_BTREE_SPLIT_L
│  └─ etc.
└─ [Other resource managers for different access methods]

WAL Insertion and Flushing:

Insert operation:
1. LogNewPage() or LogRecordData()
   ├─ Acquire WALInsertLock
   ├─ Write to WAL buffer in shared memory
   ├─ Update currrentLogEndPos
   └─ Release WALInsertLock

2. XLogFlush(requestPtr)
   ├─ Acquire WALFlushLock
   ├─ If synchronous_commit:
   │  ├─ fsync() WAL file to disk
   │  └─ Update lastFlushedPtr
   └─ Release WALFlushLock

Synchronous Commit Levels:
├─ off: Return immediately after buffer write
├─ local: Return after local fsync()
├─ remote_write: Return after replica writes to disk
├─ remote_apply: Return after replica applies
└─ on: Same as remote_apply (default)

WAL Format:
Page Header (32 bytes) | Record | Record | ... | Trailer (optional)

Each page starts with:
├─ Magic number (XLOG_PAGE_MAGIC)
├─ Page header info
├─ XLogFileHeader for first page in file
└─ Records follow
```

---

## 5. STORAGE ACCESS FLOW

### 5.1 Buffer Pool Interaction

The buffer pool is PostgreSQL's in-memory cache of disk pages:

```
Buffer Pool Architecture:

Shared Memory Buffer Pool:
├─ Allocation: shared_buffers parameter (default 128MB)
├─ Page size: 8KB (BLCKSZ)
├─ Number of buffers: shared_buffers / 8192
│
├─ Data Structure: BufferDesc array in shared memory
│  ├─ One BufferDesc per buffer
│  └─ Each BufferDesc contains:
│     ├─ tag: RelFileLocator {spcOid, dbOid, relOid, segNo}
│     ├─ freeNext, freePrev: Free list links
│     ├─ data: Pointer to buffer page data
│     ├─ refcount: Pin count
│     ├─ usageCount: LRU usage counter
│     ├─ flags: Dirty, I/O in progress, etc.
│     └─ spinlock: Protect header modifications
│
├─ Buffer Hash Table (buf_table.c)
│  ├─ dynahash table mapping tag → BufferDesc
│  ├─ Partitioned for scalability
│  └─ Protected by BufMappingLock
│
└─ Free List (freelist.c)
   ├─ List of unused buffers
   ├─ Protected by buffer_strategy_lock
   └─ Used for buffer replacement

Buffer Replacement Strategy (Clock Sweep):

nextVictimBuffer → [B0] [B1] [B2] [B3] [B4] ...
                     ↑ (clock hand cycles through)

Algorithm:
1. Lock buffer_strategy_lock
2. Select buffer at nextVictimBuffer
3. Advance nextVictimBuffer
4. Unlock buffer_strategy_lock
5. If buffer pinned or usageCount > 0:
   ├─ Decrement usageCount
   └─ Go to step 1 (try next buffer)
6. If buffer dirty:
   ├─ Write to disk (background writer usually does this)
7. Pin buffer and use
```

### 5.2 Page Access

```
ReadBuffer() [storage/bufmgr.c]
├─ Caller provided:
│  ├─ rel: Relation object
│  ├─ blockNum: Block number to read
│  └─ strategy: Buffer ring strategy (optional)
│
├─ ReadBufferExtended(rel, fork, blockNum, mode, strategy)
│  │
│  ├─ Compute RelFileLocator tag
│  │  └─ tag {spcOid, dbOid, relOid, segNo}
│  │
│  ├─ LockBufMappingPartition(tag)
│  │  └─ Acquire BufMappingLock for this tag's partition
│  │
│  ├─ buf_table.c: BufTableLookup(tag)
│  │  ├─ Hash table lookup
│  │  └─ Check if page already in cache
│  │
│  ├─ If found:
│  │  ├─ PinBuffer(bufHdr)
│  │  │  ├─ Increment refcount
│  │  │  ├─ Increment usageCount
│  │  │  └─ Update LRU info
│  │  ├─ UnlockBufMappingPartition()
│  │  └─ Return buffer
│  │
│  └─ If not found (cache miss):
│     ├─ UnlockBufMappingPartition()
│     ├─ StrategyGetBuffer(strategy) - Get victim buffer
│     │  ├─ Use clock-sweep algorithm
│     │  ├─ Evict if dirty (write to disk)
│     │  └─ Return unused buffer
│     │
│     ├─ LockBufMappingPartition(tag) - Reacquire lock
│     ├─ BufTableInsert() - Add to hash table
│     │  └─ Update buffer tag
│     │
│     ├─ UnlockBufMappingPartition()
│     ├─ ReadBuffer_Disk()
│     │  ├─ mdopen() - Open relation fork
│     │  ├─ mdread() - Read block from disk
│     │  │  └─ lseek() + read() system calls
│     │  └─ Mark page as loaded
│     │
│     └─ Return buffer

ReleaseBuffer(buffer) [storage/bufmgr.c]
├─ Get BufferDesc
├─ UnpinBuffer(bufHdr)
│  ├─ Decrement refcount
│  ├─ If refcount == 1 and someone waiting for cleanup lock:
│  │  └─ Signal condition variable
│  └─ Release spinlock
└─ Mark buffer as released

BufferAccess Flow:

Buffer Pin (ownership):
├─ Must pin buffer before accessing
├─ Can hold multiple pins on same buffer
├─ Cannot hold pins across transaction boundaries
└─ Pins protect buffer from eviction

Buffer Lock (content access):
├─ BUFFER_LOCK_SHARE (read access)
│  └─ Multiple readers allowed
├─ BUFFER_LOCK_EXCLUSIVE (write access)
│  └─ Exclusive access required
└─ Must hold pin before acquiring lock

Typical Access Pattern:
1. ReadBuffer(rel, blockno) → pin buffer
2. LockBuffer(buf, BUFFER_LOCK_SHARE) → lock for reading
3. Access buffer data: (HeapTupleHeader) BufferGetPage(buf)
4. UnlockBuffer(buf) → unlock
5. Continue accessing data (pin still held)
6. ReleaseBuffer(buf) → unpin buffer

Transaction Boundary:
└─ All pins must be released at transaction end
   └─ Called automatically in EndTransaction
```

### 5.3 Lock Acquisition

Locks in PostgreSQL come in multiple layers:

```
Lock Hierarchy:

1. Spinlocks (very short-term)
   ├─ Used for: Buffer headers, lock manager partitions
   ├─ Held for: Few instructions only
   ├─ Busy-wait mechanism
   └─ No deadlock detection

2. Lightweight Locks (LWLocks)
   ├─ Used for: Shared memory structure protection
   ├─ Modes: Shared, Exclusive
   ├─ Modes: Blocking on semaphore
   └─ No deadlock detection
   
   Common LWLocks:
   ├─ BufMappingLock: Protects buffer mapping hash table
   ├─ BufFreelistLock: Protects free buffer list
   ├─ LockMgrLock: Protects lock manager structures
   ├─ WALInsertLock: Protects WAL insertion
   ├─ WALFlushLock: Protects WAL flushing
   └─ [~100 other LWLocks]

3. Heavyweight Locks (Regular Locks)
   ├─ Used for: Table/relation/tuple locking
   ├─ Types: AccessShareLock, RowExclusiveLock, ExclusiveLock, etc.
   ├─ Modes: Explicit conflict checking
   └─ Features: Deadlock detection, automatic release
   
   Lock Types:
   ├─ AccessShareLock: SELECT (lowest)
   ├─ RowShareLock: SELECT FOR SHARE
   ├─ RowExclusiveLock: INSERT, UPDATE, DELETE
   ├─ ShareUpdateExclusiveLock: VACUUM, ANALYZE
   ├─ ShareLock: CREATE INDEX
   ├─ ShareRowExclusiveLock: 
   ├─ ExclusiveLock: LOCK TABLE
   └─ AccessExclusiveLock: ALTER TABLE, DROP (highest)

Relation Lock Acquisition:

LockRelationOid(relOid, lockMode)
├─ Lookup lock in LOCK hash table
├─ Check for conflicts with existing locks
├─ If no conflict:
│  ├─ Grant lock immediately
│  └─ Record in PROCLOCK structure
├─ If conflict:
│  ├─ Add to wait queue
│  ├─ Set alarm for deadlock detection
│  ├─ Sleep on semaphore
│  └─ Wake when lock granted or deadlock detected
└─ Return lock

Lock Conflict Matrix:
```

### 5.4 Tuple Visibility Checking

Tuple visibility is the core of MVCC:

```
Tuple Visibility Check Flow:

HeapTupleSatisfiesMVCC(htup, snapshot, buffer)
├─ Get tuple header: t_data = htup->t_data
│  ├─ t_infomask: Status bits
│  │  ├─ HEAP_XMIN_COMMITTED: Xmin is committed
│  │  ├─ HEAP_XMIN_INVALID: Xmin is invalid
│  │  ├─ HEAP_XMAX_COMMITTED: Xmax is committed
│  │  ├─ HEAP_XMAX_INVALID: Xmax is invalid
│  │  ├─ HEAP_XMAX_IS_MULTI: Xmax is multixact
│  │  ├─ HEAP_XMAX_IS_LOCKED_ONLY: Xmax is lock, not delete
│  │  └─ [Other flags]
│  ├─ t_xmin: Insert transaction ID
│  └─ t_xmax: Delete transaction ID or lock xid
│
├─ Check insert status:
│  ├─ Raw Xmin: HeapTupleHeaderGetRawXmin(tuple)
│  ├─ Step 1: Determine if insert is committed
│  │  ├─ If HEAP_XMIN_COMMITTED bit set:
│  │  │  └─ Yes, committed
│  │  ├─ Else if HEAP_XMIN_INVALID bit set:
│  │  │  └─ No, was aborted
│  │  ├─ Else if Xmin == my_xid:
│  │  │  └─ Yes, I'm still in this transaction
│  │  ├─ Else if TransactionIdIsInProgress(Xmin):
│  │  │  └─ No, still in progress
│  │  ├─ Else if XidInMVCCSnapshot(Xmin, snapshot):
│  │  │  └─ No, active when my snapshot was taken
│  │  ├─ Else if TransactionIdDidCommit(Xmin):
│  │  │  ├─ Yes, check pg_xact status
│  │  │  └─ Set HEAP_XMIN_COMMITTED hint bit
│  │  └─ Else:
│  │     ├─ No, assume aborted/crashed
│  │     └─ Set HEAP_XMIN_INVALID hint bit
│  │
│  └─ If insert not committed:
│     └─ RETURN: INVISIBLE
│
├─ Check delete status:
│  ├─ Raw Xmax: HeapTupleHeaderGetRawXmax(tuple)
│  ├─ Step 1: Determine if tuple was deleted
│  │  ├─ If HEAP_XMAX_INVALID bit set:
│  │  │  └─ No, not deleted
│  │  ├─ Else if HEAP_XMAX_IS_LOCKED_ONLY set:
│  │  │  └─ No, only locked, not deleted
│  │  ├─ Else if Xmax == my_xid:
│  │  │  └─ Yes, I deleted it
│  │  ├─ Else if TransactionIdIsInProgress(Xmax):
│  │  │  └─ No, delete is in progress
│  │  ├─ Else if XidInMVCCSnapshot(Xmax, snapshot):
│  │  │  └─ No, delete active when my snapshot taken
│  │  ├─ Else if TransactionIdDidCommit(Xmax):
│  │  │  ├─ Yes, committed delete
│  │  │  └─ Set HEAP_XMAX_COMMITTED hint bit
│  │  └─ Else:
│  │     └─ No, assume aborted/crashed
│  │
│  └─ If deleted and delete committed:
│     └─ RETURN: INVISIBLE
│
└─ RETURN: VISIBLE

Hint Bits:
├─ Optimization to avoid repeated pg_xact lookups
├─ Once transaction status known, set hint bit on tuple
├─ SetHintBits(tuple, buffer, infomask, xid)
│  ├─ Check if safe to set:
│  │  ├─ For commit bits: Check if WAL is flushed
│  │  └─ For abort bits: Always safe
│  ├─ tuple->t_infomask |= infomask
│  └─ MarkBufferDirtyHint(buffer, true)
└─ Next time tuple accessed, bits already set

Snapshot Functions:
├─ XidInMVCCSnapshot(xid, snapshot)
│  ├─ Binary search in snapshot.xip[]
│  └─ Return true if xid in active list
│
├─ TransactionIdIsInProgress(xid)
│  ├─ Check PGPROC array directly
│  └─ True if xid actively running
│
└─ TransactionIdDidCommit(xid)
   ├─ Check pg_xact log
   └─ True if committed
```

---

## 6. CONTROL FLOW DIAGRAMS (Text Format)

### 6.1 Server Startup Sequence Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│ PostgreSQL Server Startup Flow                                  │
└─────────────────────────────────────────────────────────────────┘

         start postgres
                │
                ▼
    ┌───────────────────────┐
    │ main() [main.c]       │
    │ - Check root          │
    │ - Init memory         │
    │ - Init locale         │
    └───────────┬───────────┘
                │
                ▼
    ┌───────────────────────┐
    │ PostmasterMain()      │
    │ [postmaster.c]        │
    │ - Init globals        │
    │ - Load config         │
    │ - Create shmem        │
    │ - Setup signals       │
    └───────────┬───────────┘
                │
                ▼
    ┌───────────────────────┐
    │ ServerLoop()          │
    │ Main event loop       │
    │ - Accept connections  │
    │ - Manage children     │
    │ - State machine       │
    └───────────┬───────────┘
                │
     ┌──────────┼──────────┐
     │          │          │
     ▼          ▼          ▼
 Startup    Background  Connection
 Process    Processes   Handlers
  ├─ Redo  ├─BgWriter   ├─Auth
  ├─ Init  ├─Checkpntr  ├─Backend
  └─ Ready └─Others     └─Loops
```

### 6.2 Query Processing Pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│ SQL Query Processing Pipeline                                   │
└─────────────────────────────────────────────────────────────────┘

Input: SELECT * FROM users WHERE age > 18;

    ┌──────────────────────────────────┐
    │ PARSING PHASE                    │
    │ String → Abstract Syntax Tree    │
    └────────────┬─────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │ Output: RawStmt                           │
    │ ├─ SelectStmt node                       │
    │ ├─ targetList (columns)                  │
    │ ├─ fromClause (tables)                   │
    │ └─ whereClause (predicates)              │
    └────────────┬──────────────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │ ANALYSIS PHASE                           │
    │ AST → Semantic Analysis + Rewriting      │
    └────────────┬──────────────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │ Output: Query                            │
    │ ├─ commandType (CMD_SELECT)              │
    │ ├─ rtable (range table with OIDs)        │
    │ ├─ targetList (resolved columns)         │
    │ ├─ jointree (processed WHERE)            │
    │ └─ permissions needed                    │
    └────────────┬──────────────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │ PLANNING PHASE                           │
    │ Query → Optimized Execution Plan         │
    │ ├─ Cost-based optimization               │
    │ ├─ Path enumeration                      │
    │ ├─ Join ordering                         │
    │ └─ Index selection                       │
    └────────────┬──────────────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │ Output: PlannedStmt                      │
    │ ├─ planTree (execution plan tree)        │
    │ ├─ relcache (relation info)              │
    │ └─ Cost estimates                        │
    │                                          │
    │ Example planTree:                        │
    │     Limit (if LIMIT clause)              │
    │       └─ Sort (if ORDER BY)              │
    │           └─ SeqScan on users           │
    │               └─ Filter (age > 18)       │
    └────────────┬──────────────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │ EXECUTION PHASE                          │
    │ Plan → Results                           │
    │ ├─ ExecutorStart()                       │
    │ ├─ ExecutorRun()                         │
    │ │  ├─ Open table/index                   │
    │ │  ├─ Scan pages                         │
    │ │  ├─ Check tuple visibility (MVCC)     │
    │ │  ├─ Apply filters                      │
    │ │  ├─ Sort if needed                     │
    │ │  └─ Return rows                        │
    │ ├─ ExecutorFinish()                      │
    │ └─ ExecutorEnd()                         │
    └────────────┬──────────────────────────────┘
                 │
    ┌────────────▼──────────────────────────────┐
    │ Output: Result Rows                      │
    │ ├─ RowDescription message                │
    │ ├─ DataRow messages                      │
    │ └─ CommandComplete message               │
    └──────────────────────────────────────────────┘
```

### 6.3 Connection Establishment Flow

```
┌──────────────────────────────────────────────────────┐
│ Client Connection Establishment                      │
└──────────────────────────────────────────────────────┘

Client                          Postmaster

  │                                │
  ├─ TCP connection ──────────────>│
  │                                │
  │                         Accept() call
  │                                │
  │                           fork()
  │                           ├─ Child (Backend)
  │                           └─ Parent (Postmaster)
  │                                │
  │<────── Welcome Message ────────┤ Backend
  │        (start-up params)        │
  │                                │
  │─ Start-up message ────────────>│
  │ (user, password, database)      │
  │                                │
  │                         Validate user
  │                         ClientAuthentication()
  │                                │
  │                         Check pg_hba.conf
  │                         Get auth method
  │                                │
  │<──── Auth request ─────────────┤
  │ (password, MD5, SCRAM, etc.)    │
  │                                │
  │─ Auth response ───────────────>│
  │ (password/credentials)          │
  │                                │
  │                         Verify credentials
  │                         Check pg_authid
  │                         Check permissions
  │                                │
  │<──── Auth success ─────────────┤
  │ (or error)                      │
  │                                │
  │                         InitPostgres()
  │                         Load relcache
  │                         Create temp schema
  │                                │
  │<──── ReadyForQuery ────────────┤
  │ (status = 'I' for idle)         │
  │                                │
  │─ Query ───────────────────────>│
  │                                │
  │                         Parse → Analyze
  │                         Plan → Execute
  │                                │
  │<─── RowDescription ────────────┤
  │<─── DataRow ───────────────────┤
  │<─── DataRow ───────────────────┤
  │<─── CommandComplete ───────────┤
  │<─── ReadyForQuery ─────────────┤
  │                                │
```

---

## 7. KEY DATA STRUCTURES

### 7.1 Core Execution Structures

```c
// Query Description - Passed through execution stages
typedef struct QueryDesc {
    char       *sourceText;        // SQL text
    CachedPlan *cplan;            // Cached plan
    PlannedStmt *plannedstmt;     // Execution plan
    SnapShot    snapshot;         // MVCC snapshot
    SnapShot    crosscheck_snapshot;
    CommandDest dest;             // Output destination
    ParamListInfo params;         // Bind parameters
    QueryEnvironment *queryEnv;
    Instrument *instrument_options;
    EState     *estate;           // Executor state (set by ExecutorStart)
    PlanState  *planstate;        // Plan state tree (set by ExecutorStart)
    bool        already_executed;
} QueryDesc;

// Executor State - Main execution context
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

// Plan State - Node-specific execution state
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

// Plan Node (tree node representing operation)
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

### 7.2 MVCC Related Structures

```c
// Snapshot - Defines visible transaction set
typedef struct SnapshotData {
    SnapshotType snapshot_type;
    TransactionId xmin;           // Oldest active XID
    TransactionId xmax;           // Next XID to assign
    TransactionId *xip;           // Active XIDs
    uint32      xcnt;             // Number of active XIDs
    TransactionId *subxip;        // Active subtransaction XIDs
    int32       subxcnt;          // Number of subtransaction XIDs
    bool        suboverflowed;    // Too many subtransactions to track
    bool        takenDuringRecovery;
    bool        copied;
    CommandId   curcid;           // Command ID for command-level visibility
    uint32      speculativeToken;
    XLogRecPtr  lsn;              // WAL position when snapshot taken
    struct SnapshotData *active_count_delta;
} SnapshotData;

// Transaction State
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

// Tuple Header - Metadata stored with each tuple
typedef struct HeapTupleHeaderData {
    union {
        HeapTupleFields t_heap;
        DatumTupleFields t_datum;
    } t_choice;
    ItemPointerData t_ctid;      // Location of tuple or its update
    uint16      t_infomask2;     // Flags
    uint16      t_infomask;      // Commit/Lock status flags
    uint8       t_hoff;          // Offset to user data
    bits8       t_bits[FLEXIBLE_ARRAY_MEMBER];
} HeapTupleHeaderData;
```

### 7.3 Buffer Pool Structures

```c
// Buffer Descriptor
typedef struct BufferDesc {
    BufferTag   tag;            // Identity of page
    int         freeNext;       // Next free buffer in free list
    int         freePrev;       // Previous free buffer
    uint32      refcount;       // Pin count
    uint32      usageCount;     // LRU usage counter
    BufFlags    flags;          // Dirty, I/O in progress, etc.
    uint16      wait_backend_pid;
    int         io_in_progress_lock;
    spin_delay_status delayStatus;
    BufferAccessStrategy ring;  // Buffer ring (if any)
    LocalBufferDesc *local_buf; // Local buffer desc (if local)
} BufferDesc;

// Buffer Tag - Page identifier
typedef struct BufferTag {
    RelFileLocator rlocator;    // Relation file identifier
    ForkNumber  forkNum;        // Relation fork
    BlockNumber blockNum;       // Block number
} BufferTag;

// Local Buffer Pool (per-backend)
typedef struct {
    Block       buffer;         // Actual page data
    BufferTag   tag;            // Page identifier
    int         flags;          // Pin count, dirty, etc.
    VacuumPhase vacuumPhase;    // For VACUUM operations
    bits8       extra;          // Extra flags
} LocalBufferDesc;
```

---

## 8. PERFORMANCE CONSIDERATIONS

### 8.1 Query Optimization Techniques

1. **Index Selection**: Planner chooses best index based on cost
2. **Join Ordering**: Optimizer determines optimal table join sequence
3. **Predicate Pushdown**: Filters pushed to earliest execution point
4. **Path Pruning**: High-cost paths eliminated
5. **Parallel Execution**: Plans marked for parallel workers when beneficial

### 8.2 MVCC Optimization

1. **Hint Bits**: Avoid repeated transaction status lookups
2. **Snapshot Caching**: Reuse snapshot across multiple operations
3. **Xmin Horizon Tracking**: Cleanup old tuple versions efficiently
4. **Lazy Freezing**: Defer expensive freezing operations
5. **Concurrent Vacuum**: Vacuum without blocking queries

### 8.3 Buffer Pool Optimization

1. **Buffer Ring Strategy**: Dedicated buffers for sequential scans
2. **Clock Sweep Algorithm**: Fair LRU replacement
3. **Partitioned Locking**: Reduce contention on buffer access
4. **Ring Buffers**: Prevent large operations from evicting cache

---

## SUMMARY

PostgreSQL's execution model consists of:

1. **Server Startup**: Initialization, configuration loading, shared memory setup
2. **Connection Flow**: Socket acceptance, authentication, session initialization
3. **Query Processing**: Parse → Analyze → Plan → Execute pipeline
4. **Transaction Management**: MVCC snapshots, isolation levels, WAL logging
5. **Storage Access**: Buffer pool, page I/O, lock management, tuple visibility

Each component is carefully designed for performance and correctness under concurrent loads.

