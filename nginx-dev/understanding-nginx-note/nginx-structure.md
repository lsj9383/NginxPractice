# Nginx 基础架构

对应书中的 `第 8 章 Nginx 基础架构`，仅列出本章中个人感觉重要且有意义的点。

<!-- TOC -->

- [Nginx 基础架构](#nginx-基础架构)
    - [Nginx 架构设计](#nginx-架构设计)
        - [模块化设计](#模块化设计)
        - [请求的多阶段异步处理](#请求的多阶段异步处理)
    - [Nginx 的核心结构体](#nginx-的核心结构体)
        - [ngx_listen_t](#ngx_listen_t)
        - [ngx_cycle_t](#ngx_cycle_t)
    - [Nginx 启动处理流程](#nginx-启动处理流程)
    - [Nginx Worker 工作流程](#nginx-worker-工作流程)
    - [Nginx Master 工作流程](#nginx-master-工作流程)
    - [ngx_pool_t](#ngx_pool_t)
    - [疑问](#疑问)

<!-- /TOC -->

## Nginx 架构设计

### 模块化设计

### 请求的多阶段异步处理

## Nginx 的核心结构体

### ngx_listen_t

Nginx 首先需要监听端口并处理网络事件。

```c
typedef struct ngx_listening_s ngx_listening_t;

struct ngx_listening_s {
    ngx_socket_t        fd;                 // sokcet 套接字具柄
    struct socketaddr   *sockaddr;          // 监听 sockaddr 地址
    socklen_t           socklen;            // sockaddr 地址唱的
    size_t              addr_text_max_len;  // 存储 addr_text 的最大长度
    ngx_str_t           addr_text;          // 以字符串形式存储 IP 地址
    int                 type;               // 套接字类型 例如 type 是 SOCK_STREAM 表示 TCP
    int                 backlog;            // TCP 监听的 backlog 队列，在三次握手完成后，还未被 accept 的最多连接个数
    int                 rcvbuf;             // 内核中对于这个套接字的接收缓冲区大小
    int                 sndbuf;             // 内核中对于这个套接字的发送缓冲区大小

    // =============== TCP 连接创建成功后的回调处理 =============
    // 函数原型为 typedef void (*ngx_connection_handler_pt)(ngx_connection_t *c);
    // 许多事件消费模块（例如 HTTP 模块）都会定义 Listen 的 handler
    ngx_connection_handler_pt   handler;
    // ======================================================

    void                    *servers;
    ngx_log_t               log;
    ngx_log_t               *logp;
    ngx_msec_t              post_acceept_time_out;
    ngx_listening_t         *previous;

    // =========== 监听句柄对应 connection ==============
    // 在 epoll event module 初始化的时候，会将 connection->read->handler 配置为 ngx_event_accept
    // 这样在收到 TCP 三次握手请求的时候，就可以触发对应的事件回调，并获取对应的连接，并且会进一步回调 ngx_listening_t.handler
    ngx_connection_t        *connection;
    // ================================================

    unsigned                open:1;
    unsigned                remain:1;
    unsigned                ignore:1;
    unsigned                bound:1;
    unsigned                inherited:1;
    unsigned                nonblocking_Accept:1;
    unsigned                listen:1;
    unsigned                nonblocking:1;
    unsigned                shared:1;
    unsigned                addr_ntop:1;
}
```

### ngx_cycle_t

无论是 master 进程还是 worker 进程，每个进程都拥有一个 `ngx_cycle_t`。

在 Nginx 进程启动后，`ngx_init_cycle()` 用于进行 `ngx_cycle_t` 的构造。

```c
typedef struct ngx_cycle_s ngx_cycle_t;

struct ngx_cycle_s {
    void                ****conf_ctx;
    ngx_pool_t          *pool;
    ngx_log_t           *log;
    ngx_log_t           new_log;
    ngx_uint_t          files_n;
    ngx_connection_t    **files;
    ngx_connection_t    *free_connections;
    ngx_uint_t          free_connection_n;
    ngx_queue_t         reusable_connections_queue;
    ngx_array_t         listening;
    ngx_array_t         pathes;
    ngx_list_t          open_files;
    ngx_list_t          shared_memory;
    ngx_connection_t    *connectinos;
    ngx_event_t         *read_events;
    ngx_event_t         *write_events;
    ngx_cycle_t         *old_cycle;
    ngx_str_t           conf_file;
    ngx_str_t           conf_param;
    ngx_str_t           conf_prefix;
    ngx_str_t           prefix;
    ngx_str_t           lock_file;
    ngx_str_t           hostname;
};
```

每个模块都可以通过特定的回调获得对 ngx_cycle_t 的掌控权：

```c
struct ngx_module_s {
    ...

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    void                (*exit_thread)(ngx_cycle_t *cycle);
    void                (*exit_process)(ngx_cycle_t *cycle);
    void                (*exit_master)(ngx_cycle_t *cycle);

    ...
};
```

ngx_cycle_t 相关的方法：

- `ngx_cycle_t ngx_init_cycle(ngx_cycle_t *old_cycle)`
- `ngx_int_t ngx_process_options(ngx_cycle_t *cycle)`
  - Nginx 启动时，命令行通常会携带目录参数，该函数解析命令行参数，得到配置文件路径，设置到 `ngx_cycle_t` 中。
- `ngx_init_t ngx_add_inherited_sockets(ngx_cycle_t *cycle)`
  - 在平滑升级过程中，旧 master 进程通过 NGINX 环境变量传递监听端口信息。
  - 新的 master 进程通过该函数从环境变量中读取信息，来使用已经打开的监听端口。
- `ngx_init_t ngx_open_listening_sockts(ngx_cycle_t *cycle)`
  - 统一打开 cycle 中的 listening 数组中的端口，即开始监听。
- `ngx_init_t ngx_configure_listening_sockets(ngx_cycle_t *cycle)`
- `ngx_init_t ngx_close_listening_sockets(ngx_cycle_t *cycle)`
  - 关闭 cycle 中的 listening 数组中打开的端口。
- `ngx_init_t ngx_master_process_cycle(ngx_cycle_t *cycle)`
  - 进入 master 进程的工作循环。
- `ngx_init_t ngx_single_process_cycle(ngx_cycle_t *cycle)`
- `ngx_init_t ngx_start_worker_processes(ngx_cycle_t *cycle, ngx_int_t n, ngx_int_t type)`
- `void ngx_start_cache_manager_processes(ngx_cycle_t *cycle, ngx_uint_t respawn)`
- `void ngx_pass_open_channel(ngx_cycle_t *cycle, ngx_channel_t *ch)`
- `void ngx_signal_worker_processes(ngx_cycle_t *cycle, int signo)`
- `void ngx_reap_children(ngx_cycle_t *cycle)`
- `void ngx_master_process_exit(ngx_cycle_t *cycle)`
- `void ngx_worker_process_cycle(ngx_cycle_t *cycle)`
- `void ngx_worker_process_init(ngx_cycle_t *cycle)`
- `void ngx_worker_process_exit(ngx_cycle_t *cycle)`
- `void ngx_cache_manager_process_cycle(ngx_cycle_t *cycle, void *data)`
- `void ngx_process_events_and_timers(ngx_cycle_t *cycle)`

## Nginx 启动处理流程

1. Nginx 启动时，首先解析命令行参数。Nginx 会预先创建一个临时的 `ngx_cycle_t` 变量，用以存储配置文件路径和一个输出到屏幕的 log。对于配置文件路径的生成依赖于 `ngx_process_options(ngx_cycle_t)` 函数。
1. 随后判断是否处于平滑升级，若是，则通过 `ngx_add_inherited_sockets(ngx_cycle_t)` 从环境变量中继承原 master 进程的相关信息，以使用原 master 进程的监听端口，以实现平滑升级。
1. 此时调用 ngx_init_cycle 方法，会去读取、解析 `nginx.conf` 配置文件。
1. 调用所有`核心模块`的 `create_conf` 方法。而对于非核心模块的配置解析，则是交给了核心模块来完成，因为任何一个非核心模块，都是从属于某一个核心模块的。非核心模块的配置解析是由核心模块去约定的。需要注意，一般并不是核心模块的 `create_conf` 触发了去创建子模块的配置，子模块的配置创建往往是在解析对应的语句块指令时，其回调触发的。例如事件模块的子模块都是在 `event {}` 这个配置项的解析回调中对子模块的配置进行创建和解析。
1. 遍历 `nginx.conf` 的所有配置项，对于任何配置项，会检查所有`核心模块`的配置项，以找出对它感兴趣的模块。
1. 调用所有核心的的 `init_conf` 方法，通知解析完毕。
1. 创建目录、打开文件、初始化共享内存。
1. 此时所有模块都已经拿到了自己的监听端口，并添加到 `ngx_cycle_t` 的 listening 数组中了，Nginx 框架这一步将会把这些监听端口全部打开。即调用 `ngx_open_listening_sockts(ngx_cycle_t)`。
1. 调用所有模块的 `init_module` 方法。
1. 根据 Nginx 的工作模式，会有不同的执行方式：
    1. 单进程模式
        1. 调用所有模块的 `init_process` 方法。
        1. 调用 `ngx_single_process_cycle(ngx_cycle_t)`。
    1. master-worker
        1. 调用所有模块的 `init_process` 方法。
        1. 调用 `ngx_worker_process_cycle(ngx_cycle_t)`。

## Nginx Worker 工作流程

master 通过信号来控制 worker 进程，因此每个 worker 都会注册信号处理方法，即 `ngx_signal_handler`，该信号处理函数主要谁根据接收到的信号来设置全局标识：

信号 | 全局标识 | 意义
-|-|-
QUIT | ngx_quit | 优雅关闭
TERM/INT | ngx_terminate | 强制关闭
USR1 | ngx_reopen | 重新打开所有文件

除此外，还有全局标识 `ngx_exiting` 用来表示准备关闭 worker 进程，不受信号控制，而是 worker 内部控制的变量。

Worker 进程都处于 `ngx_worker_process_cycle(ngx_cycle_t)` 循环中循环运行。

## Nginx Master 工作流程

## ngx_pool_t

Nginx 内存池由 `ngx_pool_t` 表示：

```c
typedef struct ngx_pool_s ngx_pool_t;

struct ngx_pool_s {
    // 小块内存的空间，当空间不足以分配的小块内存，会再生成一个 ngx_pool_s，并通过 ngx_pool_date_t->next 进行连接
    ngx_pool_data_t     d;

    // 评估大块小块的标准
    size_t              max;

    // 大块内存直接从堆中分配，并通过链表进行连接
    // 需要注意 ngx_pfree 释放大块内存时，会遍历链表，因此要尽量避免频繁使用短周期的大块内存
    ngx_pool_large_t    *large;

    ngx_pool_cleanup_t  *cleanup;
    ngx_log_t           *log;
};

struct ngx_pool_large_s {
    // 链表连接
    ngx_pool_large_t    *next;

    // 指向大块内存
    void                *alloc;
};

typedef struct {

    // 未分配内存的首地址
    u_char      *last;

    // 当前小块的尾地址
    u_char      *end;

    // 下一个内存次
    ngx_pool_t  *next;

    // 每当剩余空间不足以分配出小内存块时，failed 递增，当超过 4，则 ngx_pool_t 的 current 将移动到下一个内存池。
    ngx_uint_t  failed;
} ngx_pool_data_t;
```

很巧妙，如果小块内存申请在当前的内存池多次失败，就意味着当前内存剩余的空间太小，所以把 current 移动到下一个内存池。

和内存池有关的方法有：

- 内存池
  - ngx_create_pool
  - ngx_destroy_pool
  - ngx_reset_pool
- 分配和释放内存
  - ngx_palloc
  - ngx_pnalloc
  - ngx_pcalloc
  - ngx_pemalign
  - ngx_pfree
- 随着内存池释放同步释放资源
  - ngx_pool_cleanup_add
  - ngx_pool_run_cleanup_file
  - ngx_pool_cleanup_file
  - ngx_pool_delete_file

使用内存池后，模块开发者只需要关注内存分配。

内存池对大内存和小内存的分配管理方式是不同的：

- 小内存直接从内存池的预分配空间中获取。
- 大内存直接从 Heap 空间中分配，并记录在 ngx_pool_t 中。

对于大内存和小内存的区分，是在 `ngx_create_pool` 的 size 参数决定的，size 参数小于 `NGX_MAX_ALLOC_FROM_POOL - sizeof(ngx_pool_t)` 时，分界线就是 `size - sizeof(ngx_pool_t)`，否则分界线就是 `NGX_MAX_ALLOC_FROM_POOL`。

```c
#define NGX_MAX_ALLOC_FROM_POOL (ngx_pagesize - 1)
```

在 X86 上，ngx_pagesize 就是 4096，即 NGX_MAX_ALLOC_FROM_POOL 是 4095。

**注意：**

- 小块内存如果释放，意味着内存浪费，因为那部分不可能再被使用。
- 大块内存如果释放，可能会导致性能降低（遍历链表导致）。

`ngx_pool_cleanup_add` 会返回一个清理处理器结构，用于指出用户自定义回收时的处理：

```c
struct ngx_pool_cleanup_s {
    ngx_pool_cleanup_pt     handler;
    void                    *data;      // 回调 handler 时，会带给 handler，用以提供回调上下文。
    ngx_pool_clanup_t       *next;
};
```

![](resource/ngx_pool_t.png)

## 疑问

- 非核心模块的配置文件解析和设置由谁负责
- init_module 和 init_process 谁全部模块还是仅核心模块
- pool 工作原理，如果释放内存是否会导致内存碎片积累
