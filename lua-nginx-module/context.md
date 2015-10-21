#content

##ngx_http_lua_ctx_t
从 lua phase handler 章节，我们学习到了 ngx lua module 在 nginx 中插入了各种 handler 以及 filter，这些 handler 和 filter 以及 init_by_lua* 等系列指令中，都有执行 Lua 代码的地方，为了区分 Lua 代码执行的上下文，ngx lua module 添加了上下文信息结构：

```c
typedef struct ngx_http_lua_ctx_s {
    /* for lua_coce_cache off: */
    ngx_http_lua_vm_state_t  *vm_state;

    ngx_http_request_t      *request;
    ngx_http_handler_pt      resume_handler;

    ngx_http_lua_co_ctx_t   *cur_co_ctx; /* co ctx for the current coroutine */

    /* FIXME: we should use rbtree here to prevent O(n) lookup overhead */
    ngx_list_t              *user_co_ctx; /* coroutine contexts for user
                                             coroutines */

    ngx_http_lua_co_ctx_t    entry_co_ctx; /* coroutine context for the
                                              entry coroutine */

    ngx_http_lua_co_ctx_t   *on_abort_co_ctx; /* coroutine context for the
                                                 on_abort thread */

    int                      ctx_ref;  /*  reference to anchor
                                           request ctx data in lua
                                           registry */

    unsigned                 flushing_coros; /* number of coroutines waiting on
                                                ngx.flush(true) */

    ngx_chain_t             *out;  /* buffered output chain for HTTP 1.0 */
    ngx_chain_t             *free_bufs;
    ngx_chain_t             *busy_bufs;
    ngx_chain_t             *free_recv_bufs;

    ngx_http_cleanup_pt     *cleanup;

    ngx_chain_t             *body; /* buffered subrequest response body
                                      chains */

    ngx_chain_t            **last_body; /* for the "body" field */

    ngx_str_t                exec_uri;
    ngx_str_t                exec_args;

    ngx_int_t                exit_code;

    void                    *downstream;  /* can be either
                                             ngx_http_lua_socket_tcp_upstream_t
                                             or ngx_http_lua_co_ctx_t */

    ngx_uint_t               index;              /* index of the current
                                                    subrequest in its parent
                                                    request */

    ngx_http_lua_posted_thread_t   *posted_threads;

    int                      uthreads; /* number of active user threads */

    uint16_t                 context;   /* the current running directive context
                                           (or running phase) for the current
                                           Lua chunk */

    unsigned                 run_post_subrequest:1; /* whether it has run
                                                       post_subrequest
                                                       (for subrequests only) */

    unsigned                 waiting_more_body:1;   /* 1: waiting for more
                                                       request body data;
                                                       0: no need to wait */

    unsigned         co_op:2; /*  coroutine API operation */

    unsigned         exited:1;

    unsigned         eof:1;             /*  1: last_buf has been sent;
                                            0: last_buf not sent yet */

    unsigned         capture:1;  /*  1: response body of current request
                                        is to be captured by the lua
                                        capture filter,
                                     0: not to be captured */


    unsigned         read_body_done:1;      /* 1: request body has been all
                                               read; 0: body has not been
                                               all read */

    unsigned         headers_set:1; /* whether the user has set custom
                                       response headers */

    unsigned         entered_rewrite_phase:1;
    unsigned         entered_access_phase:1;
    unsigned         entered_content_phase:1;

    unsigned         buffering:1; /* HTTP 1.0 response body buffering flag */

    unsigned         no_abort:1; /* prohibit "world abortion" via ngx.exit()
                                    and etc */

    unsigned         header_sent:1; /* r->header_sent is not sufficient for
                                     * this because special header filters
                                     * like ngx_image_filter may intercept
                                     * the header. so we should always test
                                     * both flags. see the test case in
                                     * t/020-subrequest.t */

    unsigned         seen_last_in_filter:1;  /* used by body_filter_by_lua* */
    unsigned         seen_last_for_subreq:1; /* used by body capture filter */
    unsigned         writing_raw_req_socket:1; /* used by raw downstream
                                                  socket */
    unsigned         acquired_raw_req_socket:1;  /* whether a raw req socket
                                                    is acquired */
} ngx_http_lua_ctx_t;
```

这个 ngx_http_lua_ctx_t 上下文结构体信息量挺大，其中的成员在此就不一一描述，基本上跟各个逻辑有关系，具体在探讨到某个功能或者逻辑的时候，我们再深入了解这些结构体成员的作用。

## content

在本章我们只说明 ngx_http_lua_ctx_t 的其中一个成员变量，它就是 content ：

```c
    uint16_t                 context;   /* the current running directive context
                                           (or running phase) for the current
                                           Lua chunk */
```

如同注释上的描述，它是 Lua 当前执行环境的直接上下文，也就是所在的 phase 。
它的值有如下几个定义：

```c
#define NGX_HTTP_LUA_CONTEXT_SET            0x001
#define NGX_HTTP_LUA_CONTEXT_REWRITE        0x002
#define NGX_HTTP_LUA_CONTEXT_ACCESS         0x004
#define NGX_HTTP_LUA_CONTEXT_CONTENT        0x008
#define NGX_HTTP_LUA_CONTEXT_LOG            0x010
#define NGX_HTTP_LUA_CONTEXT_HEADER_FILTER  0x020
#define NGX_HTTP_LUA_CONTEXT_BODY_FILTER    0x040
#define NGX_HTTP_LUA_CONTEXT_TIMER          0x080
#define NGX_HTTP_LUA_CONTEXT_INIT_WORKER    0x100
```

这个成员变量在各个 phase 的 Lua handler  内，Lua 代码执行之前被赋值，比如说在 lua_vm_usage 章节中提及到的

```c
static ngx_int_t
ngx_http_lua_access_by_chunk(lua_State *L, ngx_http_request_t *r)
{
    ...
    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    ngx_http_lua_reset_ctx(r, L, ctx);

    // 配置 ctx 数据
    ctx->entered_access_phase = 1;
    ctx->context = NGX_HTTP_LUA_CONTEXT_ACCESS;
    
    ...
    
    // 把 Lua 代码块放在新协程虚拟机上运行
    rc = ngx_http_lua_run_thread(L, r, ctx, 0);

    c = r->connection;
    if (rc == NGX_AGAIN) {
        // Lua 代码让出了，等待下一次运行时机，接着跑这个 request 子协程
        rc = ngx_http_lua_run_posted_threads(c, L, r, ctx);
    } else if (rc == NGX_DONE) {
        // Lua 代码要求结束请求，接着跑这个 request 子协程
        ngx_http_lua_finalize_request(r, NGX_DONE);
        rc = ngx_http_lua_run_posted_threads(c, L, r, ctx);
    }
    ...
}

```

由上面代码可知：
*  上下文结构都是从 ngx_http_get_module_ctx(r, ngx_http_lua_module) 获取的。
*  在 access phase 作为 http 请求过程中的第一个 Lua handler ，ctx 会被调用 ngx_http_lua_reset_ctx 清理已有数据一遍。
*  ctx->context 被赋值为 NGX_HTTP_LUA_CONTEXT_ACCESS。

类似于 access 阶段，其他阶段也会对 ctx->context 做赋值，然后再执行 Lua 代码。

为了让 Lua 代码知道自己所处的上下文，ngx lua module　还提供了一个接口函数：

```c
static int
ngx_http_lua_get_raw_phase_context(lua_State *L)
{
    ngx_http_request_t      *r;
    ngx_http_lua_ctx_t      *ctx;

    r = lua_touserdata(L, 1);
    if (r == NULL) {
        return 0;
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return 0;
    }

    lua_pushinteger(L, (int) ctx->context);
    return 1;
}

static void
ngx_http_lua_inject_ngx_api(lua_State *L, ngx_http_lua_main_conf_t *lmcf,
    ngx_log_t *log)
{
    lua_createtable(L, 0 /* narr */, 99 /* nrec */);    /* ngx.* */

    lua_pushcfunction(L, ngx_http_lua_get_raw_phase_context);
    lua_setfield(L, -2, "_phase_ctx");
    
    ...
    
    lua_setglobal(L, "ngx");
    
    ...
}

```

从上面代码可以得知：
*  Lua 代码可以调用 ngx._phase_ctx 去获取当前 phase 上下文
*  ngx._phase_ctx 其实是一个 C 函数 ngx_http_lua_get_raw_phase_context 实现
*  ngx_http_lua_get_raw_phase_context 其实是把 ctx->context 作为整数返回值返回给 Lua 调用代码处。
*  当然，因为函数是 _ 开头的，并没有开放成 api 提供 Lua 代码使用，所以从 官方 doc 中也是没有这个函数的任何描述，但其实是可以被 Lua 代码调用的，有兴趣的同学不妨试试。

## phase 校验
另外，在 ngx lua module 中，也有大量针对 phase 做校验的代码，代码如下：

```c

#define ngx_http_lua_check_context(L, ctx, flags)                            \
    if (!((ctx)->context & (flags))) {                                       \
        return luaL_error(L, "API disabled in the context of %s",            \
                          ngx_http_lua_context_name((ctx)->context));        \
    }


static ngx_inline ngx_int_t
ngx_http_lua_ffi_check_context(ngx_http_lua_ctx_t *ctx, unsigned flags,
    u_char *err, size_t *errlen)
{
    if (!(ctx->context & flags)) {
        *errlen = ngx_snprintf(err, *errlen,
                               "API disabled in the context of %s",
                               ngx_http_lua_context_name((ctx)->context))
                  - err;

        return NGX_DECLINED;
    }

    return NGX_OK;
}

```
其中， ngx_http_lua_check_context 宏用于判断上下文是否在某几个 phase 的其中一个中，在 ngx lua module 提供的大量 lua apis 中存在这种 phase 检测，比如：

```c
static int
ngx_http_lua_ngx_echo(lua_State *L, unsigned newline)
{
    ngx_http_request_t          *r;
    ngx_http_lua_ctx_t          *ctx;

    r = ngx_http_lua_get_req(L);

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);

    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT);

    ...
}

ngx.say 以及 ngx.print 的底层实现函数 ngx_http_lua_ngx_echo 中，就使用了 ngx_http_lua_check_context 宏，对上下文是 rewrite 、 access 或者 content 的 phase，才允许正常使用这些api，否则直接报错退出。
这也符合 ngx.say 和 ngx.print 的功能预期。类似的，在大量的 api 都有这样的 phase 校验。

