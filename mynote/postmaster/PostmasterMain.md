# PostmasterMain 总览
Postmaster 函数中主要了这几件事
PostmasterMain 函数全览
定义在 postmaster.c:495-1417，是整个 PostgreSQL 服务器的父进程入口，负责启动所有子系统并进入主事件循环。整个过程可以分为以下阶段：

## 第一阶段：进程初始化（行 503-540）
调用 InitProcessGlobals() 设置启动时间戳和随机种子
设置 PostmasterPid = MyProcPid，记录自己的进程 PID
设置 IsPostmasterEnvironment = true，标记当前环境
umask(PG_MODE_MASK_OWNER) 将默认权限设为最严格（仅 owner）
创建 PostmasterContext 内存上下文，postmaster 的 palloc 都从这里分配


### InitProcessGlobals 函数解析
这个函数定义在 postmaster.c:1948-1987，声明在 postmaster.h:95。

它在 postmaster 和每个 backend 进程的早期阶段都会被调用，负责三件事：

1. 记录进程启动时间

MyStartTimestamp = GetCurrentTimestamp();  // 微秒级时间戳
MyStartTime = timestamptz_to_time_t(MyStartTimestamp);  // 转为 time_t（秒级）
MyStartTimestamp 是 TimestampTz 类型（微秒精度），通过 gettimeofday() 获取当前时间。
MyStartTime 是由前者转换得到的 time_t（秒级），用于一些不需要高精度的场景。
2. 初始化全局随机数种子

if (unlikely(!pg_prng_strong_seed(&pg_global_prng_state)))
{
    rseed = ((uint64) MyProcPid) ^
            ((uint64) MyStartTimestamp << 12) ^
            ((uint64) MyStartTimestamp >> 20);
    pg_prng_seed(&pg_global_prng_state, rseed);
}
首先尝试用 高质量随机源（如 /dev/urandom）给 pg_global_prng_state 播种。
如果失败（unlikely 说明极少发生），则退而求其次，用 PID 和启动时间戳的异或组合作为种子。左移 12 位 + 右移 20 位的设计是为了让时间戳的低位（变化快）和高位（变化慢）都能充分混入种子。
3. 为 random(3) 播种（兼容性）

srandom(pg_prng_uint32(&pg_global_prng_state));
PostgreSQL 核心代码已不再使用 random(3)，但考虑到 扩展（extensions）可能还在用，这里用 pg 自己的 PRNG 产生一个值来初始化 libc 的 srandom，确保扩展中的随机行为也是可预测且独立的。
调用位置
调用方	位置
Postmaster 主进程	postmaster.c:507
普通 backend 进程	miscinit.c:109（InitPostgres 中）
WAL sender 等辅助进程	miscinit.c:188（InitStandaloneProcess 中）
简单总结：每个进程启动时尽早调用，确保 MyStartTime/MyStartTimestamp 可用，同时让随机数生成器在不同进程间产生不同的序列。




## 第二阶段：信号处理注册（行 544-590）
注册 postmaster 专用的信号处理器：

信号	处理函数	用途
SIGHUP	handle_pm_reload_request_signal	重新加载配置文件（pg_ctl reload）
SIGINT	handle_pm_shutdown_request_signal	智能关闭（fast shutdown）
SIGQUIT	handle_pm_shutdown_request_signal	立即关闭（immediate shutdown）
SIGTERM	handle_pm_shutdown_request_signal	优雅关闭（smart shutdown）
SIGUSR1	handle_pm_pmsignal_signal	子进程状态变更通知
SIGCHLD	handle_pm_child_exit_signal	子进程退出处理
SIGTTIN/SIGTTOU	SIG_IGN	避免子进程写 stderr 时冻结
SIGXFSZ	SIG_IGN	ulimit 文件大小限制 ⇒ 当作磁盘满处理

## 第三阶段：解析命令行参数（行 595-784）
通过 getopt 解析参数，将选项转化为 GUC 配置：

参数	对应 GUC
-B	shared_buffers
-c / --	通用的 GUC 设置
-D	数据目录路径
-d	调试级别
-h / -i	listen_addresses
-k	unix_socket_directories
-l	启用 SSL
-N	max_connections
-p	port
-S	work_mem
-W	post_auth_delay

## 第四阶段：配置文件与数据目录校验（行 787-876）
SelectConfigFiles() — 定位并加载 postgresql.conf、pg_hba.conf、pg_ident.conf
checkDataDir() — 验证数据目录是否存在、合法
checkControlFile() — 检查 pg_control 文件
ChangeToDataDir() — 切换工作目录到数据目录
验证 GUC 参数组合的合法性（如 superuser_reserved_connections + reserved_connections < max_connections，WAL level 与归档/streaming 的兼容性等）

## 第五阶段：共享内存与 IPC 初始化（行 919-1050）
CreateDataDirLockFile() — 创建 postmaster.pid 锁文件，防止多个 postmaster 同时运行
LocalProcessControlFile() — 读取 pg_control 控制文件
process_shared_preload_libraries() — 加载共享库（如 pg_stat_statements、auth_delay 等扩展）
ApplyLauncherRegister() — 注册逻辑复制的 apply launcher 后台 worker
InitializeMaxBackends() + InitPostmasterChildSlots() — 计算最大后端数，初始化子进程管理槽位
InitializeFastPathLocks() — 初始化快速路径锁数组
process_shmem_requests() — 让预加载库有机会申请额外共享内存
CreateSharedMemoryAndSemaphores() — 创建并 attach 共享内存和信号量（这是关键一步）
set_max_safe_fds() — 评估可打开的最大文件描述符数
InitPostmasterDeathWatchHandle() — 创建管道，让子进程能在 postmaster 死亡时被唤醒


### ApplyLauncherRegister
这个函数把创建的进程注册到 postmaster 的进程列表中，方便后续管理。

### CreateSharedMemoryAndSemaphores
这个绝对是最关键的函数了，这个函数会创建并 attach 共享内存和信号量，这是关键步骤。



## 第六阶段：辅助进程启动（行 1069-1094）
RemovePromoteSignalFiles() — 清理上一次的 standby promote 信号文件
StartSysLogger() — 如果 logging_collector 开启，启动 syslogger 子进程
whereToSendOutput = DestNone — 停止向 stderr 输出日志，转由 syslogger 处理

## 第七阶段：监听套接字（行 1115-1301）
遍历 listen_addresses，为每个地址调用 ListenServerPort() 创建 TCP 监听套接字
遍历 unix_socket_directories，创建 Unix domain socket
将监听信息写入 postmaster.pid 锁文件
CreateOptsFile() — 保存启动参数到文件，供 pg_ctl 使用
写入外部 PID 文件（如果配置了 external_pid_file）
RemovePgTempFiles() — 清理上次残留的临时文件

## 第八阶段：认证与收尾准备（行 1332-1391）
autovac_init() — 初始化 autovacuum 子系统
load_hba() — 加载 pg_hba.conf 客户端认证规则
load_ident() — 加载 pg_ident.conf 用户名映射
记录 PgStartTime（postmaster 正式启动时间，区别于进程自身的 MyStartTimestamp）
更新 postmaster.pid 状态为 PM_STATUS_STARTING
UpdatePMState(PM_STARTUP) — 将 postmaster 状态切换到 STARTUP

## 第九阶段：进入主循环（行 1393-1417）
启动 checkpointer 和 bgwriter 子进程（辅助恢复）
启动 startup 子进程（负责 WAL 恢复）
maybe_start_bgworkers() — 启动注册的后台 worker
ServerLoop() — 进入主事件循环，永不返回。循环内部通过 select() 监听：
客户端连接请求 → fork() 新 backend
信号事件（SIGHUP 重载配置、SIGTERM 关机等）
子进程退出
理论上 ServerLoop() 不会返回，如果返回则调用 ExitPostmaster() 退出

## 整体流程图

PostmasterMain()
 ├── InitProcessGlobals()         # 时间戳 + 随机种子
 ├── 信号处理注册                   # SIGHUP/SIGINT/SIGTERM/SIGCHLD 等
 ├── getopt 解析命令行参数          # -B/-D/-N/-p/-c 等 → GUC
 ├── SelectConfigFiles()          # 加载 postgresql.conf
 ├── checkDataDir()               # 校验数据目录
 ├── CreateDataDirLockFile()      # 创建 postmaster.pid 锁文件
 ├── process_shared_preload_libraries()
 ├── CreateSharedMemoryAndSemaphores()  # ★ 共享内存
 ├── StartSysLogger()             # 日志收集器
 ├── ListenServerPort() × N       # TCP + Unix socket 监听
 ├── load_hba() / load_ident()    # 认证配置
 ├── StartChildProcess() × 3      # checkpointer + bgwriter + startup
 └── ServerLoop()                 # ★ 主事件循环（永不返回）


