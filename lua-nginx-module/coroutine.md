#coroutine and thread

##coroutine apis

在 ngx apis 章节中，我们了解到本章的主角 coroutine 的接口定义的初步信息，其中 ngx_http_lua_inject_coroutine_api 函数用于定义 coroutine apis，而 coroutine apis 包括了如下接口：
*  create
*  resume
*  yield
*  wrap
*  running
*  status

关于 ngx_http_lua_inject_coroutine_api 我们摘录如下代码：

```c
void
ngx_http_lua_inject_coroutine_api(ngx_log_t *log, lua_State *L)
{
    int         rc;

    // 创建 新的 coroutine table
    lua_createtable(L, 0 /* narr */, 14 /* nrec */);

    //获取旧的 coroutine table
    lua_getglobal(L, "coroutine");

    // 把旧的 running 设置到新的里边
    lua_getfield(L, -1, "running");
    lua_setfield(L, -3, "running");

    // 把旧的 create 设置到新的 _create 里边 
    lua_getfield(L, -1, "create");
    lua_setfield(L, -3, "_create");
    
    // 把旧的 resume 设置到新的 _resume 里边 
    lua_getfield(L, -1, "resume");
    lua_setfield(L, -3, "_resume");

    // 把旧的 yield 设置到新的 _yield 里边 
    lua_getfield(L, -1, "yield");
    lua_setfield(L, -3, "_yield");

    // 把旧的 status 设置到新的 _status 里边 
    lua_getfield(L, -1, "status");
    lua_setfield(L, -3, "_status");

    // 把旧的 coroutine 出栈
    lua_pop(L, 1);

    // 设置 __create 函数
    lua_pushcfunction(L, ngx_http_lua_coroutine_create);
    lua_setfield(L, -2, "__create");

    // 设置 __resume 函数
    lua_pushcfunction(L, ngx_http_lua_coroutine_resume);
    lua_setfield(L, -2, "__resume");

    // 设置 __yield 函数
    lua_pushcfunction(L, ngx_http_lua_coroutine_yield);
    lua_setfield(L, -2, "__yield");

    // 设置 __status 函数
    lua_pushcfunction(L, ngx_http_lua_coroutine_status);
    lua_setfield(L, -2, "__status");

    // 把新的 table 设置为全局的 coroutine table，替代就的 coroutine table
    lua_setglobal(L, "coroutine");

    /* inject coroutine APIs */
    {
        const char buf[] =
            "local keys = {'create', 'yield', 'resume', 'status'}\n"
            "local getfenv = getfenv\n"
            "for _, key in ipairs(keys) do\n"
               "local std = coroutine['_' .. key]\n"
               "local ours = coroutine['__' .. key]\n"
               "local raw_ctx = ngx._phase_ctx\n"
               "coroutine[key] = function (...)\n"
                    "local r = getfenv(0).__ngx_req\n"
                    "if r then\n"
                        "local ctx = raw_ctx(r)\n"
                        /* ignore header and body filters */
                        "if ctx ~= 0x020 and ctx ~= 0x040 then\n"
                            "return ours(...)\n"
                        "end\n"
                    "end\n"
                    "return std(...)\n"
                "end\n"
            "end\n"
            "local create, resume = coroutine.create, coroutine.resume\n"
            "coroutine.wrap = function(f)\n"
               "local co = create(f)\n"
               "return function(...) return select(2, resume(co, ...)) end\n"
            "end\n"
            "package.loaded.coroutine = coroutine";

#if 0
            "debug.sethook(function () collectgarbage() end, 'rl', 1)"
#endif
            ;
        // 把 Lua 代码编入缓存
        rc = luaL_loadbuffer(L, buf, sizeof(buf) - 1, "=coroutine.wrap");
    }

    // 执行上述的 Lua 代码，在不同的 上下文 中执行新旧两套不同的函数
    rc = lua_pcall(L, 0, 0, 0);
}

```

从这段代码中可以了解到如下信息：
*  原本 Lua 虚拟机环境中本来已存在一套 coroutine 接口了。
*  函数中把 coroutine 扩充了一套新的函数实现，包括 __create, __yield, __resume, __status 四个函数，原本的四个函数都改名为 _create, _yield, _resume, _status
*  最后的一段 Lua 代码分成两段内容：
    * 第一段内容，在 coroutine table 添加 create ， yield ， resume ， status 四个成员函数。
    * 这四个成员函数均的实现都是先判断当前上下文 local ctx = raw_ctx(r)
    * 根据上下文是否是 header filter 或者 body filter 中，决定最终调用 \_create 还是 \_\_create 等，从上面 ngx_http_lua_inject_coroutine_api 代码可以得知在这里 \_\* 函数就代表了 coroutine 原本的 apis ，而 \_\_\* 函数则代表了新加入的 coroutine apis 。从上面代码可以得知，在 header 和 body filter 上下文情况下，使用标准库 coroutine apis，而在其他上下文情况下，则使用 ngx lua module 新加入的 coroutine apis
    * 第二段内容，则是对 wrap 函数的重新定义，把 create 和 resume 的上述变化，都引入到 wrap 中，也就是根据不同上下文执行不同的 apis。
    * 最后一句 package.loaded.coroutine = coroutine 的意义在于，用刚定义的 coroutine 重新覆盖已加载的 coroutine 模块，使得 coroutine 不会被后面调用的 require 所覆盖。
    
到这里，就完成了 coroutine apis 加载过程，如上面代码的意思，coroutine apis 的 create , yield , resume , status 这四个函数被重新定义了，在非 body 和 header filter 情况下，会使用新的实现，那么为什么要做这样的重新定义呢？

##coroutine.create

我们先来看看 create 函数所指向的 C 函数 ngx_http_lua_coroutine_create ：

```c
static int
ngx_http_lua_coroutine_create(lua_State *L)
{
    ngx_http_request_t          *r;
    ngx_http_lua_ctx_t          *ctx;

    r = ngx_http_lua_get_req(L);
    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);

    return ngx_http_lua_coroutine_create_helper(L, r, ctx, NULL);
}

int
ngx_http_lua_coroutine_create_helper(lua_State *L, ngx_http_request_t *r,
    ngx_http_lua_ctx_t *ctx, ngx_http_lua_co_ctx_t **pcoctx)
{
    lua_State                     *vm;  /* the Lua VM */
    lua_State                     *co;  /* new coroutine to be created */
    ngx_http_lua_co_ctx_t         *coctx; /* co ctx for the new coroutine */

    // 参数检查，参数1 必须是函数
    luaL_argcheck(L, lua_isfunction(L, 1) && !lua_iscfunction(L, 1), 1,
                 "Lua function expected");

    // 上下文检查，必须在 rewrite access content 或者 timer 上下文中才能使用该函数，否则退出。
    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT
                               | NGX_HTTP_LUA_CONTEXT_TIMER);
    // 这是主线程 vm
    vm = ngx_http_lua_get_lua_vm(r, ctx);

    // 在 vm 基础上创建新的线程 co，也就是一个新的栈 co
    co = lua_newthread(vm);
    
    // 这里对线程上下文做一些状态信息收集
    coctx = ngx_http_lua_get_co_ctx(co, ctx);
    if (coctx == NULL) {
        // 找不到现有的，就新建一个上下文信息
        coctx = ngx_http_lua_create_co_ctx(r, ctx);
        if (coctx == NULL) {
            return luaL_error(L, "no memory");
        }

    } else {
        // 重新清理本线程的上下文信息，毕竟这是全新的线程啊
        ngx_memzero(coctx, sizeof(ngx_http_lua_co_ctx_t));
        coctx->co_ref = LUA_NOREF;
    }

    coctx->co = co;
    coctx->co_status = NGX_HTTP_LUA_CO_SUSPENDED;

    // 新线程 co 与原线程 L 共享 globals table 数据
    ngx_http_lua_get_globals_table(L);
    lua_xmove(L, co, 1);
    ngx_http_lua_set_globals_table(co);

    // 把 co 从主线程 vm 移动到工作线程 L 的栈上
    lua_xmove(vm, L, 1); 

    // 把 存放在 工作线程 L 上的线程入口函数 复制到 co
    lua_pushvalue(L, 1);    /* copy entry function to top of L*/
    lua_xmove(L, co, 1);    /* move entry function from L to co */

    // 把 线程上下文信息作为输出参数输出到函数外
    if (pcoctx) {
        *pcoctx = coctx;
    }

#ifdef NGX_LUA_USE_ASSERT
    coctx->co_top = 1;
#endif

    return 1;    /* return new coroutine to Lua */
}

```

我们把旧的 coroutine.create 的 C 实现列出来做个对比：

```c
LJLIB_CF(coroutine_create)
{
  lua_State *L1;
  
  // 参数检查，参数1 必须是函数
  if (!(L->base < L->top && tvisfunc(L->base)))
    lj_err_argt(L, 1, LUA_TFUNCTION);
  
  // 在当前线程 L 基础上创建新的线程 L1
  L1 = lua_newthread(L);
  
  // 把 L 栈顶的线程入口函数复制到 L1 的栈顶
  setfuncV(L, L1->top++, funcV(L->base));
  return 1;
}

```
这个函数实现是在 LuaJit 源代码中找到，其实现的功能比 ngx lua module 新提供的 ngx_http_lua_coroutine_create 要简单得多，新函数有如下新增内容：
*  函数创建的时候不是基于当前工作线程去创建线程，而是基于主线程 vm 去创建线程
*  与 工作线程 L 共享 globals table 了，而不是主线程 vm
*  对请求过程的上下文设置了线程上下文的内容，以便实现各种功能。


对于线程上下文的相关函数有如下：

```c

ngx_http_lua_co_ctx_t *
ngx_http_lua_get_co_ctx(lua_State *L, ngx_http_lua_ctx_t *ctx)
{
    ngx_uint_t                   i;
    ngx_list_part_t             *part;
    ngx_http_lua_co_ctx_t       *coctx;

    // 如果是 请求的入口线程 则返回 ctx->entry_co_ctx
    if (L == ctx->entry_co_ctx.co) {
        return &ctx->entry_co_ctx;
    }

    // 除了入口线程，还有一个工作线程 list 链表记录着这个请求说启动的所有线程上下文信息
    part = &ctx->user_co_ctx->part;
    coctx = part->elts;

    // 当然这里做枚举了，作者在这里加了注释说有必要的话可以改成用 rbtree 提供 O(log(n))性能，对于轻度线程使用者来说，这不是问题
    for (i = 0; /* void */; i++) {

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }

            part = part->next;
            coctx = part->elts;
            i = 0;
        }

        // 找到了 L 的上下文信息结构体了，返回它
        if (coctx[i].co == L) {
            return &coctx[i];
        }
    }

    // 找不到返回 NULL
    return NULL;
}


ngx_http_lua_co_ctx_t *
ngx_http_lua_create_co_ctx(ngx_http_request_t *r, ngx_http_lua_ctx_t *ctx)
{
    ngx_http_lua_co_ctx_t       *coctx;

    // 在第一次，需要把线程 list 给创建起来
    if (ctx->user_co_ctx == NULL) {
        ctx->user_co_ctx = ngx_list_create(r->pool, 4, sizeof(ngx_http_lua_co_ctx_t));

    }

    // 在线程 list 中添加一个上下文信息结构体
    coctx = ngx_list_push(ctx->user_co_ctx);
    if (coctx == NULL) {
        return NULL;
    }

    // 初始化结构体
    ngx_memzero(coctx, sizeof(ngx_http_lua_co_ctx_t));
    coctx->co_ref = LUA_NOREF;

    // 作为返回值返回
    return coctx;
}

```

这个地方牵涉到 ngx_http_lua_ctx_t ngx lua module 请求上下文结构体的两个成员变量：
```c

typedef struct ngx_http_lua_ctx_s {
    ...
    ngx_http_lua_co_ctx_t   *cur_co_ctx; /* 当前线程上下文信息 */
    ngx_list_t              *user_co_ctx;  /* 用户创建的线程上下文信息列表 */
    ngx_http_lua_co_ctx_t    entry_co_ctx; /* 请求入口线程上下文信息 */
    unsigned         co_op:2; /*  coroutine API 操作 */
    ...
} ngx_http_lua_ctx_t;
```

entry_co_ctx 是在 phase handler 创建用于执行 Lua 代码的工作线程之后，用于保存这个工作线程信息用的。在 access 、 content 、 rewrite 等阶段都有这样的代码存在。

##coroutine.yield

coroutine.yield 函数在官方网站上的功能描述是，暂停当前正在运行的线程。在 ngx lua module 中，他有一个 C 函数实现：

```c
static int
ngx_http_lua_coroutine_yield(lua_State *L)
{
    ngx_http_request_t          *r;
    ngx_http_lua_ctx_t          *ctx;
    ngx_http_lua_co_ctx_t       *coctx;

    // 获取上下文信息
    r = ngx_http_lua_get_req(L);
    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);

    // 上下文检查，在 rewrite access content timer 中允许运行，否则退出
    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT
                               | NGX_HTTP_LUA_CONTEXT_TIMER);

    coctx = ctx->cur_co_ctx;
    coctx->co_status = NGX_HTTP_LUA_CO_SUSPENDED;

    ctx->co_op = NGX_HTTP_LUA_USER_CORO_YIELD;

    if (!coctx->is_uthread && coctx->parent_co_ctx) {
        // 如果有母线程，把母线程状态修改成正在运行
        coctx->parent_co_ctx->co_status = NGX_HTTP_LUA_CO_RUNNING;
    } else {
        ngx_http_lua_probe_user_coroutine_yield(r, NULL, L);
    }

    //暂停当前线程，如果有母线程会让母线程继续跑下去
    return lua_yield(L, lua_gettop(L));
}

```
这里 lua_yield 执行之前，添加了一些额外的 上下文检查以及母线程状态切换的逻辑。

## coroutine.resume

我们再来看看 coroutine.resume 的 C 函数实现 ngx_http_lua_coroutine_resume：

```c

static int
ngx_http_lua_coroutine_resume(lua_State *L)
{
    lua_State                   *co;
    ngx_http_request_t          *r;
    ngx_http_lua_ctx_t          *ctx;
    ngx_http_lua_co_ctx_t       *coctx;
    ngx_http_lua_co_ctx_t       *p_coctx; /* parent co ctx */

    // 从栈顶(参数1)获得要执行的线程 co
    co = lua_tothread(L, 1);
    r = ngx_http_lua_get_req(L);
    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);

    // 上下文检查
    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT
                               | NGX_HTTP_LUA_CONTEXT_TIMER);

    // 把当前线程的上下文设为母线程上下文
    p_coctx = ctx->cur_co_ctx;
    p_coctx->co_status = NGX_HTTP_LUA_CO_NORMAL;

    // 设置要执行的线程的母线程上下文，下面代码与执行 resume 有关。
    coctx = ngx_http_lua_get_co_ctx(co, ctx);
    coctx->parent_co_ctx = p_coctx;
    coctx->co_status = NGX_HTTP_LUA_CO_RUNNING;
    ctx->co_op = NGX_HTTP_LUA_USER_CORO_RESUME;

    // 把要执行的线程设置为当前线程
    ctx->cur_co_ctx = coctx;

    // 这里手法挺特别，把当前线程放弃掉，然后回归到更上一层的线程，让它执行 resume 命令。
    return lua_yield(L, lua_gettop(L) - 1);
}

```

coroutine.resume 的 C 实现有好多特别的地方：
*  它并没有按照正常流程调用 lua_resume
*  它调用 lua_yield 把当前线程放弃掉，然后回归到更上一层的线程，让它执行 resume 命令
*  为了让上一层线程正确执行要执行的线程， 它先把当前线程以及 co_op 等上下文都设置了一遍。
*  如果每个线程都是如此先让出执行 lua_yield 再由 上一层线程执行 lua_resume 的话，那么应该所有的 lua_resume 都在同一条线程上执行。
*  这个上一层线程实际上执行的是 ngx_http_lua_run_thread 函数，它并非由某条线程运行而是 C 宿主的调用。

##  ngx_http_lua_run_thread

ngx_http_lua_run_thread 这个函数有一个相当大的循环：

```c
ngx_int_t
ngx_http_lua_run_thread(lua_State *L, ngx_http_request_t *r,
    ngx_http_lua_ctx_t *ctx, volatile int nrets)
{
    ngx_http_lua_co_ctx_t   *next_coctx, *parent_coctx, *orig_coctx;

    lua_atpanic(L, ngx_http_lua_atpanic);

    NGX_LUA_EXCEPTION_TRY {

        ...

        for ( ;; ) {

            orig_coctx = ctx->cur_co_ctx;
            rv = lua_resume(orig_coctx->co, nrets);

            switch (rv) {
            case LUA_YIELD:

                if (r->uri_changed) {
                    return ngx_http_lua_handle_rewrite_jump(L, r, ctx);
                }

                if (ctx->exited) {
                    return ngx_http_lua_handle_exit(L, r, ctx);
                }

                if (ctx->exec_uri.len) {
                    return ngx_http_lua_handle_exec(L, r, ctx);
                }

                // 判断是否 coroutine.resume 或者 coroutine.yield 被调用的情况
                switch(ctx->co_op) {
                case NGX_HTTP_LUA_USER_CORO_NOP:
                    // 不再有线程需要处理了，跳出这一次循环，重新等待下一次读写事件。
                    ctx->cur_co_ctx = NULL;
                    return NGX_AGAIN;

                case NGX_HTTP_LUA_USER_THREAD_RESUME:
                    ctx->co_op = NGX_HTTP_LUA_USER_CORO_NOP;
                    nrets = lua_gettop(ctx->cur_co_ctx->co) - 1;
                    break;

                case NGX_HTTP_LUA_USER_CORO_RESUME:
                    // ctx->co_op == NGX_HTTP_LUA_USER_CORO_RESUME
                    // 这里是 coroutine.resume 被调用的情况
                    ctx->co_op = NGX_HTTP_LUA_USER_CORO_NOP;
                    old_co = ctx->cur_co_ctx->parent_co_ctx->co;
                    nrets = lua_gettop(old_co);
                    if (nrets) {
                        lua_xmove(old_co, ctx->cur_co_ctx->co, nrets);
                    }
                    break;

                default:
                    /* ctx->co_op == NGX_HTTP_LUA_USER_CORO_YIELD */
                    // 这里是 coroutine.yield 被调用的情况
                    ctx->co_op = NGX_HTTP_LUA_USER_CORO_NOP;

                    if (ngx_http_lua_is_thread(ctx)) {
                        ngx_http_lua_probe_thread_yield(r, ctx->cur_co_ctx->co);

                        // coroutine.yield() 的调用参数都被干掉了，不会被传到 resume 
                        lua_settop(ctx->cur_co_ctx->co, 0);
                        ctx->cur_co_ctx->co_status = NGX_HTTP_LUA_CO_RUNNING;

                        // 判断有没有需要后处理的线程，排队等待处理
                        if (ctx->posted_threads) {
                            ngx_http_lua_post_thread(r, ctx, ctx->cur_co_ctx);
                            ctx->cur_co_ctx = NULL;
                            return NGX_AGAIN;
                        }

                        // 如果木有，直接让自己恢复执行即可，直接回到 for 循环开头
                        nrets = 0;
                        continue;
                    }

                    // 下面的代码是 切换当前线程 到 母线程 的逻辑
                    nrets = lua_gettop(ctx->cur_co_ctx->co);
                    next_coctx = ctx->cur_co_ctx->parent_co_ctx;
                    next_co = next_coctx->co;

                    // 在 coroutine.resume 调用者线程的栈上插入一个返回值 true
                    lua_pushboolean(next_co, 1);

                    // 把 yield 的参数都作为返回值发往母线程的栈中
                    if (nrets) {
                        lua_xmove(ctx->cur_co_ctx->co, next_co, nrets);
                    }

                    // 加上第一个返回值 true
                    nrets++;
                    ctx->cur_co_ctx = next_coctx;
                    break;
                }

                // 回到 for 循环开头，执行 resume
                continue;

            case ...:
                ...
        }

    } NGX_LUA_EXCEPTION_CATCH {
        ...
    }

    return NGX_OK;
}

```

在这里我们没有把 ngx_http_lua_run_thread 的所有代码都做展开，只是把与 coroutine.resume 有关的代码进行分析，从代码上可以分析出：
*  ngx_http_lua_run_thread 是 nginx 宿主执行Lua线程的源头
*  在 ctx->co_op = NGX_HTTP_LUA_USER_CORO_RESUME; 的情况下，它会把当前线程去执行 resume ，这对应上 ngx_http_lua_coroutine_resume 的程序逻辑。
*  在 ctx->co_op = NGX_HTTP_LUA_USER_CORO_YIELD; 的情况下，它把当前线程的母线程执行 resume ，这对应上 ngx_http_lua_coroutine_yield 的程序逻辑。
*  在所有分支逻辑里边最终会把 ctx->co_op 设置成 NGX_HTTP_LUA_USER_CORO_NOP
*  在 ctx->co_op = NGX_HTTP_LUA_USER_CORO_NOP，也就是再也没有需要执行的线程的情况下，会跳出函数，跳出这一次 Lua 代码执行，回到请求处理流程中

事实上， ngx_http_lua_run_thread 函数其实就是 access 、 content 、 rewrite 、 timer 等阶段的 Lua 代码启动函数，他里边的逻辑就包含了对线程的 yield 和 resume 的处理。

## coroutine.status

这个函数功能相对简单，就是把线程的状态返回给 Lua 调用者，他的实现由 C 函数 ngx_http_lua_coroutine_status 提供

```c
static int
ngx_http_lua_coroutine_status(lua_State *L)
{
    co = lua_tothread(L, 1);
    r = ngx_http_lua_get_req(L);
    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);

    // 上下文检查
    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT
                               | NGX_HTTP_LUA_CONTEXT_TIMER);

    // 获得上下文
    coctx = ngx_http_lua_get_co_ctx(co, ctx);

    //　把状态值转换为字符串处理，作为返回值入栈
    lua_pushlstring(L, (const char *)
                    ngx_http_lua_co_status_names[coctx->co_status].data,
                    ngx_http_lua_co_status_names[coctx->co_status].len);
    return 1;
}
```

## 总结
其实就是简单的使用 lua_yield 以及 lua_resume 的使用，外加一些线程上下文的状态保存逻辑。
