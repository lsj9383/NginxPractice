# HTTP 框架的执行流程s

<!-- TOC -->

- [HTTP 框架的执行流程s](#http-框架的执行流程s)
    - [新建连接](#新建连接)
    - [第一次可读事件的处理](#第一次可读事件的处理)
    - [接收 HTTP 请求行](#接收-http-请求行)
    - [接收 HTTP 头部](#接收-http-头部)
    - [处理 HTTP 请求](#处理-http-请求)
    - [subrequest](#subrequest)
    - [处理 HTTP 包体](#处理-http-包体)
        - [开始接收包体](#开始接收包体)
        - [放弃接收包体](#放弃接收包体)
    - [发送 HTTP 响应](#发送-http-响应)
        - [ngx_http_send_handler](#ngx_http_send_handler)
        - [ngx_http_output_filter](#ngx_http_output_filter)
        - [ngx_http_writer](#ngx_http_writer)
    - [结束 HTTP 请求](#结束-http-请求)
        - [ngx_http_close_connection](#ngx_http_close_connection)
        - [ngx_http_free_request](#ngx_http_free_request)
        - [ngx_http_close_request](#ngx_http_close_request)
        - [ngx_http_finalize_connection](#ngx_http_finalize_connection)
        - [ngx_http_terminate_request](#ngx_http_terminate_request)
        - [ngx_http_finalize_request](#ngx_http_finalize_request)

<!-- /TOC -->

## 新建连接

用户发起 TCP 链接请求时，事件框架会负责把 TCP 连接建立起来，并回调监听方法。对于 HTTP 模块建立的连接，回调方法是 `ngx_http_init_connection` 方法：

```c
void ngx_http_init_connection(ngx_connection_t *c);
```

在该函数中，会设置连接对应的读方法和定时器：

1. 将新建立的连接 c 的读事件设置为 ngx_http_init_request，以及写事件为 ngx_http_empty_handler。
   - `c->read->handler = ngx_http_init_request`。
   - `c->write->handler = ngx_http_empty_handler`，由于此时处于接收 HTTP 头部，所以不应该处理任何写事件，该 handler 仅仅只会记录日志。
1. 如果读事件的 ready 为 1，代表这个套接字已经存在发过来的数据，因此可以调用 `ngx_http_init_request` 处理数据。
1. 设置读事件的定时器，超时时间为 `client_header_timeout`。
1. 将新连接的读事件添加到 EPOLL 中。

## 第一次可读事件的处理

建立连接后，第一次出现可读事件，会触发回调方法 `ngx_http_init_request`。

**注意：**

- 在 `ngx_http_init_connection` 中处理新建立连接事件时，由于TCP三次握手结束时，本身就能在最后的步骤中 client 携带数据到 server，因此在该建立连接的回调中也会调用 `ngx_http_init_request`，来处理建立连接时的数据。

```c
static void ngx_http_init_request(ngx_event_t *reg);
```

该函数的主要目的是为了构造请求对象、分配相关内存、创造上下文结构体的存放空间，并开始准备获取请求行。

1. 如果由于读事件超时（ngx_event_t->timeout == 1 ?），导致触发了 ngx_http_init_request 的执行，直接调用 `ngx_http_close_request` 进行关闭请求。
1. 如果没有超时，进行正常的处理，构建 ngx_http_request_t 对结构体，并将设置 `connection->data = ngx_http_request_t`。
1. 获取 default server 的 ngx_http_core_module 的 srv 配置。
1. 将读事件修改为 `ngx_http_process_request_line` 处理请求行。
1. 为连接创建用户态的内存空间，用于将内核缓冲区的 TCP 流复制到用户态的内存中。这是在 connection 中分配用户态内存空间，该内存大小与配置指令 `client_header_buffer_size` 指定的大小一致。ngx_connection_t->buffer 和 ngx_http_request_t->header_in 共同指向该内存。
1. 为 ngx_http_request_t 构建内存池，这个内存池用于存放和请求相关的数据。
1. 初始化请求上下文数组，用于支持所有模块配置请求上下文。
1. 调用 `ngx_http_process_request_line` 主动处理请求行。

**注意：**

- 现在的 Nginx 源码已经没有 `ngx_http_init_request` 了，取而代之的是 `ngx_http_wait_request_handler`。
- 现在的 Nginx 源码是在 `ngx_http_init_connection` 中获取的 default server 的 ngx_http_core_module 的 srv 配置。
- 书上的流程和现在的 Nginx 代码出入有点大。

## 接收 HTTP 请求行

在接收到 HTTP 数据时，该数据必然是请求行，会触发 `ngx_http_process_request_line` 回调。该回调函数的目的在于解析请求行的信息，对于不同的 HTTP 版本可能会有不同的处理，并且在解析完请求行的同时，会初始化用于解析 HTTP 请求头的相关结构体。

请求行格式：

```http
GET /uri HTTP/1.1
```

**注意**

- 在 `ngx_http_init_request` 接收到可读事件的时候，通常可能已经携带数据了，所以在可读事件回调中会主动触发一次 `ngx_http_process_request_line`。
- 请求行的长度是不固定的，与 URI 长度相关，内核套接字缓冲区大小未必可以一次接收完所有的 HTTP 请求行，因此可能会触发多次 `ngx_http_process_request_line` 来解析请求行。

`ngx_http_process_request_line` 处理步骤如下：

1. 检查读事件是否超时，超时时间为 `client_header_timeout`，若超时，则调用 `ngx_http_close_request` 结束请求。
1. r->header_in 接收缓冲区中若为空，则通过 recv 方法将 Linux Socket 缓冲区中的数据读取 header_in 缓冲区中。
1. recv 如果知道连接是否出错或者客户端已经关闭了连接，若存在一场则调用则结束。
1. recv 如果未能读出数据，则会将读事件添加到定时器和 epoll 中（每次触发事件时，都会从定时器或epoll中移除，因此每次要重新触发，都需要先添加）。
1. recv 如果接收到数据，则会调用 `ngx_http_parse_request_line` 解析请求行，并根据该函数的返回判断是否请求行接收完毕，或者请求行接收失败，或者未接收完毕，需要进一步接收。
1. 如果需要进一步接收请求行，判断 r->header_in 缓冲区是否耗尽，如果耗尽则调用 `ngx_http_alloc_large_header_buffer` 分配更大的接收缓冲区。配置 `large_client_header_buffers` 用来指定该大缓冲区的大小（对于大型请求的客户端才会启用，一般情况下 `client_header_buffers` 就够用了）。
1. 解析完成数据后，将解析处理出的 uri、method_name、http_proto 等等参数设置到 ngx_http_request_t 中。
1. 对于小于 HTTP 1.0 的请求，不会解析 HTTP 请求头，这里省略对于小于 HTTP 1.0 请求的处理。
1. 对于 HTTP 1.0 及其后续的请求，会初始化用于接收请求头的相关结构。
1. 设置回调方法为 `ngx_http_process_request_headers`，开始接收 HTTP 头的数据。
1. 主动调用 `ngx_http_process_request_headers` 处理 HTTP 头的数据（因为处理请求行的时候，缓冲区中可能已经有了 HTTP 请求头的数据）。

## 接收 HTTP 头部

通过 `ngx_http_process_request_headers` 回调函数处理 HTTP 头部。该回调的目的主要是解析 HTTP 头部，根据 Host 找到对应 server 的 srv 级别配置，检查 HTTP 头的合法性等。

这是一个完整的 HTTP 请求：

```http
GET /uri HTTP/1.1
cread: xxxx
username: ttt
content-length: 4

test
```

很明显，HTTP 头部也是一个可变长的数据，请求行和 body 之间都是通过换行来区分的。

请求头的大小是有限制，首次缓冲区分配 `client_buffer_size`，如果不够，会再分配 `large_client_buffer_size` 的缓冲区，如果请求头超过了 `large_client_buffer_size` 则 Nginx 将会返回错误。

`ngx_http_process_request_headers` 的相关处理步骤：

1. 判断是否读事件超时，超时时间为 `client_header_timeout`。如果超时，关闭请求。
1. 检查 r->header_in 缓冲区是否耗尽，如果耗尽则调用 `ngx_http_alloc_large_header_buffer` 分配更大更多的缓冲区。如果无法分配更大的缓冲区，则关闭请求。
1. 通过 recv 接收 Socket 内核缓冲区中的数据，复制到 r->header_in 中。
1. 如果 recv 发现连接错误或者关闭，则关闭请求。
1. 如果 recv 发现连接没有数据，需要进一步接收数据，则将事件重新注册到 epoll 和定时器中。
1. 如果 recv 读出数据，则通过 `ngx_http_parse_header_line` 解析 HTTP 请求头。
1. 当解析出 Header 中的 Host 字段时，会去找到对应的虚拟主机（即对应 server {} 的 ngx_http_core_srv_conf_t 结构体）。
1. 检查 HTTP Headers 的合法性。
1. 调用 `ngx_http_process_request` 开始使用 HTTP 各个模块处理请求。

**注意：**

- 请求头解析完毕后，读写事件的设置在 `ngx_http_process_request` 函数中完成。

## 处理 HTTP 请求

在 Nginx 接收完 HTTP 头部后，通过调用 `ngx_http_process_request` 进行请求的处理。该函数的目的是关闭定时事件，设置读写回调为 `ngx_http_request_handler` 处理请求。

步骤如下：

1. 因为不需要读取数据了，所以从定时器中移除读事件。
1. 将读写事件的回调都设置为 `ngx_htp_request_handler`，该方法。
1. 设置 `r->read_event_handler` 为 `ngx_http_block_reading`（在 `ngx_http_request_handler` 中会回调 `r->read_event_handler`）。
1. 检查 `r->internal`，判断是否为内部请求。
1. 如果是内部请求，表示通过了 rewrite，所以设置 `phase_handler` 序号为 `server_rewrite_index`。
1. 如果是正常请求，将 `phase_handler` 设置为 0，表示从第一个 handler 开始处理请求。
1. 设置 `r->write_event_handler` 为 `ngx_http_core_run_phases` 方法。
1. 执行 `ngx_http_core_run_phases` 方法。
1. 执行 `ngx_http_run_posted_requests` 方法执行 post 请求（子请求）。

在 `ngx_http_core_run_phases` 中，会调用各个 handler 对请求进行处理。

```c
void ngx_http_core_run_phases(ngx_http_request_t *r) {
    ngx_int_t rc;
    ngx_http_phase_handler_t *ph;
    ngx_http_core_main_conf_t *cmcf;

    cmcf = ngx_http_get_module_main_conf(r, ngx_http_core_module);
    
    ph = cmcf->phase_engine.handlers;

    while (ph[r->phase_handler].checker) {
        // 在 checker 中会根据处理请求修改 r->phase_handler 的值，以使得调用不同的 handler
        rc = ph[r->phase_handler].checker(r, &ph[r->phase_handler]);
        if (rc == NGX_OK) {
            return;
        }
    }
}
```

不同阶段的 checker 执行方式是不同的，这里就不展开了。

因为一次 `ngx_http_process_request` 可能并不能将请求处理完，会分多次进行，每当触发事件都会调用 `ngx_http_request_handler`，在该回调中主要是调用对应的读写 handler，即 `r->read_event_handler/r->write_event_handler`，然后调用 `ngx_http_run_posted_requests` 处理子请求。

**注意：**

- 这里的 `r->read_event_handler` 是 `ngx_http_block_reading`，实质上不会进行任何处理。
- 这里的 `r->write_event_handler` 是 `ngx_http_core_run_phases`，重新开始进行各个 handler 的处理。
- 准确来说，ngx 中的 read 事件表示的被动接收网络事件的处理，而 write 是 Nginx 主动进行处理事件，因此 write 事件设置为会进行 handler 处理的事件。例如在子请求处理完毕后，Nginx 都会主动调用 write_event_handler 进行处理。

## subrequest

`ngx_http_run_posted_requests` 会进行子请求的处理，其目的是让每个子请求都进行 `ngx_http_core_run_phases` 处理。

步骤如下：

1. 检查连接是否销毁，如果连接销毁就结束请求。
1. 循环取出每个子请求的 request，并调用 `request->write_event_handler`，该回调就是执行 `ngx_http_core_run_phases`。

```c
void ngx_http_run_posted_requests(ngx_connection_t *c) {
    ngx_http_request_t         *r;
    ngx_http_posted_request_t  *pr;
 
    /* 遍历 */
    for ( ;; ) {
 
    	/* 如果连接已经销毁，则退出 */
        if (c->destroyed) {
            return;
        }
 
        r = c->data;
        pr = r->main->posted_requests;
 
        if (pr == NULL) {
            return;
        }
 
        // 每次 r->main->posted_requests 指针都会指向下一个
        r->main->posted_requests = pr->next;
 
        r = pr->request;
 
        ngx_http_set_log_request(c->log, r);
 
        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                       "http posted request: \"%V?%V\"", &r->uri, &r->args);
 
        /* 调用write handler.   */
        r->write_event_handler(r);
    }
}
```

## 处理 HTTP 包体

HTTP 处理包体有两种方式：

- 把请求中的包体接收到内存或者文件中。
- 丢弃包体。丢弃包体本质上也会触发接收，只是在接收后立即丢弃。

在 HTTP 模块中每进行一类新的操作（包括为一个请求添加新的事件），都需要给请求的引用计数加一，防止操作未结束时请求被 HTTP 框架销毁了。

要求每个操作结束的时候，都调用 `ngx_http_close_request`，该方法会给引用计数减一，当减为 0 时会销毁请求。

派生子请求时，因为这也是个独立的动作，会向 epoll 和定时器添加新的事件，所以引用计数需要加 1，当操作结束的时候调用 `ngx_http_close_request`。

需要注意，子请求的场景下，引用计数是针对原始请求的 count。

在接收 HTTP 包体的时候，也要设置引用计数。

### 开始接收包体

```c
ngx_int_t ngx_http_read_client_request_body(ngx_http_request *r, ngx_http_client_body_handler_pt post_handler);
```

调用完 `ngx_http_read_client_request_body` 方法后，就开启了接收包体的动作，在动作完成后，就会回调 HTTP 模块定义的 post_handler 方法。

用于保存 body 的结构：

```c
typedef struct {
    ngx_temp_file_t *temp_file;                         // 存放 HTTP 包体的临时文件
    ngx_chain_t *bufs;                                  // 接收 HTTP 包体的缓冲区链表，当 buf 无法存放完时就会使用该结构保存
    ngx_buf_t *buf;                                     // 直接接收 HTTP 包体的缓存
    off_t reset;                                        // 根据 content-length 计算出还需要接收的包体的长度
    ngx_chain_t *to_write;                              // 存放要写入文件的包体
    ngx_http_client_body_handler_pt post_handler;       // 接收完包体后的回调方法
} ngx_http_request_body_t;
```

`r->request_body` 成员，就是该结构体。通常，一个 `ngx_http_read_client_request_body` 并不能接收完整的包体，需要反复触发事件，因此会设置读回调为 `ngx_http_read_client_request_body_handler` 以对包体进行反复接收。

`ngx_http_read_client_request_body` 的具体步骤：

1. 原始请求的引用计数加 1。在 post_handler 中一定要调用 `ngx_http_finalize_request` 去结束请求，否则引用技术始终无法清零，从而导致请求无法释放。
1. 检查 `r->request_body` 是否分配过，如果包体为放弃或已经读取过 body，则直接回调 post_handler。
1. 初始化 `r->request_body` 结构，并检查请求的 content-length。
1. 如果 content-length 小于或等于 0，则直接回调 post_handler 方法。
1. 检查接收 headers 时是否已经接收了 body，如果已经接收了 body，这些数据在 header_in 中。
1. 如果 header_in 中已经接收到了全部包体，不用再接收，因此直接调用 post_handler。
1. 如果 header_in 中未接收到全部包体，但是 header_in 的缓冲区中可以容纳足够的剩余包体（根据 content-length 判断），就不再分配包体内存。
1. 如果 header_in 不能存放所有包体，则分配用于接收包体的缓冲区。缓冲区长度由 `client_body_buffer_size` 指定。
1. 设置 `r->read_event_handler` 为 `ngx_http_read_client_body_handler`，以反复接收 body。
1. 调用 `ngx_http_do_read_client_request_body` 读取 TCP 缓冲区中的数据。如果接收到完整请求会直接回调 post_handler。

对于 `ngx_http_do_read_client_request_body` 用于读取 body 并放入缓冲区，整个流程都受定时器限制。

1. 如果缓冲区已满，则将缓冲区写入到文件中。写入到文件中后，重置缓冲区，这样缓冲区就能重复使用了。
1. 调用 recv 从 Socket 中读取包体到缓冲区。如果 recv 错误，则关闭连接，设置 error 标志位为 1，返回错误码 NGX_HTTP_BAD_REQUEST。
1. 根据从 recv 中获取到的数据，更新相关属性，例如 r->request_body->reset 更新剩余未读的字节数，r->request_length 已接收的 body 大小等。
1. 如果数据没有接收完，则添加定时器，将读事件添加到 epoll 中等待再次读取 body。
1. 如果数据读完，则删除定时器，并重置 `r->read_event_handler` 为 `ngx_http_block_reading`。执行 post_handler 回调。

在缓冲区可读时，会触发 `r->read_event_handler` 方法， 此时调用的方法是 `ngx_http_read_client_request_body_handler`，该方法的步骤为：

1. 读事件超时，调用 `ngx_http_finalize_request`，结束请求并发 408 响应。
1. 读事件未超时，调用 `ngx_http_do_read_client_request_body` 接收包体。

### 放弃接收包体

调用特定的函数放弃接收包体：

```c
ngx_int_t ngx_http_discard_request_body(ngx_http_request *r);
```

在该函数中丢弃包体会给引用计数加 1，当包体丢弃完毕会将引用计数减 1（通过调用 `ngx_http_finalize_request`）。

`ngx_http_discard_request_body` 执行步骤如下：

1. 检查请求是否为子请求，如果是子请求则直接返回 NGX_OK，因为子请求实际上是调用内部的函数，不会有网络 io。
1. 移除读事件定时器，这是因为丢弃包体的操作不需要考虑超时（linger_timer 例外）。
1. 请求头的 Content-Length 为 0，表示没有包体，直接返回 NGX_OK。
1. 是否在接收 Header 时已经接收了完整包体，如果是，则直接返回 NGX_OK。
1. 需要反复读取包体进行丢弃，因此设置读回调为 `ngx_http_discarded_request_body_handler`，并添加到 epoll 中。
1. 主动调用 `ngx_http_read_discarded_request_body`，如果已经丢弃完成，则直接返回 NGX_OK。
1. 如果没有丢弃完成，则需要反复丢弃，设置 count 引用计数加一。

在 `ngx_http_read_discarded_request_body` 进行包体的接收和丢弃。

1. 判断丢弃的包体个数是否已经打到了 content-length 长度，如果已经丢弃完毕则设置 `r->read_event_handler=ngx_http_block_reading`。
1. 如果到 Socket 没有可读的内容，直接返回 NGX_AGAIN 等待进一步处理。
1. 由 TCP 连接的套接字缓冲区读取请求包体。
1. 如果需要继续读取，则返回 NGX_AGAIN。如果客户端关闭了连接，也会返回 NGX_OK，告诉调用方不用再丢弃了。
1. 接收到包体后，更新 r->content_length_n，记录丢弃了多少包体。

若一次没有将包体丢弃完，会调用 `ngx_http_discarded_request_body_handler` 方法，该方法主要步骤就三步：

1. 调用 `ngx_http_read_discarded_request_body` 丢弃包体。
1. 包体丢弃成功，调用 `ngx_http_finalize_request` 将引用计数减一。
1. 包体还需要继续丢弃，则将 `ngx_http_discarded_request_body_handler` 重新放入 epoll 事件循环。

## 发送 HTTP 响应

发送 HTTP 响应有两个方面：

- 通过 `ngx_http_send_header` 发送响应头。
- 通过 `ngx_http_output_filter` 发送响应包体。

Nginx 响应发送时，经常会由于响应包体过大，导致无法一次发送，进而会一次有一次的触发写事件回调来不停的发送。进行发送的函数是 `ngx_http_writer` 方法。

**注意：**

- 因为 Nginx 认为开始发送响应意味着请求的结束，因此不需要发送响应完成的回调函数。
- 在调用 `ngx_http_output_filter` 方法进行响应 body 后，需要调用 `ngx_http_finalize_request` 方法，因为该方法会将 r->write_event_handler 设置为 ngx_http_writer，以支持将没有发送完的响应进行发送。

### ngx_http_send_handler

`ngx_http_send_handler` 用于发送响应头，其工作步骤主要就是两个步骤：

1. 依次调用各个过滤模块。
1. 最后一个过滤模块（ngx_header_filter）负责将 headers_out 中的成员变量序列化为字符串并发送出去。

当然 ngx_header_filter 可能无法一次发送完所有的响应头，未发送的部分将会保存在 r->out 中，并且会返回 `NGX_AGAIN` ，调用 `ngx_http_finalize_request` 方法传入该 NGX_AGAIN，Nginx 就了解还有头没有发送完，将会设置 `r->writer_event_hanlder=ngx_http_writer` 用以发送 `r->out` 中的剩余部分。

1. 检查 `r->header_sent` 标志位，如果位 1 则说明这个请求响应头已经发送过了，无需再次发送，直接返回 NGX_OK。
1. 如果没有发送过响应头，首先把 `r->header_sent=1`。
1. 判断是否为子请求，如果是子请求，则不存在发送 header 的 IO 动作，直接返回 NGX_OK。
1. 判断 HTTP 版本是否小于 1.0，如果小于 1.0 则不需要 HTTP 请求头。
1. 计算序列化响应头的字节长度，并在内存池中分配缓冲区。
1. 将响应行、头部按照 HTTP 的规范序列化复制到缓冲区中。
1. 将缓冲区作为参数调用 `ngx_http_write_filter` 方法，将响应头发送出去。

**注意：**

- `ngx_http_write_filter` 如果不能发送完缓冲区的全部内容，会返回 NGX_OK，同时将未发送的部分放到 r->out 中。
- 当 `ngx_http_send_handler` 通过 `ngx_http_write_filter` 不能完全发送内容，则会通过 `ngx_http_finalize_request` 来将 `ngx_http_writer` 设置到 `r->write_event_handler`。

### ngx_http_output_filter

通过 `ngx_http_send_headers` 类似，在调用 `ngx_http_output_fitler` 时，会一次调用所有的过滤模块处理包体，最后一个模块是 `ngx_http_write_filter_module` 其处理方法 `ngx_http_write_filter` 会将数据发送至客户端。

### ngx_http_writer

无论是 ngx_http_send_handler 还是 ngx_http_output_filter 都无法将响应发送完，最后都是通过触发可写事件，回调 `ngx_http_writer` 来进行剩余数据的发送。

ngx_http_writer 的主要步骤如下：

1. 检查是否写事件超时，如果超时可能是有两种原因导致：
   1. 网络异常和客户端不接收响应导致，是真实的网络超时，此时结束请求并返回 408 响应码。
   1. 由于上一次发送响应时过快，由限速导致超时（通过 event 的 delayed 标志位判断），并将 delayed 和 timedout 标志位置为 0。
1. 若数据没有准备好，将写事件添加到定时器（超时时间为 `send_timeout`），并将写时间重新放入 epoll 中，等待再次调用。
1. 若数据已经准备好，调用 `ngx_http_output_filter` 方法发送响应。
1. out 缓冲区数据没有发送完成，则将写事件添加到定时器（超时时间为 `send_timeout`），并将写时间重新放入 epoll 中，等待再次调用。
1. out 缓冲区数据发送完成，重置 `r->write_event_handler` 为 `ngx_http_request_empty_handler`。
1. 调用 `ngx_http_finalize_request` 结束请求。

## 结束 HTTP 请求

Nginx 的一个请求可能会被多个事件触发，当事件依赖某个请求行后，在事件结束后应该销毁掉请求，但是请求可能也被其他事件所依赖，因此并不能简单的去销毁请求。

对于 Nginx，每一个请求都有一个引用计数，每派生一种独立的事件时，都会把引用计数加 1，这样每个事件的动作结束的时候都可以通过 `ngx_http_finalize_request` 结束请求。ngx_http_finalize_request 会对引用计数减一，并在值为 0 时，销毁掉请求

### ngx_http_close_connection

### ngx_http_free_request

### ngx_http_close_request

### ngx_http_finalize_connection

### ngx_http_terminate_request

### ngx_http_finalize_request
