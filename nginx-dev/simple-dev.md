# Nginx模块开发

<!-- TOC -->

- [Nginx模块开发](#nginx模块开发)
    - [一、概述](#一概述)
    - [二、Nginx重要结构体梳理](#二nginx重要结构体梳理)
        - [2.1 基础结构](#21-基础结构)
        - [2.2 底层结构](#22-底层结构)
        - [2.3 第三方模块结构](#23-第三方模块结构)
        - [2.4 请求于与响应的结构](#24-请求于与响应的结构)
        - [2.5 重要的配置项关系](#25-重要的配置项关系)
    - [三、常用的配置与上下文获取的宏](#三常用的配置与上下文获取的宏)
    - [四、配置解析](#四配置解析)
        - [4.1 基本原理](#41-基本原理)
        - [4.2 配置项的解析方式](#42-配置项的解析方式)
        - [4.3 预设的配置项解析函数](#43-预设的配置项解析函数)
        - [4.4 选择配置项](#44-选择配置项)
        - [4.5 预设的配置项合并函数](#45-预设的配置项合并函数)
        - [4.6 复杂配置项](#46-复杂配置项)
        - [4.7 一个配置项 Demo](#47-一个配置项-demo)
    - [五、添加内部变量](#五添加内部变量)
    - [六、模块处理函数挂载](#六模块处理函数挂载)
        - [6.1 *`NGX_HTTP_CONTENT_PHASE`阶段挂载方案*](#61-ngx_http_content_phase阶段挂载方案)
        - [6.2 *通用挂载方案*](#62-通用挂载方案)
        - [4.3 *处理器返回值*](#43-处理器返回值)
    - [七、日志](#七日志)
        - [7.1 日志的格式化](#71-日志的格式化)
        - [7.2 日志的获取方式](#72-日志的获取方式)
        - [7.3 对于复杂变量的打印](#73-对于复杂变量的打印)
    - [八、模块开发步骤](#八模块开发步骤)
        - [8.1 自定义配置结构体](#81-自定义配置结构体)
        - [8.2 配置项命令](#82-配置项命令)
        - [8.3 模块上下文](#83-模块上下文)
    - [九、模块构建](#九模块构建)
        - [9.1 模块准备](#91-模块准备)
        - [9.2 模块编译](#92-模块编译)
    - [附录](#附录)

<!-- /TOC -->

## 一、概述
Nginx模块按功能区分有三种重要的角色:
* 处理请求并产生输出的 Handler 模块；
* 处理由 Handler 产生的输出的 Filter（滤波器）模块；
* 当出现多个后台服务器时，Load-balancer （负载均衡器）模块负责选择其中一个后台服务器发送请求

![](https://box.kancloud.cn/2016-09-01_57c7edd05325b.jpg)

在开始前，需要明确一点，配置的解析仅仅存在于nginx启动(或热加载)时。当配置加载完毕后，每个模块对应的每个语句块的配置相关数据都已经保存在内存中了，当请求到的时候，根据其所处的阶段，可以获取相应的配置信息，以对其进行处理。

## 二、Nginx重要结构体梳理

### 2.1 基础结构
#### 2.2.0 简单结构
简单结构和系统bit还有关系，这里假设系统为64bit系统
```c
/* 在文件 src/core/ngx_config.h 定义了基本的数据映射 */
typedef intptr_t        ngx_int_t;
typedef uintptr_t       ngx_uint_t;
typedef intptr_t        ngx_flag_t;

/* 在文件 /usr/include/stdint.h 文件中*/
typedef long int            intptr_t;
typedef unsigned long int   uintptr_t;
```

#### 2.1.1 *ngx_str_t*
nginx字符串通常使用ngx_str_t, 该结果包含了字符数组以及字符串长度。
```c
typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
```
ngx_str_t.data并未限制终止字符是什么, 但是在某些配置文件解析的代码中是假定了NULL终止符, 目的是为了简化字符串比较, 并方便传递给系统调用。

Nginx提供了一些非常方便的用户字符串初始化和赋值的宏：
```c
#define ngx_string(str) {sizeof(str)-1, (u_char *) str}

#define ngx_null_string {0, NULL}

#define ngx_str_set(str, text) \
    (str)->len = sizeof(text)-1; (str)->data = (u_char *)text

#define ngx_str_null(str)   (str)->len = 0; (str)->data = NULL

// 一种初始化方式
ngx_str_t str1 = ngx_string("hello nginx");
ngx_str_t str2 = ngx_null_string;

// 一种赋值方式
ngx_str_t str3, str4;
ngx_str_set(&str3, "hello nginx");
ngx_str_null(&str4);
```

#### 2.1.2 *ngx_array_t*
ngx的动态数组数组。
```c
typedef struct {
    void        *elts;      // array中的元素
    ngx_uint_t   nelts;     // array中的元素个数
    size_t       size;      // 单个元素的大小
    ngx_uint_t   nalloc;    // array的容量, 当超过最大容量, 则重新分配一个新的数组, 并把数据从老的数组移动到新的数组, nalloc翻番
    ngx_pool_t  *pool;
} ngx_array_t;
```

![](https://box.kancloud.cn/2016-09-01_57c7edcf4bebd.jpg)

* array = ngx_array_create(pool, n, size), 从pool中分配内存创建n个元素，每个元素大小为size
* ngx_array_init(array, pool, n, size), 对array进行初始化, 相比于create，就是少了个ngx_array_t内存的申请
* ngx_array_push(array), 在array的尾部创建一个元素, 并返回该元素的指针
* ngx_array_push_n(array, n), 在array的第n个位置插入一个元素， 并返回该元素的指针

#### 2.1.3 *ngx_list_t*
ngx的链表。
```c
typedef struct ngx_list_part_s  ngx_list_part_t;

// 链表节点, 每个链表节点都是一个数组
struct ngx_list_part_s {
    void             *elts;             // 当前节点的数组首地址
    ngx_uint_t        nelts;            // 当前节点的数组元素个数
    ngx_list_part_t  *next;             // 下一个节点
};

typedef struct {
    ngx_list_part_t  *last;             // 链表的最后一个元素的指针
    ngx_list_part_t   part;             // 链表的首个元素
    size_t            size;             // 链表节点中每个元素的大小
    ngx_uint_t        nalloc;           // 每个链表节点中的数组最大元素个数
    ngx_pool_t       *pool;
} ngx_list_t;
```
这个结构可以无限制的往链表中添加元素, 当每个节点的元素个数满了以后，则会添加一个新的链表节点。`ngx_list_t`主要用于记录http的headers, 该结构不能进行元素删除, 但是可以标记元素缺失。

![](https://box.kancloud.cn/2016-09-01_57c7edcf652d0.jpg)

* ngx_list_create(pool, n, size), 从pool中创建ngx_list_t, size是每个元素的大小, n是每个链表节点的数组可容纳的元素个数(即nalloc)。
* ngx_list_init(list, pool, n, size), 和create类似, 但需要已经创建好的list
* ngx_list_push(list), 往list中添加一个新元素, 并返回该元素.

#### 2.1.4 *ngx_queue_s*
```c
struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};
```

#### 2.1.5 *rbtree*
```c
typedef struct {
    ngx_rbtree_t       rbtree;
    ngx_rbtree_node_t  sentinel;

    /* custom per-tree data here */
} my_tree_t;

typedef struct {
    ngx_rbtree_node_t  rbnode;

    /* custom per-node data */
    foo_t              val;
} my_node_t;
```

#### 2.1.6 *ngx_table_elt_t*
一个keyvalue条目
```c
typedef struct {
    ngx_uint_t        hash;             // key的hash值, 方便在ngx_hash_t中使用
    ngx_str_t         key;
    ngx_str_t         value;
    u_char           *lowcase_key;
} ngx_table_elt_t;
```

#### 2.1.7 *ngx_buf_t*
```c
/* 缓冲区结构 */
typedef void *            ngx_buf_tag_t;

typedef struct ngx_buf_s  ngx_buf_t;

struct ngx_buf_s {
    u_char          *pos;   /* 缓冲区数据在内存的起始位置 */
    u_char          *last;  /* 缓冲区数据在内存的结束位置 */
    /* 这两个参数是处理文件时使用，类似于缓冲区的pos, last */
    off_t            file_pos;
    off_t            file_last;

    /* 由于实际数据可能被包含在多个缓冲区中，则缓冲区的start和end指向
     * 这块内存的开始地址和结束地址，
     * 而pos和last是指向本缓冲区实际包含的数据的开始和结尾
     */
    u_char          *start;         /* start of buffer */
    u_char          *end;           /* end of buffer */
    ngx_buf_tag_t    tag;
    ngx_file_t      *file;          /* 指向buffer对应的文件对象 */
    /* 当前缓冲区的一个影子缓冲区，即当一个缓冲区复制另一个缓冲区的数据，
     * 就会发生相互指向对方的shadow指针
     */
    ngx_buf_t       *shadow;

    /* 为1时，表示该buf所包含的内容在用户创建的内存块中
     * 可以被filter处理变更
     */
    /* the buf's content could be changed */
    unsigned         temporary:1;

    /* 为1时，表示该buf所包含的内容在内存中，不能被filter处理变更 */
    /*
     * the buf's content is in a memory cache or in a read only memory
     * and must not be changed
     */
    unsigned         memory:1;

    /* 为1时，表示该buf所包含的内容在内存中，
     * 可通过mmap把文件映射到内存中，不能被filter处理变更 */
    /* the buf's content is mmap()ed and must not be changed */
    unsigned         mmap:1;

    /* 可回收，即这些buf可被释放 */
    unsigned         recycled:1;
    unsigned         in_file:1; /* 表示buf所包含的内容在文件中 */
    unsigned         flush:1;   /* 刷新缓冲区 */
    unsigned         sync:1;    /* 同步方式 */
    unsigned         last_buf:1;/* 当前待处理的是最后一块缓冲区 */
    unsigned         last_in_chain:1;/* 在当前的chain里面，该buf是最后一个，但不一定是last_buf */

    unsigned         last_shadow:1;
    unsigned         temp_file:1;

    /* STUB */ int   num;
};


/* ngx_chain_t 数据类型是与缓冲区类型 ngx_buf_t 相关的链表结构 */
struct ngx_chain_s {
    ngx_buf_t    *buf;  /* 指向当前缓冲区 */
    ngx_chain_t  *next; /* 指向下一个chain，形成chain链表 */
};
```
![](https://box.kancloud.cn/2016-09-01_57c7edcf34cf2.jpg)

### 2.2 底层结构
#### 2.2.1 内存池
内存池Pool分为header(ngx_pool_t和ngx_pool_data_t)和data, header中有指明data的位置，如下图所示:
![](https://box.kancloud.cn/2016-09-01_57c7edcebbcb0.jpg)

* 实际的data由ngx_pool_data_t中的last标出其已经使用的位置
* data的最大存储位置为end
* 内存池是链表结构, 可以无限扩展

具体结构参考源码:
```c
/* 内存池结构 */
/* 内存池数据结构模块 */
typedef struct {
    u_char               *last;     /* 当前内存分配的结束位置，即下一段可分配内存的起始位置 */
    u_char               *end;      /* 内存池的结束位置 */
    ngx_pool_t           *next;     /* 指向下一个内存池 */
    ngx_uint_t           failed;    /* 记录内存池内存分配失败的次数 */
} ngx_pool_data_t;  /* 维护内存池的数据块 */

/* 内存池的管理模块，即内存池头部结构 */
struct ngx_pool_s {
    ngx_pool_data_t       d;        /* 内存池的数据块 */
    size_t                max;      /* 内存池数据块的最大值 */
    ngx_pool_t           *current;  /* 指向当前内存池 */
    ngx_chain_t          *chain;    /* 指向一个 ngx_chain_t 结构 */
    ngx_pool_large_t     *large;    /* 大块内存链表，即分配空间超过 max 的内存 */
    ngx_pool_cleanup_t   *cleanup;  /* 析构函数，释放内存池，这是一个handler链表 程序可以注册自己的清理函数 */
    ngx_log_t            *log;      /* 内存分配相关的日志信息 */
};


/* 大块内存 */
struct ngx_pool_large_s{  
    ngx_pool_large_t  *next;    //指向下一块大块内存  
    void    *alloc;             //指向分配的大块内存  
};


/* 文件 core/ngx_core.h */
typedef struct ngx_pool_s   ngx_pool_t;
typedef struct ngx_chain_s  ngx_chain_t;
typedef struct ngx_pool_large_s ngx_pool_large_t;

/* 相关函数 */
ngx_pool_t *ngx_create_pool(size_t size, ngx_log_t *log);       /* 创建Pool内存池 */
void ngx_reset_pool(ngx_pool_t *pool);                          /* 重置内存池, 将释放所有 large 内存，并且将d->last复位 */
```

每次从Pool内存池中申请空间都分为申请大内存和小内存, 大内存和小内存的分配逻辑如下图所示：
![](https://box.kancloud.cn/2016-09-01_57c7edce99d27.jpg)

#### 2.2.2 cycle
cycle保存了ngx的全局数据, 其中包含了各个模块的配置上下文，如：
* `cycle->modules[i]`
* `cycle->conf_ctx[index]`

```c
/* ngx_cycle_t 全局变量数据结构
* 保存所有模块配置项的结构体指针，该数组每个成员又是一个指针，
* 这个指针又指向存储指针的数组
*/
struct ngx_cycle_s {
    void                  ****conf_ctx;             /* 所有模块配置上下文的数组 */
    ngx_pool_t               *pool;                 /* 内存池 */

    ngx_log_t                *log;                  /* 日志 */
    ngx_log_t                 new_log;

    ngx_uint_t                log_use_stderr;       /* unsigned  log_use_stderr:1; */

    ngx_connection_t        **files;                /* 连接文件 */
    ngx_connection_t         *free_connections;     /* 空闲连接 */
    ngx_uint_t                free_connection_n;    /* 空闲连接的个数 */

    ngx_module_t            **modules;              /* 模块的数组 */
    ngx_uint_t                modules_n;            /* 模块的个数 */
    ngx_uint_t                modules_used;         /* unsigned  modules_used:1; */

    ngx_queue_t               reusable_connections_queue;

    ngx_array_t               listening;
    ngx_array_t               paths;
    ngx_array_t               config_dump;
    ngx_list_t                open_files;
    ngx_list_t                shared_memory;

    ngx_uint_t                connection_n; /* 已连接个数 */
    ngx_uint_t                files_n;      /* 已打开文件的个数 */

    ngx_connection_t         *connections;  /* 连接 */
    ngx_event_t              *read_events;  /* 读事件 */
    ngx_event_t              *write_events; /* 写事件 */

    ngx_cycle_t              *old_cycle;

    ngx_str_t                 conf_file;    /* nginx 配置文件 */
    ngx_str_t                 conf_param;   /* nginx 处理配置文件时需要特殊处理的，在命令行携带的参数 */
    ngx_str_t                 conf_prefix;  /* nginx 配置文件的路径 */
    ngx_str_t                 prefix;       /* nginx 安装路径 */
    ngx_str_t                 lock_file;    /* 加锁文件 */
    ngx_str_t                 hostname;     /* 主机名 */
};
```

#### 2.2.3 *ngx_conf_t*
```c
/* 解析配置时所使用的结构体 */
struct ngx_conf_s {
    char                 *name;     /* 当前解析到的指令 */
    ngx_array_t          *args;     /* 当前指令所包含的所有参数 */

    ngx_cycle_t          *cycle;    /* 待解析的全局变量ngx_cycle_t */
    ngx_pool_t           *pool;     /* 内存池 */
    ngx_pool_t           *temp_pool;/* 临时内存池，分配一些临时数组或变量 */
    ngx_conf_file_t      *conf_file;/* 待解析的配置文件 */
    ngx_log_t            *log;      /* 日志信息 */

    void                 *ctx;      /* 描述指令的上下文 */
    ngx_uint_t            module_type;/* 当前解析的指令的模块类型 */
    ngx_uint_t            cmd_type; /* 当前解析的指令的指令类型 */

    ngx_conf_handler_pt   handler;  /* 模块自定义的handler，即指令自定义的处理函数 */
    char                 *handler_conf;/* 自定义处理函数需要的相关配置 */
};
```

### 2.3 第三方模块结构
* ngx_module_t, 定义了一个模块对象, 该对象有非常重要的两个成员:
    * `ngx_command_t *commands`, 描述了模块所拥有的指令, 以及描述指令的解析和存储方式
    * `void *ctx;`, 配置上下文, 描述了配置文件的初始化、生成方式, 根据type不同有以下几种配置上下文:
        * `ngx_core_module_t *ctx`, 核心模块的配置上下文
        * `ngx_http_module_t *ctx`, http模块的配置上下文


#### 2.3.1 *ngx_module_t*
```c
// 我们通常这样定义一个ngx_module_t
ngx_module_t  ngx_http_<module name>_module = {
    NGX_MODULE_V1,
    &ngx_http_${module}_module_ctx,         /* module context */
    ngx_http_${module}_commands,            /* module directives */
    NGX_HTTP_MODULE,                        /* module type */
    NULL,                                   /* init master */
    NULL,                                   /* init module */
    NULL,                                   /* init process */
    NULL,                                   /* init thread */
    NULL,                                   /* exit thread */
    NULL,                                   /* exit process */
    NULL,                                   /* exit master */
    NGX_MODULE_V1_PADDING
};


// ngx_module_t的完整结构
struct ngx_module_s {
    ngx_uint_t            ctx_index;
    ngx_uint_t            index;

    char                 *name;

    ngx_uint_t            spare0;
    ngx_uint_t            spare1;

    ngx_uint_t            version;
    const char           *signature;

    void                 *ctx;
    ngx_command_t        *commands;
    ngx_uint_t            type;

    ngx_int_t           (*init_master)(ngx_log_t *log);

    ngx_int_t           (*init_module)(ngx_cycle_t *cycle);

    ngx_int_t           (*init_process)(ngx_cycle_t *cycle);
    ngx_int_t           (*init_thread)(ngx_cycle_t *cycle);
    void                (*exit_thread)(ngx_cycle_t *cycle);
    void                (*exit_process)(ngx_cycle_t *cycle);

    void                (*exit_master)(ngx_cycle_t *cycle);

    uintptr_t             spare_hook0;
    uintptr_t             spare_hook1;
    uintptr_t             spare_hook2;
    uintptr_t             spare_hook3;
    uintptr_t             spare_hook4;
    uintptr_t             spare_hook5;
    uintptr_t             spare_hook6;
    uintptr_t             spare_hook7;
};
```

通常而言，只需要定义 ctx 和 commands：
* ctx，对于 http 模块，该类型为 `ngx_http_module_t`，即如何解析模块的配置项，以及添加 handlers 挂载函数。
* commands, 该变量告诉 Nginx HTTP 框架，该模块所拥有的配置项，HTTP 框架回去解析这些配置项。

#### 2.3.2 *ngx_command_t*
用来定义一个模块的配置项，以及这个配置项的解析和存储方式
```c
typedef struct ngx_command_s ngx_command_t;
struct ngx_command_s {
    ngx_str_t   name;
    ngx_uint_t  type;
    char        *(*set)(ngx_conf_t *cf,ngx_command_t *cmd,void *conf)；
    ngx_uint_t  conf;
    ngx_uint_t  offset;
    void        *post;
};
```

下述为配置项各个成员的含义:

| name | desc |
| ------ | ------ |
| name | 配置项在配置文件中使用的名称 |
| type | 配置项类型，该值会指定该配置项可出现的位置, 以及可以指定携带多少个参数 |
| set | 在解析配置时，碰到了该配置项名后, 将会调用该方法来解析其后面的参数, 并保存在自定义结构体中, nginx 预设了一些解析函数 |
| conf | 该参数决定该配置项用 creat_`http/srv/loc`_conf 中的哪个上下文进行存储, 很明显，一个配置只可能属于一种结构体。  |
| offset | 配置项出现时，将其保存在自定义结构体中的内存偏移量, 主要是为了简化`set`。可用预定宏`offsetof(struct，member)`, 当然若`set`自定义，那么offset无意义 |
| post | 通常为null |

更具体的可以参考 [四、配置解析](#四配置解析)

#### 2.3.3 *ngx_http_module_t*
```c
typedef struct {
    // 配置解析前
    ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);

    // 配置解析完毕后
    ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);

    // 当碰到http块时的回调函数，用于创建http块配置项
    void*       (*create_main_conf)(ngx_conf_t *cf);

    // 初始化http块配置项，该函数会在配置项合并前调用
    char*       (*init_main_conf)(ngx_conf_t *cf, void *conf);

    // 当碰到 http/server 块时的回调函数，用于创建 server 块配置项
    void*       (*create_srv_conf)(ngx_conf_t *cf);

    // 用于合并 main 级别和 srv 级别的 server 块配置项
    char*       (*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);

    // 当碰到 http/server/location 块时的回调函数，用于创建 location 块的配置项
    void*       (*create_loc_conf)(ngx_conf_t *cf);

    // 用于合并main, srv 级别和 loc 级别的location块配置项
    char*       (*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);
} ngx_http_module_t;
```
* postconfiguration, 配置解析完毕后调用, 通常在这个函数中注册某个Phase的handler。
* create_main_conf
* create_srv_conf
* create_loc_conf

#### 2.3.4 *ngx_http_conf_ctx_t*
每次碰到 http / server / location 语句块，都会生成一个 ngx_http_conf_ctx_t 对象，用于存储对应级别的 http / server / location 配置。

```c
typedef struct {
    void **main_conf;
    void **srv_conf;
    void **loc_conf;
} ngx_http_conf_ctx_t;
```

一个 ngx_http_conf_ctx_t 会存储所有模块的 http / server / location 配置。

* `ngx_http_conf_ctx_t.main_conf[ctx_index]`, 某个模块的 http 配置。
* `ngx_http_conf_ctx_t.srv_conf[ctx_index]`, 某个模块的 server 配置。
* `ngx_http_conf_ctx_t.loc_conf[ctx_index]`, 某个模块的 location 配置。

### 2.4 请求于与响应的结构

#### 2.4.1 请求
```c
typedef struct {
...
    ngx_pool_t                          *pool;              /* the memory pool, used in the ngx_palloc functions */
    ngx_str_t                           uri;                /* 是request请求的路径,e.g. "/query.cgi". */
    ngx_str_t                           args;               /* 是请求串参数中问号后面的参数(e.g. "name=john") */
    ngx_http_headers_in_t               headers_in;         /* 请求头 */
    ngx_http_headers_out_t              headers_out;        /* 响应头 */
...
} ngx_http_request_t;
```

#### 2.4.2 响应
```c
/* 响应结构体头部 */
typedef stuct {
...
    ngx_uint_t                        status;
    size_t                            content_type_len;
    ngx_str_t                         content_type;
    ngx_table_elt_t                  *content_encoding;     /* 不在ngx_http_headers_out_t结构体中的headers用该字段整合 */
    off_t                             content_length_n;
    time_t                            date_time;
    time_t                            last_modified_time;
..
} ngx_http_headers_out_t;

// 发送响应头部
ngx_http_send_header(r);
```

### 2.5 重要的配置项关系

## 三、常用的配置与上下文获取的宏
* ngx_http_cycle_get_module_main_conf, 通过 cycle 获得指定模块的 main_conf。
```c
# define ngx_http_cycle_get_module_main_conf(cycle, module)                                         \
(                                                                                                   \
    cycle->conf_ctx[ngx_http_module.index] ?                                                        \
    ((ngx_http_conf_ctx *) cycle->conf_ctx[ngx_http_module.index])->main_conf[module.ctx_index] :   \
    NULL                                                                                            \
)
```
* ngx_http_conf_get_module_main_conf, 通过 `nginx_conf_t` 获得指定模块的 main_conf。
* ngx_http_get_module_loc_conf, 通过 `ngx_http_request_t` 获得指定模块的 loc_conf，常用来获得模块自己定义的 loc_conf。


## 四、配置解析
### 4.1 基本原理


后续会列出配置项的各种预设宏定义，在开始前先再看下  `ngx_command_t` 的各项成员：
```c
typedef struct ngx_command_s ngx_command_t;
struct ngx_command_s {
    ngx_str_t   name;
    ngx_uint_t  type;
    char        *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)；
    ngx_uint_t  conf;
    ngx_uint_t  offset;
    void        *post;
};
```

### 4.2 配置项的解析方式
`ngx_command_t` 中的 type 参数，描述了解析配置项的方式，可以通过 `|` 进行多个功能特性的同时选择。

常用的宏：
| type | desc |
| ------ | ------ |
| NGX_HTTP_MAIN_CONF | 该配置项可出现在http块内(不包含server块) |
| NGX_HTTP_SRV_CONF | 该配置项可出现在server块内(不包含location块) |
| NGX_HTTP_LOC_CONF | 该配置项可出现在location块内 |
| NGX_CONF_NOARGS | 该配置项没有参数 |
| NGX_CONF_TAKE1 | 可以携带1个参数 |
| NGX_CONF_TAKE2 | 可以携带2个参数 |
| NGX_CONF_TAKE12 | 可以携带1或者2个参数 |

例如配置项可以出现在 main、server、location，并且可以支持两个参数，则需要如下配置：
```c
NGX_HTTP_MAIN_CONF | NGX_HTTP_SRV_CONF | NGX_HTTP_LOC_CONF | NGX_CONF_TAKE2
```

### 4.3 预设的配置项解析函数

`ngx_command_t` 中的 set 参数，用于配置预设的配置项赋值方式，这非常方便，避免自己写解析配置项值的函数。

常用的宏：
| set | desc |
| ------ | ------ |
| ngx_conf_set_flag_slot | 配置项只能携带一个参数，且必须是 on/off |
| ngx_conf_set_str_slot | 该配置项只能携带一个参数 |
| ngx_conf_set_num_slot | 配置项只能携带一个参数，且必须为数字 |
| ngx_http_set_complex_value_slot | 配置项只能携带一个参数，可以为变量 |

和 set 参数息息相关的是 offset 参数，因为要告知 Nginx 解析出来的配置项值需要赋值给配置对象中的哪个变量，使用 offset 函数非常方便：
```c
// 指定 struct 的 field 字段
offset(${struct}, ${field})
```

### 4.4 选择配置项

`ngx_command_t` 中的 conf 参数，用于选择具体的配置项交给 set 函数进行赋值配置。

| conf                      | desc                                                             |
| ------------------------- | ---------------------------------------------------------------- |
| NGX_HTTP_MAIN_CONF_OFFSET | http 使用的配置项，即 `ngx_http_conf_ctx->main_conf[ctx_index]`     |
| NGX_HTTP_SRV_CONF_OFFSET  | server 使用的配置项，即 `ngx_http_conf_ctx->srv_conf[ctx_index]`    |
| NGX_HTTP_LOC_CONF_OFFSET  | location 使用的配置项，即 `ngx_http_conf_ctx->loc_conf[ctx_index]`  |

通常，我们使用的是 location 配置项，因此这里通常使用 `NGX_HTTP_LOC_OFFSET`。

注意，这里提到的 `location 使用的配置项`，并不是指只出现在 location 中的配置项，而是`匹配到一个请求进行处理`，这类均是属于 location 范畴。

配置项没有出现在 location 块中，但是出现在 http / server 块中，仍然属于 location 会使用到的配置项。

到解析 http server 块的时候，也会生成 `ngx_http_conf_ctx` 对象，也会调用 `create_loc_conf` 创建在 location 中会使用的配置。

### 4.5 预设的配置项合并函数

在合并函数中，可以通过一些预设的命令来对常见类型的配置项进行合并。

* `ngx_conf_merge_value(prev, conf, default)`，合并可以使用等号直接赋值的变量，这些变量要求初始化为 NGX_CONF_UNSET。
* `ngx_conf_merge_str_value(prev, conf, default)`，合并 ngx_str_t 类型的变量，default 为 `char*` 字符串。
* `ngx_conf_merge_ptr_value(prev, conf, default)`， 合并指针类型变量，这些变量要求初始化为 NGX_CONF_UNSET_PTR。

更多的配置项合并可以参考 [src/core/ngx_conf_file.h](https://github.com/nginx/nginx/blob/master/src/core/ngx_conf_file.h)

需要注意，对于复杂配置项没有合并函数，可以通过以下方法进行合并：
```c
if (child->mytest_complex == NULL) {
    child->mytest_complex = parent->mytest_complex;
}
```

### 4.6 复杂配置项

复杂配置项是用于快速支持配置项中存在变量的情况，如果使用复杂配置项，可以正确解析参数中的 `$var` 变量的值。如果使用 ngx_str_t 作为解析方式，则无法取到参数对应的值，而是取到字符串 `"$var"`。

复杂配置项的类型为：`ngx_http_complex_value_t`，通常是一个指针。

复杂配置项的解析函数：`ngx_http_set_complex_value_slot`。

复杂配置项的使用需要使用固定的函数转换为 ngx_str_t 进行使用: `ngx_http_complex_value`

配置项对应值的提取：
```c
ngx_str_t val;

if (slcf->my_complex == NULL) {
    return NGX_DECLINED;
}

if (ngx_http_complex_value(r, slcf->my_complex, &val) != NGX_OK) {
    return NGX_ERROR;
}
```

### 4.7 一个配置项 Demo

```c
typedef struct{
    ngx_str_t                   mytest_str;
    ngx_uint_t                  mytest_uint;
    ngx_http_complex_value_t    *mytest_complex;        // 需要注意，复杂变量这里配置的是指针
}ngx_http_mytest_conf_t;


static ngx_command_t ngx_http_mytest_commands[] = {
    {
        ngx_string("mytest_str"),
        NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
        ngx_conf_set_str_slot,
        NGX_HTTP_LOC_CONF_OFFSET,
        offsetof(ngx_http_mytest_conf_t, mytest_str),
        NULL
    },
    {
        ngx_string("mytest_uint"),
        NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
        ngx_conf_set_num_slot,
        NGX_HTTP_LOC_CONF_OFFSET,
        offsetof(ngx_http_mytest_conf_t, mytest_uint),
        NULL
    },
    {
        ngx_string("mytest_complex"),
        NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
        ngx_http_set_complex_value_slot,
        NGX_HTTP_LOC_CONF_OFFSET,
        offsetof(ngx_http_mytest_conf_t, mytest_complex),
        NULL
    },

    ngx_null_command
};

// 创建配置项
void* ngx_http_mytest_create_loc_conf(ngx_conf_t *cf){
    ngx_http_mytest_conf_t *conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_mytest_conf_t));

    if(!conf) {
        return NULL;
    }

    return conf;
}

// 合并配置项
char* ngx_http_mytest_merge_loc_conf(ngx_conf_t *cf, void *prev, void *conf) {
    ngx_http_mytest_conf_t *parent = prev;
    ngx_http_mytest_conf_t *child  = conf;

    // 对于复杂值，通过判断 child complex 是否为 NULL 来判断是否使用 parent complex 来进行合并
    if (child->mytest_complex == NULL) {
        child->mytest_complex = parent->mytest_complex;
    }

    return NGX_CONF_OK;
}

// 模块上下文
static ngx_http_module_t ngx_http_mytest_module_ctx = {
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    ngx_http_mytest_create_loc_conf,
    ngx_http_mytest_merge_loc_conf
};
```

## 五、添加内部变量
Nginx 存在两种变量：
* 内部变量，由 Nginx C 模块配置的变量。
* 外部变量，即通过 set 命令设置的变量。

## 六、模块处理函数挂载
模块定义好后，上下文的内容也创建好了，解析方式也有了，接着就是http请求来了以后，请求到了某个阶段，将会触发模块对请求进行处理。在指定阶段对请求进行处理，需要将模块的处理函数挂载到对应的阶段上。
### 6.1 *`NGX_HTTP_CONTENT_PHASE`阶段挂载方案*
该方案在配置命令的set回调函数中, 进行挂载, 只适合于`NGX_HTTP_CONTENT_PHASE`。
```c
static char* ngx_http_example_set_func(ngx_conf_t *cf,ngx_command_t *cmd,void *conf){
    ngx_http_core_loc_conf_t clcf = ngx_http_conf_get_module_loc_conf(cf,ngx_http_core_module);
    clcf->handler = ngx_http_example_handler;
    return NGX_CONF_OK;
}
```

### 6.2 *通用挂载方案*
所有的可挂载点，都可以在上下文配置解析完毕后进行挂载，即通过`ngx_int_t   (*postconfiguration)(ngx_conf_t *cf)`回调:
```c
//该函数用于挂载到日志阶段，执行mylog模块的逻辑
static ngx_int_t ngx_http_mytest_handler(ngx_http_request_t *r);

static ngx_int_t postconfiguration(ngx_conf_t *cf) {
    ngx_http_handler_pt *h;
    ngx_http_core_main_conf_t *cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);
    /*
     * ngx_http_core_main_conf_t的phases成员保存了各个阶段的回调函数
     */

    // 获得指定阶段的处理函数指针，配置该指针指向处理函数的地址
    h = ngx_array_push(&cmcf->phases[NGX_HTTP_LOG_PHASE].handlers);
    if(!h){
        return NGX_ERROR;
    }
    // 挂载处理函数
    *h = ngx_http_mytest_handler;
    return NGX_OK;
}
```

### 4.3 *处理器返回值*
挂载在各个Phase上的Handler有着多种可能的返回值，每种返回值的意义以及ngx的处理方式并不相同：
* NGX_OK, 处理成功, ngx会引导请求进入下一个Phase
* NGX_DECLINED, 处理成功, ngx会引动请求进入该Phase的下一个Handler, 若这是最后一个Handler则进入下一个Phase
* NGX_AGAIN, NGX_DONE, 在该阶段挂起, 当事件触发时(如io或者延时)ngx重新执行handler
* 其他值则直接认为是响应的状态码, 将会直接终止Handler, 并返回该状态码。

## 七、日志
Nginx C 模块打印日志的方法非常多，主要是因为有的平台不支持可变参数导致，Linux 下通常都可以使用可变参数，因此这里主要列出可变参数的日志打印方法：
* ngx_log_error(level, log, args...)
* ngx_log_debug(level, log, args...)
* ngx_log_error_core(level, log, err, fmt, ...)

```c
#define ngx_log_error(level, log, args...) \
    if ((log)->log_level >= level) ngx_log_error_core(level, log, args)

#define ngx_log_debug(level, log, args...) \
    if ((log)->log_level & level) ngx_log_error_core(NGX_LOG_DEBUG, log, args)
```

ngx_log_error 根据日志的级别不同进行日志打印，其中的 level 参数支持以下的日志级别：
* NGX_LOG_STDERR, 0, 最高级别日志，日志内容不会在写入 log 参数指定的文件，而是会直接将日志输出到标准错误设备。
* NGX_LOG_EMERG, 1
* NGX_LOG_ALERT, 2
* NGX_LOG_CRIT, 3
* NGX_LOG_ERR, 4
* NGX_LOG_WARN, 5
* NGX_LOG_NOTICE, 6
* NGX_LOG_INFO, 7
* NGX_LOG_DEBUG, 8

ngx_log_debug 打印的是 DEBUG 日志，其中的 level 参数用于区分日志用途：
* NGX_LOG_DEBUG_CORE
* NGX_LOG_DEBUG_ALLOC
* NGX_LOG_DEBUG_MUTEX
* NGX_LOG_DEBUG_EVENT
* NGX_LOG_DEBUG_HTTP
* NGX_LOG_DEBUG_MAIL
* NGX_LOG_DEBUG_MYSQL

无论是 ngx_log_error 或是 ngx_log_debug， 都是依赖于 ngx_log_error_core 函数实现的日志打印，其中的参数有：
* level，已在上述给出。
* log，日志对象。
* err， 表示错误码，不为 0 时， Nginx 会输出对应错误码的字符串信息。
* fmt，可变参数，包括字符串和格式化参数的值。

demo:
```c
ngx_log_error(NGX_LOG_ALERT, r->connection->log, 0, "test_flag=%d, test_str=%V, path=%*s",
              mycf->my_flag,        // ngx_flag_t
              &mycf->my_str,        // ngx_str_t
              mycf->my_path->name.len, mycf->my_path->name.data);   // ngx_path_t
```


### 7.1 日志的格式化
这里仅列出部分常用格式，更具体的参考 [src/core/ngx_string.c](https://github.com/nginx/nginx/blob/master/src/core/ngx_string.c)
| format | desc                         | demo                                                                        |
| ------ | ---------------------------- | --------------------------------------------------------------------------- |
| %T     | time_t                       | `ngx_log_error(NGX_LOG_ALERT, cf->log, 0, "time=%T", now)`                  |
| %i     | ngx_int_t                    | `ngx_log_error(NGX_LOG_ALERT, cf->log, 0, "name=%i", 18)`                   |
| %p     | void *                       | `ngx_log_error(NGX_LOG_ALERT, cf->log, 0, "name=%p", &name)`                |
| %V     | ngx_str_t *                  | `ngx_log_error(NGX_LOG_ALERT, cf->log, 0, "name=%V", &name)`                |
| %s     | u_char * (null-terminated)   | `ngx_log_error(NGX_LOG_ALERT, cf->log, 0, "name=%s", "lsj")`                |
| %*s    | size_t + u_char *            | `ngx_log_error(NGX_LOG_ALERT, cf->log, 0, "name=%*s", name.len, name,data)` |

需要注意，上述 `%p`, `%V` 都是用的对象的指针。

### 7.2 日志的获取方式
* 对于处理请求，可以从请求的连接中拿到日志:
```c
ngx_log_error(NGX_LOG_ALERT, r->connection->log, 0, "the log in request->connection");
```
* 若处于非请求中（例如解析配置文件），则可以从 ngx_conf_t 中拿到日志:
```c
ngx_log_error(NGX_LOG_ALERT, cf->log, 0, "the log in ngx_conf");
```

### 7.3 对于复杂变量的打印
上面提到的复杂变量，复杂变量通常需要将其提取为 ngx_str_t 类型的数据才能进行数据打印：
```c
ngx_str_t val;

if (slcf->my_complex == NULL) {
    return NGX_DECLINED;
}

if (ngx_http_complex_value(r, slcf->my_complex, &val) != NGX_OK) {
    return NGX_ERROR;
}

ngx_log_error(NGX_LOG_ALERT, r->connection->log, 0, "my complex: %V", &val);
```

## 八、模块开发步骤

### 8.1 自定义配置结构体
配置结构体用于模块保存自己的配置信息, 需要在ngx解析配置文件时对配置结构体对象进行初始化、赋值。

配置结构体有三种: main, server, location，通常的命名方式为:`ngx_(core|http)_${module}_(main|srv|loc)_conf_t`

```c
// 自定义结构体，用于保存配置项的解析结果
typedef struct{
    ngx_flag_t  switch;
    ngx_array_t formats;
}ngx_http_mylog_conf_t;
```

### 8.2 配置项命令
需要确定有一些什么配置项，以及保存配置项的结构体，nginx启动的时候进行解析，并将相关的数据保存到结构体中。
```c
/*
期望的使用方式:
http / server / location {
    mylog_switch on;
    mylog_format '$remote_addr $http_user_agent $http_cookie $status $body_bytes_sent';
}
*/

// 对配置的解析, 在模块上下文中进行描述
static char* ngx_http_mylog_set_switch(ngx_conf_t *cf,ngx_command_t *cmd,void *conf);
static char* ngx_http_mylog_set_format(ngx_conf_t *cf,ngx_command_t *cmd,void *conf);

// 定义所有的配置项, nginx启动的时候，将会解析配置文件，若解析到该配置，将会设置结构体成员数据
static ngx_command_t ngx_http_mylog_commands[] = {
    {
        ngx_string("mylog_switch"),
        NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
        ngx_http_set_swtich,
        NGX_HTTP_LOC_CONF_OFFSET,
        offsetof(ngx_http_mylog_conf_t,switch),
        NULL
    },
    {
        ngx_string("mylog_format"),
        NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_1MORE,
        ngx_http_set_format,
        NGX_HTTP_LOC_CONF_OFFSET,
        0,
        NULL
    },
    ngx_null_command
};
```
在模块上下文中，将会描述如何解析这些配置。

### 8.3 模块上下文
模块上下文可以理解为模块的配置项在内存中实际存储的信息。
```c
/* 函数声明 */

static ngx_int_t ngx_http_post_init(ngx_conf_t *cf);
static void* ngx_http_mcreate_main_conf(ngx_conf_t *cf);
static void* ngx_http_mcreate_srv_conf(ngx_conf_t *cf);
static char* ngx_http_mmerge_srv_conf(ngx_conf_t *cf,void *prev,void *conf);
static void* ngx_http_mcreate_loc_conf(ngx_conf_t *cf);
static char* ngx_http_mmerge_loc_conf(ngx_conf_t *cf,void *prev,void *conf);

//定义模块上下文结构, 用来指定创建模块上下文(自定义结构体)的方法, 并且可以在这里进行挂载
static ngx_http_module_t ngx_http_module_ctx = {
    NULL,
    ngx_http_post_init,
    NULL,
    NULL,
    NULL,
    NULL,
    ngx_http_create_loc_conf,
    ngx_http_merge_loc_conf
};

// 定义一个模块, 把配置项和模块上下文交给该结构
ngx_module_t ngx_http_mytest_module = {
    NGX_MODULE_V1,
    &ngx_http_module_ctx
    ngx_http_commands，
    NGX_HTTP_MODULE，
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NULL,
    NGX_MODULE_V1_PADDING
};
```

## 九、模块构建
### 9.1 模块准备
一个第三方模块通常是一个目录，包含一个config文件和源码文件:
* config, 用于告诉Nginx当前模块的信息, 以及如何编译当前模块
* 源码文件即第三方模块源代码, 在第三方模块中会定义模块、命令、handler等。

config文件通常是对变量进行定义:
* ngx_addon_name, 尽在configure执行期间使用, 通常是模块名称
* HTTP_MODULES, 保存所有HTTP模块的名称, 模块间用空格分别。一定不能直接覆盖HTTP_MODULES。
* NGX_ADDON_SRCS, 用于指定新增的模块代码, 多个带编译的换代码用空格符相连。一定不能直接覆盖NGX_ADDON_SRCS。
```sh
ngx_addon_name=${module_name}
HTTP_MODULES="$HTTP_MODULES ${ngx_addon_name}"
NGX_ADDON_SRCS="$NGX_ADDON_SRCS $ngx_addon_dir/$src_file"
```
`${module_name}`和`${src_file}`需要第三方模块进行填写

### 9.2 模块编译
当模块目录准备好后, 需要将模块二进制文件编译到ngx中, 这需要重新编译
```sh
$ ./configure ... \
            --add-module=${module_dir}

$ make && make install
```

## 附录
* [[1] Nginx Module API](https://www.nginx.com/resources/wiki/extending/api/)
* [[2] Nginx Development guide](https://nginx.org/en/docs/dev/development_guide.html)
* [[3] Nginx src](https://github.com/nginx/nginx/tree/master/src)