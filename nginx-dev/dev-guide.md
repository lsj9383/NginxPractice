# Development guide

<!-- TOC -->

- [Development guide](#development-guide)
    - [Introduction](#introduction)
        - [代码布局](#代码布局)
        - [头文件](#头文件)
        - [整数](#整数)
        - [通用返回码](#通用返回码)
        - [错误处理](#错误处理)
    - [Strings](#strings)
        - [概览](#概览)
        - [格式化](#格式化)
        - [数字转换](#数字转换)
        - [正则表达式](#正则表达式)
    - [Time](#time)
    - [容器](#容器)
        - [Array](#array)
        - [List](#list)
        - [Hash](#hash)
    - [内存管理](#内存管理)
        - [Heap](#heap)
        - [Pool](#pool)
        - [共享内存](#共享内存)
    - [日志](#日志)
    - [Cycle](#cycle)
    - [Buffer](#buffer)
    - [网络](#网络)
        - [Connection](#connection)
    - [事件](#事件)
        - [Event](#event)
        - [IO Events](#io-events)
        - [Timer Events](#timer-events)
        - [Posted Events](#posted-events)
        - [Event Loop](#event-loop)
    - [进程](#进程)
    - [线程](#线程)
    - [模块](#模块)
        - [添加新的 Modules](#添加新的-modules)
        - [核心模块](#核心模块)
        - [配置指令](#配置指令)
    - [HTTP](#http)
        - [连接](#连接)
        - [请求](#请求)
        - [配置](#配置)
        - [阶段](#阶段)
        - [变量](#变量)
        - [Complex Values](#complex-values)
    - [代码风格](#代码风格)

<!-- /TOC -->

## Introduction

### 代码布局

- auto — 构建脚本
- src
  - core — Nginx 入口，也包括了基本的类型和函数，例如：string, array, log, pool, etc.
  - event — Nginx 事件核心
    - modules — 事件通知模块，例如：epoll, kqueue, select etc.
  - http — 核心 HTTP 模块
    - modules — 其他 HTTP 模块
    - v2 — HTTP/2
  - mail — 邮件模块
  - os — 平台相关代码
    - unix
    - win32
  - stream — Stream 模块

### 头文件

在每个 Nginx 文件中，一定会以两个头文件进行开头：

```c
#include <ngx_config.h>
#include <ngx_core.h>
```

当然，如果涉及 Nginx 的 HTTP 代码，则需要头文件 `ngx_http.h`：

```c
#include <ngx_http.h>
```

Mail 代码需要的头文件：

```c
#include <ngx_mail.h>
```

Stream 代码需要的头文件：

```c
#include <ngx_stream.h>
```

### 整数

通常 Nginx 中使用两种整数：

- ngx_int_t
- ngx_uint_t

### 通用返回码

Nginx 中的函数返回值通常是个以下取值：

- NGX_OK — 操作成功。
- NGX_ERROR — 操作失败。
- NGX_AGAIN — 操作未完成，可以再次调用该函数。
- NGX_DECLINED — 操作拒绝，这并非是错误。
- NGX_BUSY — 资源没有准备好。
- NGX_DONE — 操作完成或者在别处继续。可以用作代替 NGX_OK。
- NGX_ABORT — 函数执行终端。 可以用作代替 NGX_ERROR。

**注意：**

- 上述值在 `core/nginx_core.h` 中，通过 `#define` 进行定义：

  ```c
  #define  NGX_OK          0
  #define  NGX_ERROR      -1
  #define  NGX_AGAIN      -2
  #define  NGX_BUSY       -3
  #define  NGX_DONE       -4
  #define  NGX_DECLINED   -5
  #define  NGX_ABORT      -6
  ```

### 错误处理

宏定义 `ngx_errno` 返回最后一个系统错误码，对于 POSIX 平台，会映射到 `errno`。

宏定义 `ngx_socket_errno` 返回最后一个 socket 错误码，对于 POSIX 平台，会映射到 `errorno`（这和 ngx_errno 其实是一样的）。

ngx_errno 和 ngx_socket_errno 都会触发系统调用，频繁调用可能会影响性能。若错误码需要多次使用，建议通过临时变量（要求为 `ngx_err_t`）进行存储。

宏 `ngx_set_errno(errno)` 和 `ngx_set_socket_errno(errno)` 可以用于设置错误码。

若将 ngx_errno 和 ngx_socket_errno 的值传递给日志函数 ngx_log_error 或 ngx_log_debugX，则会在日志中打印 errno 对应的描述：

```c
ngx_int_t
ngx_my_kill(ngx_pid_t pid, ngx_log_t *log, int signo)
{
    ngx_err_t  err;

    if (kill(pid, signo) == -1) {
        err = ngx_errno;

        ngx_log_error(NGX_LOG_ALERT, log, err, "kill(%P, %d) failed", pid, signo);

        if (err == NGX_ESRCH) {
            return 2;
        }

        return 1;
    }

    return 0;
}
```

## Strings

### 概览

Nginx 提供了字符串类型 `ngx_str_t`，其定义如下：

```c
typedef struct {
    size_t      len;
    u_char     *data;
} ngx_str_t;
```

其中，`len` 代表字符串长度，`data` 代表字符串数据。在 data 打到 len 长度后，可能会以 NULL 终止，也可能不会，在大多数情况下是没有用 NULL 终止的。

在某些特定的代码中，data 可能会用 NULL 终止，简化字符串的比较。

ngx 的字符串函数位于 `src/core/ngx_string.h` 中，

- 其中一些用以包装标准 C 函数：
  - ngx_strcmp()
  - ngx_strncmp()
  - ngx_strstr()
  - ngx_strlen()
  - ngx_strchr()
  - ngx_memcmp()
  - ngx_memset()
  - ngx_memcpy()
  - ngx_memmove()
- 另一些是 Nginx 特有的：
  -- ngx_memzero() — 用 0 甜宠内存.
  - ngx_explicit_memzero() — Does the same as ngx_memzero(), but this call is never removed by the compiler's dead store elimination optimization. This function can be used to clear sensitive data such as passwords and keys.
  - ngx_cpymem() — Does the same as ngx_memcpy(), but returns the final destination address This one is handy for appending multiple strings in a row.
  - ngx_movemem() — Does the same as ngx_memmove(), but returns the final destination address.
  - ngx_strlchr() — Searches for a character in a string, delimited by two pointers.
- nginx string 的初始化函数：
  - ngx_string(text) — static initializer for the ngx_str_t type from the C string literal text

    ```c
    ngx_str_t name = ngx_string("env");
    ```

  - ngx_null_string — static empty string initializer for the ngx_str_t type

    ```c
    ngx_str_t name = ngx_null_string;
    ```

  - ngx_str_set(str, text) — initializes string str of ngx_str_t * type with the C string literal text

    ```c
    ngx_str_t name = ngx_null_string;
    ngx_str_set(&name, "env");
    ```

  - ngx_str_null(str) — initializes string str of ngx_str_t * type with the empty string

    ```c
    ngx_str_t name = ngx_string("env");
    ngx_str_null(&name);
    ```

### 格式化

- `ngx_sprintf(buf, fmt, ...)`
- `ngx_snprintf(buf, max, fmt, ...)`
- `ngx_slprintf(buf, last, fmt, ...)`
- `ngx_vslprintf(buf, last, fmt, args)`
- `ngx_vsnprintf(buf, max, fmt, args)`

格式化时，不同的数据类型需要使用不同的格式化符号，完整的符号在 `src/core/ngx_string.c` 中进行定义，这里仅列出了部分常用的：

- %O — `off_t`
- %T — `time_t`
- %z — `ssize_t`
- %i — `ngx_int_t`
- %ui — `ngx_uint_t`
- %p — `void *`
- %V — `ngx_str_t *`
- %s — `u_char * (null-terminated)`
- %*s — `size_t + u_char *`

这是一个使用格式化函数的用例：

```c
u_char      buf[NGX_INT_T_LEN];
size_t      len;
ngx_uint_t  n;

/* set n here */

len = ngx_sprintf(buf, "%ui", n) — buf;
```

### 数字转换

### 正则表达式

Nginx 的正则表达式接口封装了 `PCRE` 的库。

要使用正则表达式，首先需要变异正则表达式，这通常在配置阶段完成。

**注意：**

- 因为 PCRE 是可选的，所以使用正则表达式的代码，需要通过 `NGX_PCRE` 进行保护，避免因为没开 PCRE 导致系统崩溃：

  ```c
  #if (NGX_PCRE)
  ngx_regex_t          *re;
  ngx_regex_compile_t   rc;
  
  u_char                errstr[NGX_MAX_CONF_ERRSTR];
  
  ngx_str_t  value = ngx_string("message (\\d\\d\\d).*Codeword is '(?<cw>\\w+)'");
  
  ngx_memzero(&rc, sizeof(ngx_regex_compile_t));
  
  rc.pattern = value;
  rc.pool = cf->pool;
  rc.err.len = NGX_MAX_CONF_ERRSTR;
  rc.err.data = errstr;
  /* rc.options are passed as is to pcre_compile() */
  
  if (ngx_regex_compile(&rc) != NGX_OK) {
      ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, "%V", &rc.err);
      return NGX_CONF_ERROR;
  }
  
  re = rc.regex;
  #endif
  ```

<!-- TODO(lu) 补充正则表达式的使用 -->

## Time

Nginx 中，使用 `ngx_time_t` 结构表示时间，该结构有三种时间的表示方式：秒、毫秒、GMT 偏移。

```c
// time_t 等价于 long

typedef struct {
    time_t      sec;
    ngx_uint_t  msec;
    ngx_int_t   gmtoff;
} ngx_time_t;
```

若要获取当前时间，建议直接使用可用的全局变量：

- 可以通过 `ngx_time()` 获得当前秒级时间戳，通过 `ngx_timeofday()` 获得当前时间的 `ngx_time_t` 结构。

  ```c
  // 当前时间存储与 ngx_cached_time 中
  extern volatile ngx_time_t  *ngx_cached_time;

  #define ngx_time()           ngx_cached_time->sec
  #define ngx_timeofday()      (ngx_time_t *) ngx_cached_time
  ```

- 若需要直接使用格式化后的时间，可以使用以下全局变量，都是 ngx_str_t 结构。这些格式化时间通常用于日志记录，以及 HTTP 头部。
  - ngx_cached_err_log_time — Used in error log entries: "1970/09/28 12:00:00"
  - ngx_cached_http_log_time — Used in HTTP access log entries: "28/Sep/1970:12:00:00 +0600"
  - ngx_cached_syslog_time — Used in syslog entries: "Sep 28 12:00:00"
  - ngx_cached_http_time — Used in HTTP headers: "Mon, 28 Sep 1970 06:00:00 GMT"
  - ngx_cached_http_log_iso8601 — The ISO 8601 standard format: "1970-09-28T12:00:00+06:00"

  The time is always updated when nginx returns to the event loop from system calls. To update the time immediately, call ngx_time_update(), or ngx_time_sigsafe_update() if updating the time in the signal handler context.

当 Nginx 从系统调用会到 Event Loop 时，时间总是会进行更新。如果在信号处理上下文中，可以通过 `ngx_time_update()` 或 `ngx_time_sigsafe_update()` 更新时间。

Nginx 中，可以通过以下函数 `ngx_gmtime(time_t, ngx_tm_t*)` 将 `time_t`（秒级时间戳）转换为 `ngx_tm_t`（等价于 `struct tm`，Linux 时间结构体）。

在 HTTP 中，Cookie 和 Headers 中会涉及到格式化时间戳，可以通过下面两个函数将秒级时间戳转换：

- `ngx_http_time(u_char *buf, time_t t)`，获得适用于 HTTP Headers 中的时间格式（例如：`Mon, 28 Sep 1970 06:00:00 GMT`）。
- `ngx_http_cookie_time(u_char *buf, time_t t)`，获得适用于 Cookie 中的时间格式（例如：`Thu, 31-Dec-37 23:55:55 GMT`）。

**注意：**

- 上述返回的字符串都是放在 `u_char* buf` 参数中。

## 容器

这里的容器并非 Docker 用于运行应用程序的容器，而是进行数据存储的数据结构。

### Array

Nginx 的数组类型使用 `ngx_array_t`，其结构如下：

```c
typedef struct {
    void        *elts;
    ngx_uint_t   nelts;
    size_t       size;
    ngx_uint_t   nalloc;
    ngx_pool_t  *pool;
} ngx_array_t;
```

上述结构中：

- elts，指向数组首地址。
- nelts，数组中已使用的元素个数。
- size，每个元素大小（字节）。
- nalloc，共分配的元素个数。

![](https://box.kancloud.cn/2016-09-01_57c7edcf4bebd.jpg)

元素大小需要在数组初始化时设置。很明显 `ngx_array_t` 所占用的字节数为：`nalloc * size`，当 `ngx_array_t` 的内存空间被分配完后，若需要继续添加元素，则会自动启动一个新的、更大的内存空间。

- `ngx_array_create(pool, n, size)` 在内存池 Pool 中创建数组。通常用于在 heap 中分配数组。需要指定每个元素的大小 size，以及数组分配的元素个数 n。
- `ngx_array_init(array, pool, n, size)`，可以对以分配的数组进行初始化。通常用于对 stack 中分配的数组初始化。需要指定每个元素的大小 size，以及数组分配的元素个数 n。

```c
/* create an array of strings with preallocated memory for 10 elements */
ngx_array_t  *a;
a = ngx_array_create(pool, 10, sizeof(ngx_str_t));

/* initialize string array for 10 elements */
ngx_array_t  b;
ngx_array_init(&b, pool, 10, sizeof(ngx_str_t));
```

可以通过以下函数向数组中添加元素：

- ngx_array_push(a) 向尾部添加一个元素，并返回该元素的指针。
- ngx_array_push_n(a, n) 向尾部添加 n 个元素，并返回首个元素的指针。

```c
ngx_array_t  *a;
a = ngx_array_create(pool, 10, sizeof(ngx_str_t));
ngx_str_t    *s = ngx_array_push(a);

ngx_array_t  b;
ngx_array_init(&b, pool, 10, sizeof(ngx_str_t));
ngx_str_t    *ss = ngx_array_push_n(&b, 3);
```

### List

Nginx 中，List 是每个节点均为数组的链表，优化了大量数据的插入，结构为 `ngx_list_t`，定义如下：

```c
// 链表中的一个节点，里面对应了一个数组空间，每个元素存储在该数组中。
typedef struct ngx_list_part_s  ngx_list_part_t;

struct ngx_list_part_s {
    void             *elts;
    ngx_uint_t        nelts;
    ngx_list_part_t  *next;
};

typedef struct {
    ngx_list_part_t  *last;
    ngx_list_part_t   part;
    size_t            size;
    ngx_uint_t        nalloc;
    ngx_pool_t       *pool;
} ngx_list_t;
```

数据解释：

- ngx_list_t
  - *last，链表中的尾节点指针，便于快速找到尾部插入下一个节点。
  - part，链表首节点，用于遍历链表。
  - size，节点数组元素的大小，实际上就是存储单位，如果是字符串，则 `size=sizeof(ngx_str_t)`。
  - nalloc，每个节点数组最大个数。
- ngx_list_part_t
  - *elts，数组首地址。
  - nelts，当数组中已使用的元素个数。
  - *next，下一个节点，

![](https://box.kancloud.cn/2016-09-01_57c7edcf652d0.jpg)

在使用前，List 必须经过初始化或创建：

- `ngx_list_create(pool, n, size)` 在 Heap 中创建链表。
- `ngx_list_init(list, pool, n, size)` 初始化链表，通常用于 Stack 中分配的内存。

要在 List 中添加元素，可以使用 `ngx_list_push(list)`。每次新增元素都是发生在最后一个节点数组中的末尾元素，若超过节点数组的大小，就会在链表后面新增一个节点，用以存放新数据。

对于链表的遍历：

```c
ngx_str_t        *v;
ngx_uint_t        i;
ngx_list_t       *list;
ngx_list_part_t  *part;

list = ngx_list_create(pool, 100, sizeof(ngx_str_t));
if (list == NULL) { /* error */ }

/* add items to the list */

v = ngx_list_push(list);
if (v == NULL) { /* error */ }
ngx_str_set(v, "foo");

v = ngx_list_push(list);
if (v == NULL) { /* error */ }
ngx_str_set(v, "bar");

/* iterate over the list */

part = &list->part;
v = part->elts;

for (i = 0; /* void */; i++) {

    if (i >= part->nelts) {
        if (part->next == NULL) {
            break;
        }

        part = part->next;
        v = part->elts;
        i = 0;
    }

    ngx_do_smth(&v[i]);
}
```

List 不支持删除，因为删除操作很麻烦，对中间元素删除时会伴随这非常多的元素移位调整。但可以标记元素为已删除，这需要元素的结构体支持，例如 `ngx_table_elt_t` 中设置属性 `hash` 为 0 表示元素不存在。

### Hash

Hash 表相关函数在 `src/core/ngx_hash.h` 中声明，支持完全匹配与通配。

在使用 Hash 前，需要对 Hash 中的数据量有个评估，以便 Nginx 以最佳的效率使用 Hash。Nginx 提供了 max_size 和 bucket_size 两个参数来进行配置（可以参考文档 [Setting up hashes](https://nginx.org/en/docs/hash.html)）。

Hash 初始化设置对象由 `ngx_hash_init_t` 提供，Hash 对象本身是 `ngx_hash_t`。

```c
ngx_hash_t       foo_hash;
ngx_hash_init_t  hash;

hash.hash = &foo_hash;
hash.key = ngx_hash_key;
hash.max_size = 512;
hash.bucket_size = ngx_align(64, ngx_cacheline_size);
hash.name = "foo_hash";
hash.pool = cf->pool;
hash.temp_pool = cf->temp_pool;
```

其中 `hash.key` 是通过字符串创建的 hash 值，这是一个整数。有两个方法可以从字符串创建 Hash 值：

- `ngx_hash_key(data, len)`
- `ngx_hash_key_lc(data, len)`，将字符串数据转换为小数生成 hash 值。

当 Hash 表构建好以后，可以使用 `ngx_hash_find(hash, key, name, len)` 去查询元素：

```c
ngx_str_t    k1 = ngx_string("key1");
ngx_uint_t   key = ngx_hash_key(k1.data, k1.len);

// ngx_hash_t foo_hash 已经构建好
my_data_t   *data = ngx_hash_find(&foo_hash, key, k1.data, k1.len);
if (data == NULL) {
    /* key not found */
}
```

## 内存管理

### Heap

可以从进程的堆空间中分配内存，这和 C 语言的 malloc 是类似的。

- ngx_alloc(size, log) — 从 Heap 中分配内存，本质上是包装了 `malloc()` 函数，以提供日志支持，以便记录错误和调试信息。
- ngx_calloc(size, log) — 从 Heap 中分配内存，类似 `ngx_alloc`，并且分配完成后会用 0 值初始化内存。
- ngx_memalign(alignment, size, log) — 从 Heap 中分配对齐的内存，这是对 `posix_memalign()` 的包装。在提供该功能的平台上才有真实用处，否则会退化为 `ngx_alloc()`。
- ngx_free(p) — 释放分配的内存空间。

### Pool

大多数 Nginx 中的内存分配在内存池 Pool 中进行，在 Pool 中分配的内存，可以在 Pool 被销毁掉时自动释放掉，这提供了非常好的性能和内存控制。

Pool 会申请连续的内存块，并在其中分配对象，若内存快用尽，则生成新的内存块，并添加到 Pool 中。当请求的数据过大，一个内存块中无法分配时，则会转化到 System Allocator，并返回头指针，该指针会存放在 Pool，以便后续自动销毁。

Nginx 的 Pool 用 `ngx_pool_t` 表示。

- ngx_create_pool(size, log) — Create a pool with specified block size. The pool object returned is allocated in the pool as well. The size should be at least NGX_MIN_POOL_SIZE and a multiple of NGX_POOL_ALIGNMENT.
- ngx_destroy_pool(pool) — Free all pool memory, including the pool object itself.
- ngx_palloc(pool, size) — Allocate aligned memory from the specified pool.
- ngx_pcalloc(pool, size) — Allocate aligned memory from the specified pool and fill it with zeroes.
- ngx_pnalloc(pool, size) — Allocate unaligned memory from the specified pool. Mostly used for allocating strings.
- ngx_pfree(pool, p) — Free memory that was previously allocated in the specified pool. Only allocations that result from requests forwarded to the system allocator can be freed.

```c
u_char      *p;
ngx_str_t   *s;
ngx_pool_t  *pool;

pool = ngx_create_pool(1024, log);
if (pool == NULL) { /* error */ }

s = ngx_palloc(pool, sizeof(ngx_str_t));
if (s == NULL) { /* error */ }
ngx_str_set(s, "foo");

p = ngx_pnalloc(pool, 3);
if (p == NULL) { /* error */ }
ngx_memcpy(p, "foo", 3);
```

可以注册 Pool 的清理回调函数，通过 `ngx_pool_cleanup_add(pool, size)` 实现，该函数返回 `ngx_pool_cleanup_t` 结构。

使用者可以通过 `ngx_pool_cleanup_t` 指定回调函数以及回调上下文 data，其中回调上下文的大小由 `ngx_pool_cleanup_add(pool, size)` 中的 size 参数指定。

```c
ngx_pool_cleanup_t  *cln;

cln = ngx_pool_cleanup_add(pool, 0);
if (cln == NULL) { /* error */ }

cln->handler = ngx_my_cleanup;
cln->data = "foo";

...

static void
ngx_my_cleanup(void *data)
{
    u_char  *msg = data;

    ngx_do_smth(msg);
}
```

### 共享内存

共享内存用于在 Nginx 的多个进程中共享数据。

通过 `ngx_shared_memory_add(cf, name, size, tag)` 函数，可以添加一个新的共享内存条目，返回的数据结构为 `ngx_shm_zone_t`。

- name，指定共享区域的名称。每个共享区域必须有一个唯一的 name。
- size，共享区域的大小。
- tag，共享区域的标记。通常将模块的地址，作为 tag 进行传参，这样可以在相同的模块中，通过 name 来区分共享区域。

当创建共享内存时，若 name 和 tag 已经存在，则直接重用共享内存。如果 name 存在，但是 tag 不同，则返回错误。

共享内存 `ngx_shm_zone_t` 结构体有如下字段：

- init — Initialization callback, called after the shared zone is mapped to actual memory
- data — Data context, used to pass arbitrary data to the init callback
- noreuse — Flag that disables reuse of a shared zone from the old cycle
- tag — Shared zone tag
- shm — Platform-specific object of type ngx_shm_t, having at least the following fields:
  - addr — Mapped shared memory address, initially NULL
  - size — Shared memory size
  - name — Shared memory name
  - log — Shared memory log
  - exists — Flag that indicates shared memory was inherited from the master process (Windows-specific)

在 `ngx_init_cycle()` 完成配置解析后，将会把共享内存映射到实际内存中，在 Linux 中，就是使用 mmap 系统调用完成。

当 Nginx 的共享内存创建完成（mmap 映射后），会自动创建用于分配内存的 slab pool，结构为 `ngx_slab_pool_t`，可以通过 `(ngx_slab_pool_t *) shm_zone->shm.addr` 使用。

- `ngx_slab_alloc(pool, size)`，pool 为 `ngx_slab_pool_t`。
- `ngx_slab_calloc(pool, size)`，pool 为 `ngx_slab_pool_t`。
- `ngx_slab_free(pool, p)`，pool 为 `ngx_slab_pool_t`。

```c
ngx_str_t        name;
ngx_foo_ctx_t   *ctx;
ngx_shm_zone_t  *shm_zone;

ngx_str_set(&name, "foo");

/* allocate shared zone context */
ctx = ngx_pcalloc(cf->pool, sizeof(ngx_foo_ctx_t));
if (ctx == NULL) {
    /* error */
}

/* add an entry for 64k shared zone */
shm_zone = ngx_shared_memory_add(cf, &name, 65536, &ngx_foo_module);
if (shm_zone == NULL) {
    /* error */
}

/* register init callback and context */
shm_zone->init = ngx_foo_init_zone;
shm_zone->data = ctx;


...


static ngx_int_t
ngx_foo_init_zone(ngx_shm_zone_t *shm_zone, void *data)
{
    ngx_foo_ctx_t  *octx = data;

    size_t            len;
    ngx_foo_ctx_t    *ctx;
    ngx_slab_pool_t  *shpool;

    value = shm_zone->data;

    if (octx) {
        /* reusing a shared zone from old cycle */
        ctx->value = octx->value;
        return NGX_OK;
    }

    shpool = (ngx_slab_pool_t *) shm_zone->shm.addr;

    if (shm_zone->shm.exists) {
        /* initialize shared zone context in Windows nginx worker */
        ctx->value = shpool->data;
        return NGX_OK;
    }

    /* initialize shared zone */

    ctx->value = ngx_slab_alloc(shpool, sizeof(ngx_uint_t));
    if (ctx->value == NULL) {
        return NGX_ERROR;
    }

    shpool->data = ctx->value;

    return NGX_OK;
}
```

为了保护在多个进程中使用共享内存，可以使用 `ngx_slab_pool_t` 结构中的互斥锁，即：

- ngx_shmtx_lock(&shpool->mutex)
- ngx_shmtx_unlock(&shpool->mutex)

## 日志

对于日志记录，Nginx 使用 `ngx_log_t` 对象。Nginx 的 Logger 支持以下几种输出：

- stderr — Logging to standard error (stderr)
- file — Logging to a file
- syslog — Logging to syslog
- memory — Logging to internal memory storage for development purposes; the memory can be accessed later with a debugger

Logger 实例中可以包含一系列的 Logger，并通过 `next` 字段相连，在输入日志时会触发相连的 Logger 也进行记录。

对于每个 Logger，都设定了严重性级别的阈值，打到严重性大于等于 Logger 的阈值时，日志消息才会进行输出。支持以下日志级别：

日志级别：

- NGX_LOG_EMERG
- NGX_LOG_ALERT
- NGX_LOG_CRIT
- NGX_LOG_ERR
- NGX_LOG_WARN
- NGX_LOG_NOTICE
- NGX_LOG_INFO
- NGX_LOG_DEBUG

对于 NGX_LOG_DEBUG 可以使用调试掩码：

- NGX_LOG_DEBUG_CORE
- NGX_LOG_DEBUG_ALLOC
- NGX_LOG_DEBUG_MUTEX
- NGX_LOG_DEBUG_EVENT
- NGX_LOG_DEBUG_HTTP
- NGX_LOG_DEBUG_MAIL
- NGX_LOG_DEBUG_STREAM

Nginx 提供以下的日志行为：

- ngx_log_error(level, log, err, fmt, ...)，打印日志，并且没有格式化参数要求。
- ngx_log_debug0(level, log, err, fmt)，无格式化参数日志。
- ngx_log_debug1(level, log, err, fmt, arg1)，仅一个格式化参数的日志。

**注意：**

- 对于 `ngx_log_debugX` 最多支持 X=8，即 8 个格式化参数的日志输出。这是因为某些平台不支持可变参数所做的一种让步。

```c
/* specify what is currently done */
log->action = "sending mp4 to client";

/* error and debug log */
ngx_log_error(NGX_LOG_INFO, c->log, 0, "client prematurely closed connection");

ngx_log_debug2(NGX_LOG_DEBUG_HTTP, mp4->file.log, 0, "mp4 start:%ui, length:%ui", mp4->start, mp4->length);
```

## Cycle

一个 Cycle 对象，存储着由特定配置创建的 Nginx 运行时的上下文，其类型为 `ngx_cycle_t`。当前 Cycel 引用了一个全局变量 `ngx_cycle`，并且别每个 Nginx Worker 进程启动时继承。

每当 Nginx 的配置重新加载，新的配置文件会创建新的 Cycle 并替换旧的 Cycle。

Cycle 对象，由 `ngx_init_cycle()` 函数创建，该函数会以上一个 Cycle 作为参数，并寻找上一个 Cycle 的配置文件以继承尽可能多的资源。

在 Nginx 启动时，会有一个 Cycle 占位符，被称为 "init cycle"，该占位符会在 Nginx 启动时作为上一个 Cycle 对象。

Cycle 包括如下成员：

- pool — Cycle pool. 每个 Cycle 都会管理一个内存池，用以分配和管理内存。
- log — Cycle log. 从老 cycle 中继承，当配置文件加载完毕，会指向 `new_log`。
- new_log — Cycle log, 根据配置文件创建，受 error_log 指令的影响.
- connections, connection_n
  - `connections` 是 `ngx_connection_t` 结构的数组。由 Event 模块初始化每个 Nginx Worker 进程时创建。
  - `worker_connections` 指令的值将会被设置到 `connection_n`，以指定最大的连接个数。
- free_connections, free_connection_n — 空闲连接和空闲连接的个数。这个空闲连接并非是一个处于空闲状态的真实连接，而是一个未用于通信的 `ngx_connection_t` 对象。Nginx 连接对象都是预分配好的，且初始都处于 free_connections 中，连接建立时会直接从中获取连接，断开连接后放回。
- files, files_n - 用于将文件描述符映射到 Nginx 连接的数组。 事件模块使用此映射，该模块具有NGX_USE_FD_EVENT标志（当前为轮询和devpoll）。
- conf_ctx — 核心模块配置数组。 在读取nginx配置文件期间将创建并填充配置。
- modules, modules_n — `ngx_module_t` 结构的数组，当前配置加载的 ngx_module_t 类型的模块（静态和动态）的数组。
- listening — `ngx_listening_t` 结构的数组。Listening 对象由不同的模块调用 `ngx_create_listening()` 函数添加。监听的 Socket 由 Listening 对象创建。
- paths — Array of paths of type ngx_path_t. Paths are added by calling the function ngx_add_path() from modules which are going to operate on certain directories. These directories are created by nginx after reading configuration, if missing. Moreover, two handlers can be added for each path:
  - path loader — Executes only once in 60 seconds after starting or reloading nginx. Normally, the loader reads the directory and stores data in nginx shared memory. The handler is called from the dedicated nginx process “nginx cache loader”.
  - path manager — Executes periodically. Normally, the manager removes old files from the directory and updates nginx memory to reflect the changes. The handler is called from the dedicated “nginx cache manager” process.
- open_files — `ngx_open_file_t` 结构的数组，这个结构是打开的文件对象。通过调用 `ngx_conf_open_file()` 函数可以打开文件对象。当前对日志文件的打开采用的是该函数。读取配置后，Nginx 将打开 open_files 列表中的所有文件，并将每个文件描述符存储在对象的 fd 字段中。这些文件以追加模式打开，如果缺少则创建。收到重新打开信号（通常为USR1）后，Nginx Worker 将重新打开列表中的文件。在这种情况下，fd 字段中的描述符将更改为新值。
- shared_memory — 共享内存的列表，在调用 `ngx_shared_memory_add()` 函数是会在该列表中添加。共享区域在所有nginx进程中都映射到相同的地址范围，并用于共享公共数据，例如HTTP缓存内存树。

## Buffer

对于输入输出操作，Nginx 提供了缓冲区结构 `ngx_buf_t`。通常该数据结构用于指向需要输出和输入的数据。例如在进行 HTTP 响应，需要将 buffer 引用到需要返回的数据，并交给 Nginx 处理。

缓冲区可以引用内存和文件中的数据。缓冲区中的内存是独立的，和 `ngx_buf_t` 无关。

`ngx_buf_t` 有如下字段：

- start, end - 给 buffer 的指定存储块边界。如果是内存，则是内存偏移。
- post, last - 本次进行处理的边界。通常是 start .. end 的子范围。一个 buffer 可能会被反复处理多次，因此本次处理的范围在 post 和 last 给出。如果一个 buffer 只会处理一次，则 `start == post` 并且 `end == last`。通常用于内存中的数据。
- file_pos, file_last - 类似 post 和 last，不过这两个指定的是文件里的偏移。
- tag - 用以区分 buffer 的唯一值，不同的 Nginx 模块会用不同的 Tag，通常用于 buffer 重用。
- file - 文件对象。
- temporary - 表示这段数据可写。
- memoery - 表示这段数据只读。
- in_file - 表示处理的是文件。
- flush - 表示需要执行 flush 操作。
- recycled - 表示可回收。
- sync
- last_buf - 表示是否为 chain 的最后一块 buffer。（buffer 可以通 chain 串起来）
- last_in_chain。
- shadow
- last_shadow
- temp_file - 指明缓冲区在临时文件中。

buffer 可以通过 chain 连接在一起。chain 是 `ngx_chain_t` 的链表，如下定义：

```c
typedef struct ngx_chain_s  ngx_chain_t;

struct ngx_chain_s {
    ngx_buf_t    *buf;
    ngx_chain_t  *next;
};
```

- buf 指向当前的 `ngx_buf_t`。
- next 指向下一个 `ngx_chain_t`。

这是一个使用 chains 和 buffers 的 Demo：

```c
ngx_chain_t *
ngx_get_my_chain(ngx_pool_t *pool)
{
    ngx_buf_t    *b;
    ngx_chain_t  *out, *cl, **ll;

    /* first buf */
    cl = ngx_alloc_chain_link(pool);
    if (cl == NULL) { /* error */ }

    b = ngx_calloc_buf(pool);
    if (b == NULL) { /* error */ }

    b->start = (u_char *) "foo";
    b->pos = b->start;
    b->end = b->start + 3;
    b->last = b->end;
    b->memory = 1; /* read-only memory */

    cl->buf = b;
    out = cl;
    ll = &cl->next;

    /* second buf */
    cl = ngx_alloc_chain_link(pool);
    if (cl == NULL) { /* error */ }

    b = ngx_create_temp_buf(pool, 3);
    if (b == NULL) { /* error */ }

    b->last = ngx_cpymem(b->last, "foo", 3);

    cl->buf = b;
    cl->next = NULL;
    *ll = cl;

    return out;
}
```

## 网络

### Connection

连接类型在 Nginx 中是 `ngx_connection_t`，该类型是 Socket 描述符的一个包装。包含了如下字段：

- fd - Socket 描述符。
- data
- read, write - 连接对应的读写事件。
- recv, send, recv_chain, send_chain - 连接的 IO 操作。
- pool - 连接涉及到的内存池。accept 连接时建立，并在连接结束时销毁（ngx_connection_t 都是预分配的，但是只有在实际连接建立是会拿出来使用，这时才会触发 pool 的生成）。
- log - 连接 logger。
- socketaddr, socklen, addr_text - 远端 socket 地址信息。
- local_socketaddr, local_socklen - 本地 socket 地址信息。
- proxy_protocol_addr, proxy_protocol_port - 代理客户端地址和端口。
- ssl - SSL 上下文。
- idle - 指明连接处于空闲状态，例如 keepalive 请求中两次请求中间的状态。
- reusable - 指明连接处于可以重新使用的状态。
- close - 指明连接已关闭。

Nginx 连接透明地封装了 SSL 层，在这种情况下，连接的 SSL 字段包含一个指向 `ngx_ssl_connection_t` 结构的指针。

`worker_connections` 指令限制了每个 Nginx Worker 的连接个数。所欲的连接结构都是在 Worker 启动预先创建好的，并存放在 Cycle 对象中。

## 事件

### Event

Event 对象 `ngx_event_t` 在 Nginx 提供了一个通知机制以告知发生了什么事件。

`ngx_event_t` 对象包含以下属性：

- data — 事件处理中使用的上下文，通常指向事件相关连接的指针。
- handler — 当某个事件发生时的回调函数。
- write — 表明是一个写事件。若没有该标识，代表是一个读事件。
- active — 表明该事件已经注册，并用于 IO 通知。通常来自 epoll、poll、kqueue 的通知。
ready — 收到 IO 通知的标识。
delayed — 表示因为限流的原因，IO 延迟。
timer — 红黑树节点，用于插入定时器树。
timer_set — 表明设置了定时器，并且未超时。
timedout — 表明定时器超时。
eof — 表明在读的时候遇到了 EOF。
pending_eof — Flag indicating that EOF is pending on the socket, even though there may be some data available before it. The flag is delivered via the EPOLLRDHUP epoll event or EV_EOF kqueue flag.
error — 表示在读和写时遇到了错误。
cancelable — Timer event flag indicating that the event should be ignored while shutting down the worker. Graceful worker shutdown is delayed until there are no non-cancelable timer events scheduled.
posted — 表明事件已经提交到了队列。
queue — Queue node for posting the event to a queue.

一个 Nginx 事件可以配置多个通知，例如一个事件即可配置定时器超时触发，也可以同时配置为 IO 事件时触发。

### IO Events

通过 `ngx_get_connection()` 函数获得的连接，有两个很重要的成员：

- `c-read` 读事件。
- `c-write` 写事件。

这两个成员就是 IO 事件对象，用以 Socket 将数据准备好时进行通知（对于读事件，即接收到了数据触发 `c-read`，对于写事件，即有能够写数据时触发 `c-write`。）。

Nginx 的这些事件都是边缘触发的，意味着 Socket 状态变化时才会触发一次。例如，读取 Socket 的部分数据并不会触发 Nginx 发送重复的读通知（`c-read`）。甚至即便是水平触发的 IO 通知机制，Nginx 也会转换成边缘触发，以保持统一。

### Timer Events

一个 Event 可以被设置为在 timeout 时发送通知。事件使用的定时器可以用毫秒为单位。

- `ngx_add_timer(ev, timer)` 为一个事件设置 timeout。
- `ngx_del_timer(ev)` 删除先去设置的 timeout。

全局变量 `ngx_event_timer_rbtree` 存储了当前所有设置的 timeout，其中的 key 就是事件发生的事件，且类型时 `ngx_msec_t`。

使用红黑树来维护 Timer Event，使得快速插入和删除成为了可能，访问最近超时事件的也非常的方便，即树中最左边的节点。

### Posted Events

一个事件可以被提交，意味着在它的 Handler 将在当前事件循环迭代的某个时刻被调用。

提交 Events 用于简化代码和避免栈溢出的好方式。提交的事件保存在 Post Queue 中。通常提交到事件循环中的事件，会在 IO 和 Timer 事件处理完毕后，进行调用处理。

- `ngx_post_event(ev, q)` 提交事件 ev 到 Post Queue。
- `ngx_delete_posted_event(ev)` 删除 ev。
- `ngx_event_process_posted()`

这是一个 Demo：

```c
void
ngx_my_connection_read(ngx_connection_t *c)
{
    ngx_event_t  *rev;

    rev = c->read;

    ngx_add_timer(rev, 1000);

    rev->handler = ngx_my_read_handler;

    ngx_my_read(rev);
}


void
ngx_my_read_handler(ngx_event_t *rev)
{
    ssize_t            n;
    ngx_connection_t  *c;
    u_char             buf[256];

    if (rev->timedout) { /* timeout expired */ }

    c = rev->data;

    while (rev->ready) {
        n = c->recv(c, buf, sizeof(buf));

        if (n == NGX_AGAIN) {
            break;
        }

        if (n == NGX_ERROR) { /* error */ }

        /* process buf */
    }

    if (ngx_handle_read_event(rev, 0) != NGX_OK) { /* error */ }
}
```

### Event Loop

## 进程

Nginx 的进程有几种类型，并且进程类型在全局变量 `ngx_process` 中，为以下值之一：

- NGX_PROCESS_MASTER
- NGX_PROCESS_WORKER
- NGX_PROCESS_SINGLE
- NGX_PROCESS_HELPER

Nginx 会处理以下信号：

- NGX_SHUTDOWN_SIGNAL（Linux 为 SIGQUIT）
- NGX_TERMINATE_SIGNAL（Linux 为 SIGTERM）
- NGX_NOACCEPT_SIGNAL（Linux 为 SIGWINCH）
- NGX_RECONFIGURE_SIGNAL（Linux 为 SIGHUP）
- NGX_REOPEN_SIGNAL（Linux 为 SIGUSR1）
- NGX_CHANGEBIN_SIGNAL（Linux 为 SIGUSR2）

## 线程

## 模块

### 添加新的 Modules

每个独立的 Nginx 模块都位于一个独立的，并包含两个文件：

- config
- 模块源代码

config 文件包含了 Nginx 集成模块所需要的所有信息，例如：

```sh
ngx_module_type=CORE
ngx_module_name=ngx_foo_module
ngx_module_srcs="$ngx_addon_dir/ngx_foo_module.c"

. auto/module

ngx_addon_name=$ngx_module_name
```

config 其实就是一个 POSIX Shell 脚本，需要去设置以下的变量：

- ngx_module_type - 构建模块的类型。可能值有：
  - CORE
  - HTTP
  - HTTP_FILTER
  - HTTP_INIT_FILTER
  - HTTP_AUX_FILTER
  - MAIL
  - STREAM
  - MISC
- ngx_module_name - 模块的名称，若过一个源文件中有多个模块，需要用空格分割。
- ngx_addon_name - 模块名称，用于 `.configure` 通常和 `ngx_module_name` 一样。
- ngx_module_srcs - 用空格分割的源文件列表。`$ngx_addon_dir` 可以用于表示模块目录的路径。
- ngx_module_incs
- ngx_module_libs
- ngx_module_link
- ngx_module_order

以下模块通常用作参考：

- `ngx_http_copy_filter_module`
- `ngx_http_write_filter_module`

通过静态库的方式编译模块，可以在 `./configure` 时使用 `--add-module=/path/to/module` 参数。

### 核心模块

Nginx 的大部分功能都是基于模块实现的。模块源文件必须包含 `ngx_module_t` 的全局变量，定义如下：

```c
struct ngx_module_s {

    /* private part is omitted */

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

    /* stubs for future extensions are omitted */
};
```

每个模块都有这样的特性：

- 将自己的私有数据保存在 ctx 字段中。
- 根据 commands 识别配置文件中的指令。
- 并且可以在 Nginx 的某些阶段可以触发回调。

模块 type 取值会影响 ctx 使用的结构体，type 的取值有：

- NGX_CORE_MODULE
- NGX_EVENT_MODULE
- NGX_HTTP_MODULE
- NGX_MAIL_MODULE
- NGX_STREAM_MODULE

其中 NGX_CORE_MODULE 是 Nginx 最基本的模块，也是最通用，最底层的。其他模块均基于 CORE 模块实现。核心模块包括：

- ngx_core_module
- ngx_errlog_module
- ngx_regex_module
- ngx_thread_pool_module
- ngx_openssl_module
- HTTP 模块、Stream 模块、Mail 模块、Event 模块也是属于 NGX_CORE_MODULE

核心模块的 ctx 字段结构如下：

```c
typedef struct {
    ngx_str_t             name;
    void               *(*create_conf)(ngx_cycle_t *cycle);
    char               *(*init_conf)(ngx_cycle_t *cycle, void *conf);
} ngx_core_module_t;
```

- name 字段是模块名称。
- create_conf 是创建模块配置的函数指针。Nginx 解析新配置钱调用。用以配置内存并设置默认值。
- init_conf 是初始化模块配置的函数指针。成功解析所有配置之后调用。

这是一个核心模块是 Demo：

```c
#include <ngx_config.h>
#include <ngx_core.h>


typedef struct {
    ngx_flag_t  enable;
} ngx_foo_conf_t;


static void *ngx_foo_create_conf(ngx_cycle_t *cycle);
static char *ngx_foo_init_conf(ngx_cycle_t *cycle, void *conf);

static char *ngx_foo_enable(ngx_conf_t *cf, void *post, void *data);
static ngx_conf_post_t  ngx_foo_enable_post = { ngx_foo_enable };


static ngx_command_t  ngx_foo_commands[] = {

    { ngx_string("foo_enabled"),
      NGX_MAIN_CONF|NGX_DIRECT_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      0,
      offsetof(ngx_foo_conf_t, enable),
      &ngx_foo_enable_post },

      ngx_null_command
};


static ngx_core_module_t  ngx_foo_module_ctx = {
    ngx_string("foo"),
    ngx_foo_create_conf,
    ngx_foo_init_conf
};


ngx_module_t  ngx_foo_module = {
    NGX_MODULE_V1,
    &ngx_foo_module_ctx,                   /* module context */
    ngx_foo_commands,                      /* module directives */
    NGX_CORE_MODULE,                       /* module type */
    NULL,                                  /* init master */
    NULL,                                  /* init module */
    NULL,                                  /* init process */
    NULL,                                  /* init thread */
    NULL,                                  /* exit thread */
    NULL,                                  /* exit process */
    NULL,                                  /* exit master */
    NGX_MODULE_V1_PADDING
};


static void *
ngx_foo_create_conf(ngx_cycle_t *cycle)
{
    ngx_foo_conf_t  *fcf;

    fcf = ngx_pcalloc(cycle->pool, sizeof(ngx_foo_conf_t));
    if (fcf == NULL) {
        return NULL;
    }

    fcf->enable = NGX_CONF_UNSET;

    return fcf;
}


static char *
ngx_foo_init_conf(ngx_cycle_t *cycle, void *conf)
{
    ngx_foo_conf_t *fcf = conf;

    ngx_conf_init_value(fcf->enable, 0);

    return NGX_CONF_OK;
}


static char *
ngx_foo_enable(ngx_conf_t *cf, void *post, void *data)
{
    ngx_flag_t  *fp = data;

    if (*fp == 0) {
        return NGX_CONF_OK;
    }

    ngx_log_error(NGX_LOG_NOTICE, cf->log, 0, "Foo Module is enabled");

    return NGX_CONF_OK;
}

```

### 配置指令

`ngx_command_t` 结构定义了一个配置指令，这个结构描述了如何处理参数，以及调用哪些处理程序。每个模块都支持提供一个配置指令的数组。

```c
typedef struct ngx_command_s  ngx_command_t;

struct ngx_command_s {
    ngx_str_t             name;
    ngx_uint_t            type;
    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
    ngx_uint_t            conf;
    ngx_uint_t            offset;
    void                 *post;
};
```

配置结构的数组用一个特殊的 `ngx_null_command` 进行终止。

- name 字段是指令的名称。例如 `proxy_pass`，`worker_processes` 等。
- type 通过 bit 来标识配置的一些特征，例如支持什么参数，可以出现在哪些配置块。
  - 参数相关 bit 标识别：
    - NGX_CONF_NOARGS — 指令不支持参数。
    - NGX_CONF_1MORE — 指令支持至少一个参数。
    - NGX_CONF_2MORE — 指令支持至少两个参数。
    - NGX_CONF_TAKE1..NGX_CONF_TAKE7 — 指令固定的参数个数。
    - NGX_CONF_TAKE12, NGX_CONF_TAKE13, NGX_CONF_TAKE23, NGX_CONF_TAKE123,  NGX_CONF_TAKE1234 — 指令支持不同的参数个数。例如： NGX_CONF_TAKE12 意味着支持一个或两个参数。
  - 指令支持的参数类型
    - NGX_CONF_BLOCK - 指令的参数必须是一个语句块。
    - NGX_CONF_FLAG - 指令的参数必须是一个，且必须为 on/off。
  - 指令可以出现在哪些配置块：
    - NGX_MAIN_CONF
    - NGX_HTTP_MAIN_CONF
    - NGX_HTTP_SRV_CONF
    - NGX_HTTP_LOC_CONF
    - NGX_HTTP_UPS_CONF
    - NGX_HTTP_SIF_CONF
    - NGX_HTTP_LIF_CONF
    - NGX_HTTP_LMT_CONF
    - NGX_EVENT_CONF
    - NGX_DIRECT_CONF
    - ...
- set 字段定义了一个处理程序，该处理程序处理指令并将解析后的值存储到配置中。
- conf 字段定义了哪个配置结构体传递给目录处理程序。
  - NGX_HTTP_MAIN_CONF_OFFSET
  - NGX_HTTP_SRV_CONF_OFFSET
  - NGX_HTTP_LOC_CONF_OFFSET
  - ...
- offset 定义了配置值在模块的配置结构体中的偏移，set 使用 Nginx 定义的处理程序时使用。通常的处理方式是 set 使用 Nginx 定义的处理程序，offset 使用 `offsetof()` 宏。
- 对于 post 字段有两个目的：
  - 可以定义在主程序完成后调用的处理程序
  - 将其他数据传递给主程序。

post 在第一种情况下，需要使用指向处理程序的指针来初始化 `ngx_conf_post_t`，例如：

```c
static char *ngx_do_foo(ngx_conf_t *cf, void *post, void *data);
static ngx_conf_post_t  ngx_foo_post = { ngx_do_foo };

static ngx_command_t  ngx_foo_commands[] = {

    { ...
      &ngx_foo_post },

      ngx_null_command
};
```

## HTTP

### 连接

### 请求

### 配置

每个 HTTP 模块都有三种类型的配置：

- Main 配置 - 应用于 http 块中的配置，用于模块的全局配置。
- Server 配置 - 应用于 server 块中的配置。作用于指定 server 中的配置。
- Location 配置 - 应用于 location、if 或者 limit_except 语句块。作用于指定的 location 中的配置。

配置结构在 Nginx 配置阶段调用函数进行创建、分配内存、初始化、并合并。

这是一个 Demo：

```c
typedef struct {
    ngx_uint_t  foo;
} ngx_http_foo_loc_conf_t;


static ngx_http_module_t  ngx_http_foo_module_ctx = {
    NULL,                                  /* preconfiguration */
    NULL,                                  /* postconfiguration */

    NULL,                                  /* create main configuration */
    NULL,                                  /* init main configuration */

    NULL,                                  /* create server configuration */
    NULL,                                  /* merge server configuration */

    ngx_http_foo_create_loc_conf,          /* create location configuration */
    ngx_http_foo_merge_loc_conf            /* merge location configuration */
};


static void *
ngx_http_foo_create_loc_conf(ngx_conf_t *cf)
{
    ngx_http_foo_loc_conf_t  *conf;

    conf = ngx_pcalloc(cf->pool, sizeof(ngx_http_foo_loc_conf_t));
    if (conf == NULL) {
        return NULL;
    }

    conf->foo = NGX_CONF_UNSET_UINT;

    return conf;
}


static char *
ngx_http_foo_merge_loc_conf(ngx_conf_t *cf, void *parent, void *child)
{
    ngx_http_foo_loc_conf_t *prev = parent;
    ngx_http_foo_loc_conf_t *conf = child;

    ngx_conf_merge_uint_value(conf->foo, prev->foo, 1);
}
```

如上例琐事，`ngx_http_foo_create_loc_conf()` 函数创建一个新的配置结构，并且通过 `ngx_http_foo_merge_loc_conf()` 将不同层级的配置进行合并。

以下宏用于访问 HTTP 模块中的配置文件，它们都需要使用 `ngx_conf_t` 作为第一个参数：

- `ngx_http_conf_get_module_main_conf(cf, module)`
- `ngx_http_conf_get_module_srv_conf(cf, module)`
- `ngx_http_conf_get_module_loc_conf(cf, module)`

这些宏通常在加载配置的处理程序中调用：

```c
static ngx_int_t ngx_http_foo_handler(ngx_http_request_t *r);


static ngx_command_t  ngx_http_foo_commands[] = {

    { ngx_string("foo"),
      NGX_HTTP_LOC_CONF|NGX_CONF_NOARGS,
      ngx_http_foo,
      0,
      0,
      NULL },

      ngx_null_command
};


static char *
ngx_http_foo(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_core_loc_conf_t  *clcf;

    clcf = ngx_http_conf_get_module_loc_conf(cf, ngx_http_core_module);
    clcf->handler = ngx_http_bar_handler;

    return NGX_CONF_OK;
}
```

若 HTTP 处于请求 handler，可以使用以下宏：

- `ngx_http_get_module_main_conf(r, module)`
- `ngx_http_get_module_srv_conf(r, module)`
- `ngx_http_get_module_loc_conf(r, module)`

这些宏接收首个参数是 HTTP request 结构（`ngx_http_request_t`）。

请求的 Mail 配置是不可变更的。Server 配置可变更。由于重写操作或者内部重定向，用于处理请求的 Location 配置可能会多次变更。

```c
static ngx_int_t
ngx_http_foo_handler(ngx_http_request_t *r)
{
    ngx_http_foo_loc_conf_t  *flcf;

    flcf = ngx_http_get_module_loc_conf(r, ngx_http_foo_module);

    ...
}
```

### 阶段

### 变量

### Complex Values

## 代码风格
