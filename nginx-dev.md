# Nginx模块开发

<!-- TOC -->

- [Nginx模块开发](#nginx模块开发)
    - [一、概述](#一概述)
    - [二、Nginx重要结构体梳理](#二nginx重要结构体梳理)
        - [2.1 基础结构](#21-基础结构)
            - [2.1.1 *ngx_str_t*](#211-ngx_str_t)
            - [2.1.2 *ngx_array_t*](#212-ngx_array_t)
            - [2.1.3 *ngx_list_t*](#213-ngx_list_t)
            - [2.1.4 *ngx_queue_s*](#214-ngx_queue_s)
            - [2.1.5 *rbtree*](#215-rbtree)
            - [2.1.6 *ngx_table_elt_t*](#216-ngx_table_elt_t)
            - [2.1.7 *ngx_buf_t*](#217-ngx_buf_t)
        - [2.2 内存管理](#22-内存管理)
            - [2.2.1 Heap](#221-heap)
            - [2.2.2 Pool](#222-pool)
            - [2.2.3 共享内存](#223-共享内存)
        - [2.3 第三方模块结构](#23-第三方模块结构)
            - [2.3.1 *ngx_command_t*](#231-ngx_command_t)
            - [2.3.2 *ngx_http_module_t*](#232-ngx_http_module_t)
    - [三、配置解析](#三配置解析)
    - [四、模块处理函数挂载](#四模块处理函数挂载)
        - [4.1 *`NGX_HTTP_CONTENT_PHASE`阶段挂载方案*](#41-ngx_http_content_phase阶段挂载方案)
        - [4.2 *通用挂载方案*](#42-通用挂载方案)
        - [4.3 *处理器返回值*](#43-处理器返回值)
    - [五、模块开发步骤](#五模块开发步骤)
        - [5.1.配置项命令](#51配置项命令)
        - [5.2 模块上下文](#52-模块上下文)

<!-- /TOC -->

## 一、概述
Nginx模块开发主要聚焦于对HTTP模块的开发，而HTTP模块的开发主要聚焦于以下三个部分:
* 配置项命令, 主要用于定义模块在配置文件所用到的配置项, 一个模块可以有多个配置项，分散在nginx配置文件中
* HTTP模块上下文, 上下文主要用于在配置文件解析过程中的各个回调函数, 这些函数用来创造保存配置项目的结构体
* 模块处理函数处理和挂载, 主要是将handler函数挂载到指定的阶段上, http请求到的时候能够对此进行处理

在开始前，需要明确一点，配置的解析仅仅存在于nginx启动(或热加载)时。当配置加载完毕后，每个模块对应的每个语句块的配置相关数据都已经保存在内存中了，当请求到的时候，根据其所处的阶段，可以获取相应的配置信息，以对其进行处理。

## 二、Nginx重要结构体梳理

### 2.1 基础结构
#### 2.1.1 *ngx_str_t*
nginx字符串通常使用ngx_str_t, 该结果包含了字符数组以及字符串长度。
```c
typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
```
ngx_str_t.data并未限制终止字符是什么, 但是在某些配置文件解析的代码中是假定了NULL终止符, 目的是为了简化字符串比较, 并方便传递给系统调用。

* 字符串的基本操作
    * ngx_strcmp()
    * ngx_strncmp()
    * ngx_strstr()
    * ngx_strlen()
    * ngx_strchr()
    * ngx_memcmp()
    * ngx_memset()
    * ngx_memcpy()
    * ngx_memmove()
* 字符串转化及比较
    * ngx_tolower()
    * ngx_toupper()
    * ngx_strlow()
    * ngx_strcasecmp()
    * ngx_strncasecmp()
* 字符串初始化
    * ngx_string()
    * ngx_null_string()
    * ngx_str_set()
    * ngx_str_null()
* 格式化操作
    * ngx_sprintf(buf, fmt, ...)
    * ngx_snprintf(buf, max, fmt, ...)
    * ngx_slprintf(buf, last, fmt, ...)
    * ngx_vslprintf(buf, last, fmt, args)
    * ngx_vsnprintf(buf, max, fmt, args)

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

* array = ngx_array_create(pool, n, size), 从pool中分配内存创建n个元素，每个元素大小为size
* ngx_array_init(array, pool, n, size), 对array进行初始化
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

### 2.2 内存管理
#### 2.2.1 Heap
* p = ngx_alloc(size, log), 从堆中初始化一个大小为size的内存
* p = ngx_calloc(size, log), 和ngx_alloc类似, 但会初始化空间中的所有数据为NULL
* ngx_free(p), 释放内存

#### 2.2.2 Pool
ngx使用的内存通常都是来自于pool, 当pool被销毁时来自pool申请的内存也均会被自动销毁。这让内存控制变得更容易。pool在内部连续内存块中分配对象，当一个内存块写满时，就会申请一个新的内存块，并把新的内存块添加到pool的内存块列表中。
* pool = ngx_create_pool(size, log), 创建一个指定内存大小的size, size至少为NGX_MIN_POOL_SIZE，且应该是NGX_POOL_ALIGNMENT的倍数。
* ngx_pcalloc(pool, size), 从pool中创建一个指定大小的内存，且内存中的所有数据为NULL。
* ngx_destroy_pool(pool), 释放所有内存。

#### 2.2.3 共享内存
共享内存中的数据在ngx的多个进程之间共享，适合做公共缓存使用。

### 2.3 第三方模块结构
* ngx_module_t, 定义了一个模块对象, 该对象有非常重要的两个成员:
    * `ngx_command_t *commands`, 描述了模块所拥有的指令, 以及描述指令的解析和存储方式
    * `void *ctx;`, 配置上下文, 描述了配置文件的初始化、生成方式, 根据type不同有以下几种配置上下文:
        * `ngx_core_module_t *ctx`, 核心模块的配置上下文
        * `ngx_http_module_t *ctx`, http模块的配置上下文

#### 2.3.1 *ngx_command_t*
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
| set | 在解析配置时，碰到了该配置项名后, 将会调用该方法来解析其后面的参数, 并保存在自定义结构体中, nginx预设了一些解析函数 |
| conf | 该参数决定该配置项用creat_`http/srv/loc`_conf中的哪个上下文进行存储, 很明显，一个配置只可能属于一种结构体。  |
| offset | 配置项出现时，将其保存在自定义结构体中的内存偏移量, 主要是为了简化`set`。可用预定宏`offsetof(struct，member)`, 当然若`set`自定义，那么offset无意义 |
| post | 通常为null |


给出了`type`的常用宏，可以通过`|`进行组合功能:

| type | desc |
| ------ | ------ |
| NGX_HTTP_MAIN_CONF | 该配置项可出现在http块内(不包含server块) |
| NGX_HTTP_SRV_CONF | 该配置项可出现在server块内(不包含location块) |
| NGX_HTTP_LOC_CONF | 该配置项可出现在location块内 |
| NGX_CONF_NOARGS | 该配置项没有参数 |
| NGX_CONF_TAKE1 | 可以携带1个参数 |
| NGX_CONF_TAKE2 | 可以携带2个参数 |
| NGX_CONF_TAKE12 | 可以携带1或者2个参数 |

#### 2.3.2 *ngx_http_module_t*
```c
typedef struct {
    /*配置解析前*/
    ngx_int_t   (*preconfiguration)(ngx_conf_t *cf);

    /*配置解析完毕后*/
    ngx_int_t   (*postconfiguration)(ngx_conf_t *cf);

    /*当碰到http块时的回调函数，用于创建http块配置项*/
    void*       (*create_main_conf)(ngx_conf_t *cf);

    /*初始化http块配置项，该函数会在配置项合并前调用*/
    char*       (*init_main_conf)(ngx_conf_t *cf, void *conf);

    /*当碰到http/server块时的回调函数，用于创建server块配置项*/
    void*       (*create_srv_conf)(ngx_conf_t *cf);

    /*用于合并Main级别和Srv级别的server块配置项*/
    char*       (*merge_srv_conf)(ngx_conf_t *cf, void *prev, void *conf);

    /*当碰到http/server/location块时的回调函数，用于创建location块的配置项*/
    void*       (*create_loc_conf)(ngx_conf_t *cf);

    /*用于合并Main,Srv级别和Loc级别的location块配置项*/
    char*       (*merge_loc_conf)(ngx_conf_t *cf, void *prev, void *conf);
} ngx_http_module_t;
```
* postconfiguration, 配置解析完毕后调用, 通常在这个函数中注册某个Phase的handler。
* create_main_conf, 生成http级别的上下文
* create_srv_conf, 生成server级别的上下文
* create_loc_conf, 生成local级别的上下文

## 三、配置解析
现在梳理下配置文件的解析流程:
* 遇到http块时，创建的时http级的上下文, 将调用各模块的create_main_conf函数，同时还会调用create_srv_conf，create_loc_conf
* 调用各模块的http模块上下文的preconfiguration回调函数
* 解析http块内的配置项命令
* 遇到server块时，创建的时server级的上下文, 将调用各模块的create_srv_conf函数，以及create_loc_conf函数
* 解析server块内的配置项命令
* 遇到location块时，创建的时local级的上下文, 将调用各模块的create_loc_conf函数，由于location内可嵌套location，此过程可递归
* 解析location块内的配置项命令
* 调用各模块的init_main_conf方法
* 合并各模块main,srv,loc级别下的srv和loc配置项
* 调用各模块的http模块上下文的postconfiguration回调函数

下面的配置结构，将会生成如下表所示的内存布局结构，由于每个配置项只能存储在一种struct中，所以虽然是统一模块，但是main/srv/loc三种结构体中的配置项互不包含。
```conf
http {
    server {
        location { }
        location { }
    }

    server {
        location { }
        location { }
    }
}
```

| level | main_config | srv_config | loc_config |
| :------: | :------: | :------: | :------: |
| http | main_struct| srv_struct1 | loc_struct1 |
| srv-a | nil | srv_struct2 | loc_struct2 |
| loc | nil | nil | loc_struct3 |
| loc | nil | nil | loc_struct4 |
| srv-b | nil | srv_struct3 | loc_struct5 |
| loc | nil | nil | loc_struct6 |
| loc | nil | nil | loc_struct7 |

`srv_struct1`和`srv_struct2`将会合并，得到`srv-a`块的真正的struct。`srv-b`和location的合并类似。可以看到高级别语句块生成低级别config主要是方便进行合并，也就是说:
* http语句块解析时, 生成的srv_config会用于和server语句块的生成的srv_config合并
* http语句块解析时, 生成的loc_config会用于和server语句块的生成的loc_config合并
* http语句块解析时, 生成的loc_config会用于和location语句块的生成的config合并
* server语句块解析时, 生成的loc_config会用于和location语句块的生成的config合并

## 四、模块处理函数挂载
模块定义好后，上下文的内容也创建好了，解析方式也有了，接着就是http请求来了以后，请求到了某个阶段，将会触发模块对请求进行处理。在指定阶段对请求进行处理，需要将模块的处理函数挂载到对应的阶段上。
### 4.1 *`NGX_HTTP_CONTENT_PHASE`阶段挂载方案*
该方案在配置命令的set回调函数中, 进行挂载, 只适合于`NGX_HTTP_CONTENT_PHASE`。
```c
static char* ngx_http_example_set_func(ngx_conf_t *cf,ngx_command_t *cmd,void *conf){
    ngx_http_core_loc_conf_t clcf = ngx_http_conf_get_module_loc_conf(cf,ngx_http_core_module);
    clcf->handler = ngx_http_example_handler;
    return NGX_CONF_OK;
}
```

### 4.2 *通用挂载方案*
所有的可挂载点，都可以在上下文配置解析完毕后进行挂载，即通过`ngx_int_t   (*postconfiguration)(ngx_conf_t *cf)`回调:
```c
//该函数用于挂载到日志阶段，执行mylog模块的逻辑
static ngx_int_t ngx_http_mytest_handler(ngx_http_request_t *r);

static ngx_int_t postconfiguration(ngx_conf_t *cf){
    ngx_http_handler_pt *h;
    ngx_http_core_main_conf_t *cmcf = ngx_http_conf_get_module_main_conf(cf,ngx_http_core_module);
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

## 五、模块开发步骤
### 5.1.配置项命令
需要确定有一些什么配置项，以及保存配置项的结构体，nginx启动的时候进行解析，并将相关的数据保存到结构体中。
```c
/*
期望的使用方式:
http / server / location {
    mylog_switch on;
    mylog_format '$remote_addr $http_user_agent $http_cookie $status $body_bytes_sent';
}
*/

// 自定义结构体，用于保存配置项的解析结果
typedef struct{
    ngx_flag_t  switch;
    ngx_array_t formats;
}ngx_http_mylog_conf_t;

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

### 5.2 模块上下文
模块上下文可以理解为模块的配置项在内存中实际存储的信息。
```c
/* 函数声明 */
// 
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
