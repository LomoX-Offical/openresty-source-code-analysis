# Lua 执行起来

## Lua 代码执行流程

Lua 代码允许被植入到各种执行阶段中，为开发效率带来极大的提升，但如何正确的理解和使用 Lua 代码又将会成为开发团队的新课题， Lua 代码如何被执行起来的，这非常值得我们好好阅读代码并深入了解。

还记得上文提到了lua-nginx-module 的模块定义中有两个比较关键的函数指针：

*  ngx_http_lua_init_worker
*  ngx_http_lua_init

这两个函数其实是用于在 nginx 启动过程中初始化并且真正执行 Lua 代码的地方，而其中被执行 Lua 代码又是从指令 

*  init_by_lua 以及 init_by_lua_file
*  init_worker_by_lua 以及 init_worker_by_lua_file

简单看看 init_by_lua 和 init_worker_by_lua 的指令定义

```c

    { ngx_string("init_by_lua"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_http_lua_init_by_lua,
      NGX_HTTP_MAIN_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_init_by_inline },

    { ngx_string("init_worker_by_lua"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_http_lua_init_worker_by_lua,
      NGX_HTTP_MAIN_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_init_worker_by_inline },

```
可以看出该指令只能在 http 配置块上有效，并且只能带一个参数，执行函数是 ngx_http_lua_init_by_lua，带一个函数参数 ngx_http_lua_init_by_inline。

我们也都知道指令的正确用法像是这样子的：

```nginx
init_by_lua 'cjson = require "cjson"';
```
就这样，Lua 代码 'cjson = require "cjson"' 就被作为字符串类型传入 init_by_lua 指令的执行函数中，也就是 ngx_http_lua_init_by_lua。

我们先来看看  ngx_http_lua_init_by_lua 的代码：

```c

char *
ngx_http_lua_init_by_lua(ngx_conf_t *cf, ngx_command_t *cmd,
    void *conf)
{
    u_char                      *name;
    ngx_str_t                   *value;
    ngx_http_lua_main_conf_t    *lmcf = conf;

    // 设置了执行函数，在上例的环境中就是 (void *) ngx_http_lua_init_by_inline
    lmcf->init_handler = (ngx_http_lua_conf_handler_pt) cmd->post;

    if (cmd->post == ngx_http_lua_init_by_file) {
        // 把 name 输出成 value 参数的绝对路径
        name = ngx_http_lua_rebase_path(cf->pool, value[1].data,
                                        value[1].len);
        if (name == NULL) {
            return NGX_CONF_ERROR;
        }

        lmcf->init_src.data = name;
        lmcf->init_src.len = ngx_strlen(name);

    } else {
        lmcf->init_src = value[1];
    }

    return NGX_CONF_OK;
}

```

ngx_http_lua_init_by_lua 函数就做了两件事情：

*  把本模块的配置 init_handler 函数指针设置成 cmd->post 也就是 ngx_http_lua_init_by_inline。
*  把本模块的配置 init_src 源码设置成该指令的参数。

现在我们回过头来看看 ngx_http_lua_init 函数，按之前模块定义，它是在 postconfiguration 的阶段被调用，也就是在配置解析阶段的 ngx_http_lua_init_by_lua 执行完之后被调用，下面是代码摘要：

```c
static ngx_int_t
ngx_http_lua_init(ngx_conf_t *cf)
{

    if (multi_http_blocks || lmcf->requires_capture_filter) {
        // 这里做 filter 列表加入，稍后再做代码阅读
        rc = ngx_http_lua_capture_filter_init(cf);
        if (rc != NGX_OK) {
            return rc;
        }
    }

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    if (lmcf->requires_rewrite) {
        // 往 rewrite 阶段 插入 handler
        h = ngx_array_push(&cmcf->phases[NGX_HTTP_REWRITE_PHASE].handlers);
        if (h == NULL) {
            return NGX_ERROR;
        }

        *h = ngx_http_lua_rewrite_handler;
    }

    if (lmcf->requires_access) {
        // 往 access 阶段 插入 handler
        h = ngx_array_push(&cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers);
        if (h == NULL) {
            return NGX_ERROR;
        }

        *h = ngx_http_lua_access_handler;
    }

    dd("requires log: %d", (int) lmcf->requires_log);

    if (lmcf->requires_log) {
        // 往 log 阶段 插入 handler
        arr = &cmcf->phases[NGX_HTTP_LOG_PHASE].handlers;
        h = ngx_array_push(arr);
        if (h == NULL) {
            return NGX_ERROR;
        }

        if (arr->nelts > 1) {
            h = arr->elts;
            ngx_memmove(&h[1], h,
                        (arr->nelts - 1) * sizeof(ngx_http_handler_pt));
        }

        *h = ngx_http_lua_log_handler;
    }

    if (multi_http_blocks || lmcf->requires_header_filter) {
        // 往 header filter 链表 插入 filter 函数 ngx_http_lua_header_filter
        rc = ngx_http_lua_header_filter_init();
        if (rc != NGX_OK) {
            return rc;
        }
    }

    if (multi_http_blocks || lmcf->requires_body_filter) {
        // 往 body filter 链表 插入 filter 函数 ngx_http_lua_body_filter
        rc = ngx_http_lua_body_filter_init();
        if (rc != NGX_OK) {
            return rc;
        }
    }

    // 这里判断 Lua 虚拟机是否被初始化
    if (lmcf->lua == NULL) {

        // 这是干啥用的？ todo
        ngx_http_lua_content_length_hash =
                                  ngx_http_lua_hash_literal("content-length");
        ngx_http_lua_location_hash = ngx_http_lua_hash_literal("location");

        // 这里是初始化 Lua 虚拟机了
        lmcf->lua = ngx_http_lua_init_vm(NULL, cf->cycle, cf->pool, lmcf,
                                         cf->log, NULL);

        if (!lmcf->requires_shm && lmcf->init_handler) {

            // 这里是运行 init_handler 回调，并根据返回值确定程序是否继续运行
            if (lmcf->init_handler(cf->log, lmcf, lmcf->lua) != NGX_OK) {
                return NGX_ERROR;
            }
        }


    }

    return NGX_OK;
}
```

这个函数做的事情有几件：
*  初始化阶段事先就往各个阶段（rewrite，access 以及 log）插入不同的 handler ，除了 content 阶段。
*  初始化了 Lua 虚拟机，使得 Lua 代码可以使用 Lua 虚拟机执行。
*  调用了 lmcf->init_handler 回调函数，这个 init_handler 在之前的 ngx_http_lua_init_by_lua 函数已经被设置为指向 ngx_http_lua_init_by_inline 或者 ngx_http_lua_init_by_file 了

也就是说，在 postconfiguration 阶段，Lua 完成了虚拟机的初始化，并且调用了 init_by_lua* 指定的 Lua 代码。

我们在回过头来看看 ngx_http_lua_init_worker_by_lua：

```c
char *
ngx_http_lua_init_worker_by_lua(ngx_conf_t *cf, ngx_command_t *cmd,
    void *conf)
{
    value = cf->args->elts;

    // 设置了执行函数，在上例的环境中就是 (void *) ngx_http_lua_init_worker_by_inline
    lmcf->init_worker_handler = (ngx_http_lua_conf_handler_pt) cmd->post;

    if (cmd->post == ngx_http_lua_init_worker_by_file) {

        // 把 name 输出成 value 参数的绝对路径
        name = ngx_http_lua_rebase_path(cf->pool, value[1].data,
                                        value[1].len);
        if (name == NULL) {
            return NGX_CONF_ERROR;
        }

        lmcf->init_worker_src.data = name;
        lmcf->init_worker_src.len = ngx_strlen(name);

    } else {
        lmcf->init_worker_src = value[1];
    }

    return NGX_CONF_OK;
}

```
ngx_http_lua_init_worker_by_lua 代码结构基本与 ngx_http_lua_init_by_lua 一样，差异在于设置的回调函数指针不一样，前者是 lmcf->init_handler 后者是 lmcf->init_worker_handler ，也就意味着被设置的回调函数在不同的地方执行。
没错，我们看看 ngx_http_lua_init_worker  函数，它被模块定义成是在 init process 阶段调用，其实是在 worker 进程的初始化阶段执行，这个阶段在配置读取阶段之后发生，也就是说在 init_by_lua* 和 ngx_http_lua_init 的执行之后，ngx_http_lua_init_worker 会被调用，我们来看看他里边实现了什么：

```c
ngx_int_t
ngx_http_lua_init_worker(ngx_cycle_t *cycle)
{
    // 这里给conf 创建了 临时内存池   
    ngx_memzero(&conf, sizeof(ngx_conf_t));

    conf.temp_pool = ngx_create_pool(NGX_CYCLE_POOL_SIZE, cycle->log);
    if (conf.temp_pool == NULL) {
        return NGX_ERROR;
    }
    conf.temp_pool->log = cycle->log;

    // 后面很大一段都是复制 cycle 到 fake_cycle 这里
    fake_cycle = ngx_palloc(cycle->pool, sizeof(ngx_cycle_t));
    if (fake_cycle == NULL) {
        goto failed;
    }

    ngx_memcpy(fake_cycle, cycle, sizeof(ngx_cycle_t));

    // 包括复制 listening 侦听列表
    if (ngx_array_init(&fake_cycle->listening, cycle->pool,
                       cycle->listening.nelts || 1,
                       sizeof(ngx_listening_t))
        != NGX_OK)
    {
        goto failed;
    }

#if defined(nginx_version) && nginx_version >= 1003007

    // 包括复制 paths 路径列表
    if (ngx_array_init(&fake_cycle->paths, cycle->pool, cycle->paths.nelts || 1,
                       sizeof(ngx_path_t *))
        != NGX_OK)
    {
        goto failed;
    }

#endif

    part = &cycle->open_files.part;
    ofile = part->elts;

    // 包括复制 open_files 已打开文件列表
    if (ngx_list_init(&fake_cycle->open_files, cycle->pool, part->nelts || 1,
                      sizeof(ngx_open_file_t))
        != NGX_OK)
    {
        goto failed;
    }

    for (i = 0; /* void */ ; i++) {

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }
            part = part->next;
            ofile = part->elts;
            i = 0;
        }

        file = ngx_list_push(&fake_cycle->open_files);
        if (file == NULL) {
            goto failed;
        }

        ngx_memcpy(file, ofile, sizeof(ngx_open_file_t));
    }

    // 还包括复制 shared_memory 共享内存列表
    if (ngx_list_init(&fake_cycle->shared_memory, cycle->pool, 1,
                      sizeof(ngx_shm_zone_t))
        != NGX_OK)
    {
        goto failed;
    }

    conf.ctx = &http_ctx;
    conf.cycle = fake_cycle;
    conf.pool = fake_cycle->pool;
    conf.log = cycle->log;

    // 每个 http 类型模块的配置也都复制过来了
    http_ctx.loc_conf = ngx_pcalloc(conf.pool,
                                    sizeof(void *) * ngx_http_max_module);
    if (http_ctx.loc_conf == NULL) {
        return NGX_ERROR;
    }

    http_ctx.srv_conf = ngx_pcalloc(conf.pool,
                                    sizeof(void *) * ngx_http_max_module);
    if (http_ctx.srv_conf == NULL) {
        return NGX_ERROR;
    }

    for (i = 0; ngx_modules[i]; i++) {
        if (ngx_modules[i]->type != NGX_HTTP_MODULE) {
            continue;
        }

        module = ngx_modules[i]->ctx;

        if (module->create_srv_conf) {
            cur = module->create_srv_conf(&conf);
            if (cur == NULL) {
                return NGX_ERROR;
            }

            if (module->merge_srv_conf) {
                prev = module->create_srv_conf(&conf);
                if (prev == NULL) {
                    return NGX_ERROR;
                }

                rv = module->merge_srv_conf(&conf, prev, cur);
                if (rv != NGX_CONF_OK) {
                    goto failed;
                }
            }

            http_ctx.srv_conf[ngx_modules[i]->ctx_index] = cur;
        }

        if (module->create_loc_conf) {
            cur = module->create_loc_conf(&conf);
            if (cur == NULL) {
                return NGX_ERROR;
            }

            if (module->merge_loc_conf) {
                prev = module->create_loc_conf(&conf);
                if (prev == NULL) {
                    return NGX_ERROR;
                }

                rv = module->merge_loc_conf(&conf, prev, cur);
                if (rv != NGX_CONF_OK) {
                    goto failed;
                }
            }

            http_ctx.loc_conf[ngx_modules[i]->ctx_index] = cur;
        }
    }

    // temp_pool 内存池使用完了，一并做内存回收。
    ngx_destroy_pool(conf.temp_pool);
    conf.temp_pool = NULL;

    // 创建了一条假 connection 实体，这个其实挺有用的 todo 透彻了解
    c = ngx_http_lua_create_fake_connection(NULL);
    if (c == NULL) {
        goto failed;
    }

    // 错误处理回调设置
    c->log->handler = ngx_http_lua_log_init_worker_error;

    // 创建了一个假 request 实体，这个其实挺有用的 todo 透彻了解
    r = ngx_http_lua_create_fake_request(c);
    if (r == NULL) {
        goto failed;
    }

    // 使用上了之前复制出来的 fake_cycle 的内容
    r->main_conf = http_ctx.main_conf;
    r->srv_conf = http_ctx.srv_conf;
    r->loc_conf = http_ctx.loc_conf;

    clcf = ngx_http_get_module_loc_conf(r, ngx_http_core_module);

    c->log->file = clcf->error_log->file;

    if (!(c->log->log_level & NGX_LOG_DEBUG_CONNECTION)) {
        c->log->log_level = clcf->error_log->log_level;
    }

#endif

    if (top_clcf->resolver) {
        clcf->resolver = top_clcf->resolver;
    }

    ctx = ngx_http_lua_create_ctx(r);
    if (ctx == NULL) {
        goto failed;
    }

    // 设置 上下文为 init worker，关闭了读取事件处理 
    ctx->context = NGX_HTTP_LUA_CONTEXT_INIT_WORKER;
    ctx->cur_co_ctx = NULL;
    r->read_event_handler = ngx_http_block_reading;

    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);
    if (top_llcf->log_socket_errors != NGX_CONF_UNSET) {
        llcf->log_socket_errors = top_llcf->log_socket_errors;
    }

    // 为 Lua 虚拟机塞了 request 结构体进去，方便 Lua 代码使用。
    ngx_http_lua_set_req(lmcf->lua, r);

    // 这里是运行 init_worker_handler 回调，返回值丢弃了
    (void) lmcf->init_worker_handler(cycle->log, lmcf, lmcf->lua);

    ngx_destroy_pool(c->pool);
    return NGX_OK;

failed:

    // 失败收尾处理
    if (conf.temp_pool) {
        ngx_destroy_pool(conf.temp_pool);
    }

    if (c) {
        ngx_http_lua_close_fake_connection(c);
    }

    return NGX_ERROR;
}
```

函数中，复制 cycle 到 fake_cycle 的代码添加是为了解决某些bug：

    /* we fake a temporary ngx_cycle_t here because some
     * modules' merge conf handler may produce side effects in
     * cf->cycle (like ngx_proxy vs cf->cycle->paths).
     * also, we cannot allocate our temp cycle on the stack
     * because some modules like ngx_http_core_module reference
     * addresses within cf->cycle (i.e., via "&cf->cycle->new_log")
     */

在这里可以看到关键点：
*  复制 cycle 到 fake_cycle
*  创建了假 request 和 假 connection 实体，并往 request 上下文设置了 NGX_HTTP_LUA_CONTEXT_INIT_WORKER 并阻塞向 request 读取内容的操作，然后把这个假 request 丢进 Lua 虚拟机中作为 __ngx_req 的全局变量方便 Lua 代码调用。
*  执行 init_worker_handler 回调，也就是调用 ngx_http_lua_init_worker_by_lua 指定的 ngx_http_lua_init_worker_by_inline 函数。

走到这里，下面就是要执行 ngx_http_lua_init_by_inline 以及 ngx_http_lua_init_worker_by_inline 了。

ngx_http_lua_init_by_inline 代码没几行，都与 Lua 执行有关：

```c
ngx_int_t
ngx_http_lua_init_by_inline(ngx_log_t *log, ngx_http_lua_main_conf_t *lmcf,
    lua_State *L)
{
    int         status;

    status = luaL_loadbuffer(L, (char *) lmcf->init_src.data,
                             lmcf->init_src.len, "=init_by_lua")
             || ngx_http_lua_do_call(log, L);

    return ngx_http_lua_report(log, L, status, "init_by_lua");
}

int
ngx_http_lua_do_call(ngx_log_t *log, lua_State *L)
{
    int                 status, base;

    base = lua_gettop(L);  /* function index */
    lua_pushcfunction(L, ngx_http_lua_traceback);  /* push traceback function */
    lua_insert(L, base);  /* put it under chunk and args */

    status = lua_pcall(L, 0, 0, base);

    lua_remove(L, base);

    return status;
}

ngx_int_t
ngx_http_lua_report(ngx_log_t *log, lua_State *L, int status,
    const char *prefix)
{
    const char      *msg;

    if (status && !lua_isnil(L, -1)) {
        msg = lua_tostring(L, -1);
        if (msg == NULL) {
            msg = "unknown error";
        }

        ngx_log_error(NGX_LOG_ERR, log, 0, "%s error: %s", prefix, msg);
        lua_pop(L, 1);
    }

    /* force a full garbage-collection cycle */
    lua_gc(L, LUA_GCCOLLECT, 0);

    return status == 0 ? NGX_OK : NGX_ERROR;
}

int
ngx_http_lua_traceback(lua_State *L)
{
    if (!lua_isstring(L, 1)) { /* 'message' not a string? */
        return 1;  /* keep it intact */
    }

    lua_getglobal(L, "debug");
    if (!lua_istable(L, -1)) {
        lua_pop(L, 1);
        return 1;
    }

    lua_getfield(L, -1, "traceback");
    if (!lua_isfunction(L, -1)) {
        lua_pop(L, 2);
        return 1;
    }

    lua_pushvalue(L, 1);  /* pass error message */
    lua_pushinteger(L, 2);  /* skip this function and traceback */
    lua_call(L, 2, 1);  /* call debug.traceback */
    return 1;
}


```
我们把相关的函数代码都编排在一块看， ngx_http_lua_init_by_inline 函数按步骤执行了下面的功能：
*  调用 luaL_loadbuffer 把传入的 Lua 代码字符串解析成代码块，并作为一个函数压栈。
*  调用 ngx_http_lua_do_call 函数把 ngx_http_lua_traceback 函数压栈，并把 ngx_http_lua_traceback 压到刚使用 luaL_loadbuffer 压入的函数之下。
*  调用 lua_pcall 执行 luaL_loadbuffer 压入的函数，错误处理函数被指向 ngx_http_lua_traceback。
*  调用 lua_remove 从栈中删除 ngx_http_lua_traceback 函数。
*  调用 ngx_http_lua_report 根据 status 状态码，把错误信息以 NGX_LOG_ERR 日志等级写入 error 日志，并进行资源回收。

由于 ngx_http_lua_init_worker 中调用 ngx_http_lua_init_by_inline 是不处理返回值的，所以不管 Lua 代码调用成功还是失败，nginx 程序都将继续往执行。
剩下的其他函数如：ngx_http_lua_init_by_file ， ngx_http_lua_init_worker_by_inline ， ngx_http_lua_init_worker_by_file 都是类似的代码结构，这里就不一一展开了。

