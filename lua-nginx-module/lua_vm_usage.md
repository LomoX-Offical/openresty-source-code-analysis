# Lua VM Usage

## 概述
在植入 Lua 代码的指令里，为了实现其不同的功能，都会在运行 Lua 代码前，向 Lua 虚拟机塞进满足其功能使用的数据结构，以便 Lua 代码能够掌握足够的信息处理应对其业务逻辑。

在最初的章节，我们了解了 init_by_lua 的 Lua 调用细节， 也就是函数 ngx_http_lua_init_by_inline 的代码解析，初步了解了：
*  Lua 虚拟机的创建
*  C 函数入栈 Lua 虚拟机
*  Lua 代码块的加载
*  错误处理回调以及使用 pcall 安全地执行 Lua 代码块

而这些都应该是我们深入了解 lua-nginx-module 如何 Lua 虚拟机使用的基础知识，下面我们再次深入到 Lua 虚拟机使用之中。

## body_filter_by_lua

为了理解 body_filter_by_lua 指令的作用，我们来看个 Openresty 的官方 wiki 上的例子：

```nginx
    location /t {
        echo hello world;
        echo hiya globe;
 
        body_filter_by_lua '
            local chunk = ngx.arg[1]
            if string.match(chunk, "hello") then
                ngx.arg[2] = true  -- new eof
                return
            end
 
            -- just throw away any remaining chunk data
            ngx.arg[1] = nil
        ';
    }
```

代码比较好理解，但有两个变量搞不懂，分别是 ngx.arg[1] 和 ngx.arg[2]。 Openresty 的官方解释是：

The input data chunk is passed via ngx.arg[1] (as a Lua string value) and the "eof" flag indicating the end of the response body data stream is passed via ngx.arg[2] (as a Lua boolean value).

意思就是：通过 ngx.arg[1] （作为一个 Lua 字符串类型）传入 input 数据块，而 ngx.arg[2] （作为一个布尔类型变量）表示是否已经到了应答数据流的末尾（eof）。
在上面代码例子中，我们可以看到，除了在代码执行之前作为参数传入之外，ngx.arg[1] 和 ngx.arg[2] 其实还可以被修改。

最终 /t 的 body 输出结果变成了：

```
  hello world
```

那么我们回到 ngx_http_lua_body_filter_inline 这个关键的 body filter 回调函数中：

```c

ngx_int_t
ngx_http_lua_body_filter_inline(ngx_http_request_t *r, ngx_chain_t *in)
{
    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);

    L = ngx_http_lua_get_lua_vm(r, NULL);
    rc = ngx_http_lua_cache_loadbuffer(r, L, llcf->body_filter_src.value.data,
                                       llcf->body_filter_src.value.len,
                                       llcf->body_filter_src_key,
                                       "=body_filter_by_lua");
    rc = ngx_http_lua_body_filter_by_chunk(L, r, in);

    if (rc != NGX_OK) {
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

现在看来代码结构是多么的熟悉，先把 Lua 虚拟机从 location 配置中读出来，再根据 cache_key 寻找对应的代码块缓存并入栈 Lua 虚拟机，最后通过 ngx_http_lua_body_filter_by_chunk 执行 Lua 代码。

## ngx_http_lua_cache_loadbuffer
然而，我们其实还没了解 ngx_http_lua_cache_loadbuffer 是怎么实现的：

```c
ngx_int_t
ngx_http_lua_cache_loadbuffer(ngx_http_request_t *r, lua_State *L,
    const u_char *src, size_t src_len, const u_char *cache_key,
    const char *name)
{
    n = lua_gettop(L);

    rc = ngx_http_lua_cache_load_code(r, L, (char *) cache_key);
    if (rc == NGX_OK) {
        // 代码块缓存找到并压栈了           
        return NGX_OK;
    }

    // 从字符串 src 生成 Lua 代码块
    rc = ngx_http_lua_clfactory_loadbuffer(L, (char *) src, src_len, name);
    
    // 把 Lua 代码块以 cache_key 作为索引保存起来。
    rc = ngx_http_lua_cache_store_code(L, (char *) cache_key);
    
    return NGX_OK;
}

static ngx_int_t
ngx_http_lua_cache_load_code(ngx_http_request_t *r, lua_State *L,
    const char *key)
{
    // 从 registry 全局注册表，找 ngx_http_lua_code_cache_key 的地址作为索引的值，并且把值 cache_table 入栈 
    lua_pushlightuserdata(L, &ngx_http_lua_code_cache_key);
    lua_rawget(L, LUA_REGISTRYINDEX);   /*  sp++ */

    // 这个 cache_table 用于存储 Lua 代码块数据，现在以 key 为索引对 cache_table 进行查找并把结果入栈
    lua_getfield(L, -1, key);    /*  sp++ */

    // 判断栈顶的代码块是否是一个函数
    if (lua_isfunction(L, -1)) {
        // 执行栈顶的 Lua 代码块
        rc = lua_pcall(L, 0, 1, 0);
        if (rc == 0) {
            // 把 cache_table 删除掉
            lua_remove(L, -2);   /*  sp-- */
            return NGX_OK;
        }

        // 运行错误的话，把 cache_table 和代码块删除掉
        lua_pop(L, 2);
        return NGX_ERROR;
    }

    // 代码块不是一个函数的话，把 cache_table 和代码块删除掉
    lua_pop(L, 2);

    return NGX_DECLINED;
}

static ngx_int_t
ngx_http_lua_cache_store_code(lua_State *L, const char *key)
{
    int rc;

    // 从 registry 全局注册表，找 ngx_http_lua_code_cache_key 的地址作为索引的值，并且把值入栈，是一个 cache_code_table
    lua_pushlightuserdata(L, &ngx_http_lua_code_cache_key);
    lua_rawget(L, LUA_REGISTRYINDEX);

    // 把原本栈顶现在 [-2] 的代码块复制再入栈
    lua_pushvalue(L, -2);
    
    // 把栈顶的代码块以 key 作为索引值，插入到 cache_code_table，并出栈复制出来的代码块
    lua_setfield(L, -2, key); 
    
    // 把 cache_code_table 出栈
    lua_pop(L, 1);

    // 回到最初的 Lua 代码块，并执行它，想想为什么要执行它
    rc = lua_pcall(L, 0, 1, 0);
    if (rc != 0) {
        dd("Error: failed to call closure factory!!");
        return NGX_ERROR;
    }

    return NGX_OK;
}

ngx_int_t
ngx_http_lua_clfactory_loadbuffer(lua_State *L, const char *buff,
    size_t size, const char *name)
{
    ngx_http_lua_clfactory_buffer_ctx_t     ls;

    ls.s = buff;
    ls.size = size;
    ls.sent_begin = 0;
    ls.sent_end = 0;

    // 把 ngx_http_lua_clfactory_getS 作为 reader，把 ls 作为 reader 的参数使用
    return lua_load(L, ngx_http_lua_clfactory_getS, &ls, name);
}


#define CLFACTORY_BEGIN_CODE "return function() "
#define CLFACTORY_BEGIN_SIZE (sizeof(CLFACTORY_BEGIN_CODE) - 1)

#define CLFACTORY_END_CODE "\nend"
#define CLFACTORY_END_SIZE (sizeof(CLFACTORY_END_CODE) - 1)

static const char *
ngx_http_lua_clfactory_getS(lua_State *L, void *ud, size_t *size)
{
    ngx_http_lua_clfactory_buffer_ctx_t      *ls = ud;

    // 其实就相当于状态机使用 状态 1
    if (ls->sent_begin == 0) {
        ls->sent_begin = 1;
        *size = CLFACTORY_BEGIN_SIZE;

        return CLFACTORY_BEGIN_CODE;
    }

    if (ls->size == 0) {
        // 状态 2
        if (ls->sent_end == 0) {
            ls->sent_end = 1;
            *size = CLFACTORY_END_SIZE;
            return CLFACTORY_END_CODE;
        }

        return NULL;
    }

    // 状态 3
    *size = ls->size;
    ls->size = 0;

    return ls->s;
}

```
这里我把流程从 Lua 代码块的角度重新描述一遍：
*  当第一次进入 ngx_http_lua_cache_loadbuffer 时， ngx_http_lua_cache_load_code 会找不到已经加载好的代码。
*  所以进入到了 ngx_http_lua_clfactory_loadbuffer ， 会把代码添加 return function() 和 end 作为结尾，形成一个代码块，这个代码块能够返回一个函数体，
*  然后 ngx_http_lua_cache_store_code 把这个代码块保存在 registry 全局注册表的 cache_table 中，
*  调用 lua_pcall 执行这个 Lua 代码块，结果是返回一个函数并入栈，退出 ngx_http_lua_cache_loadbuffer 之后，Lua 虚拟机栈顶就是一个带有我们配置的 Lua 代码的函数，
*  在 ngx_http_lua_body_filter_by_chunk 函数中执行这个栈顶的函数，
*  当第二次进入 ngx_http_lua_cache_loadbuffer 时，ngx_http_lua_cache_load_code 从 registry 全局注册表的 cache_table 中，找到之前保存好的代码块，
*  判断这个代码块是一个函数的话，调用 lua_pcall 执行它，就如上次结果一样，返回一个函数并入栈，
*  第二次在 ngx_http_lua_body_filter_by_chunk 函数中执行这个栈顶的函数
*  第三次 ...

关于代码缓存，在这里我们总结一下关键点：
*  代码缓存的存储其实是利用 Lua 虚拟机的 registry 全局注册表的 cache_table 做代码块保存，
*  获取保存在 Lua 虚拟机的 C 函数使用全局变量（在这里就是 cache_table ）的关键， 是把 ngx_http_lua_code_cache_key 的地址（独一无二）作为索引，
*  代码缓存的索引其实是之前使用的 Lua 代码字符串 或者 Lua 代码文件的名字，做 hash 而计算得出，
*  代码缓存的代码块结构其实是 Lua 代码字符串前后添加了函数包装的前后缀，最终代码块会被解析成一个能返回函数的代码块，所以代码块入栈后，还得跑一次 lua_pcall 得到运行后的函数体。

## ngx_http_lua_body_filter_by_chunk
我们了解了代码缓存的秘密之后，然而其实我们还不知道 ngx_http_lua_body_filter_by_chunk 里边究竟发生了什么：

```c
ngx_int_t
ngx_http_lua_body_filter_by_chunk(lua_State *L, ngx_http_request_t *r,
    ngx_chain_t *in)
{
    // 初始化 Lua 虚拟机用于 body filter 的执行环境，见函数定义
    ngx_http_lua_body_filter_by_lua_env(L, r, in);

    // 很熟悉了， traceback 入栈用于错误处理
    lua_pushcfunction(L, ngx_http_lua_traceback);

    // 很熟悉了， traceback 函数被往下压， body filter 的代码块被放在栈顶
    lua_insert(L, 1);

    // 执行 Lua 代码块
    rc = lua_pcall(L, 0, 1, 1);

    // 从 Lua 虚拟机删除 traceback 函数 
    lua_remove(L, 1);

    //错误处理
    if (rc != 0) {
        err_msg = (u_char *) lua_tolstring(L, -1, &len);
        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "failed to run body_filter_by_lua*: %*s", len, err_msg);

        // 把栈上所有元素移除
        lua_settop(L, 0); 
        return NGX_ERROR;
    }

    rc = (ngx_int_t) lua_tointeger(L, -1);

    // 把栈上所有元素移除
    lua_settop(L, 0);

    if (rc == NGX_ERROR || rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        return NGX_ERROR;
    }

    return NGX_OK;
}

static void
ngx_http_lua_body_filter_by_lua_env(lua_State *L, ngx_http_request_t *r,
    ngx_chain_t *in)
{
    // 把 request 请求结构体压入 Lua 虚拟机，请看函数定义
    ngx_http_lua_set_req(L, r);

    // 把数据链表结构体作为 ngx_http_lua_chain_key ( 全局变量名）压入 Lua 虚拟机
    lua_pushlightuserdata(L, in);
    lua_setglobal(L, ngx_http_lua_chain_key);

    // 在栈顶创建一个 table 这里标记为@1 ，并在这个 table 创建名为 _G 的字段并赋值新的 table
    ngx_http_lua_create_new_globals_table(L, 0 /* narr */, 1 /* nrec */);

    // 在栈顶创建一个 table 这里标记为@2 ，并把全局变量数组塞进这个 table 的 __index 字段中
    lua_createtable(L, 0, 1 /* nrec */);
    ngx_http_lua_get_globals_table(L);
    lua_setfield(L, -2, "__index");
    
    // 把栈顶的 table 即 @2 出栈，并把 @2 作为 @1 的元表
    lua_setmetatable(L, -2);

    // 把 @1 作为新的环境数据写入 Lua 虚拟机
    lua_setfenv(L, -2);
}

static ngx_inline void
ngx_http_lua_set_req(lua_State *L, ngx_http_request_t *r)
{
    lua_pushlightuserdata(L, r);
    lua_setglobal(L, ngx_http_lua_req_key);
}


void
ngx_http_lua_create_new_globals_table(lua_State *L, int narr, int nrec)
{
    lua_createtable(L, narr, nrec + 1);
    lua_pushvalue(L, -1);
    lua_setfield(L, -2, "_G");
}

static ngx_inline void
ngx_http_lua_get_globals_table(lua_State *L)
{
    lua_pushvalue(L, LUA_GLOBALSINDEX);
}

```

ngx_http_lua_body_filter_by_lua_env 刚开始看得云里雾里，在下也是刚开始接触 Lua 虚拟机编程，对 Lua 虚拟机的出栈入栈比较头大，但看着看着也慢慢可以分析一下代码了，这里也建议读者您千万别放弃这个学习的机会。

好了我们言归正传， ngx_http_lua_body_filter_by_lua_env 看最终要实现的功能就是, 把 request 结构体数据以及输入数据链 chain 作为全局变量入栈，并且把当前 Lua 虚拟机的全局变量数据带入新创建的环境数据中，之所以要创建新的环境数据，大概是因为作者不希望 Lua 代码执行的结果影响到 Openresty 进程全局的环境数据，而是尽可能的把本次的代码执行结果的影响限制在本次代码执行的环境数据中，达到隔离 Lua 代码执行的效果，从而实现一个安全的执行环境。

## ngx_http_lua_access_by_chunk

filter_by_lua\* 系列指令我们就已经做过代码分析，流程也做了透彻了解，下面我们再看看 access_by_lua\* 系列，因为这个系列比 filter_by_lua\* 系列的代码使用更多 Lua 虚拟机特性，代码也更加复杂。

上一节中我们已经对 access_by_lua\* 的基本流程做了解释，剩下 ngx_http_lua_access_by_chunk 没有展开，有了前面我们对 ngx_http_lua_body_filter_inline 和 ngx_http_lua_body_filter_by_chunk 的解读作为基础，阅读 ngx_http_lua_access_by_chunk 函数的代码将会更顺利。 
回顾 ngx_http_lua_access_handler_inline 一下基本流程：
*  在请求到来时，先通过 ngx_http_lua_get_lua_vm 获取到 Lua 虚拟机
*  再通过 ngx_http_lua_cache_loadbuffer 获取代码缓存并入栈，在本节前文中我们已经了解代码缓存的存取了。
*  调用 ngx_http_lua_access_by_chunk 函数执行 access_by_lua* 指令指定的 Lua 代码并根据其结果调整处理流程。

```c
ngx_int_t
ngx_http_lua_access_handler_inline(ngx_http_request_t *r)
{
    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);
    L = ngx_http_lua_get_lua_vm(r, NULL);
    rc = ngx_http_lua_cache_loadbuffer(r, L, llcf->access_src.value.data,
                                       llcf->access_src.value.len,
                                       llcf->access_src_key,
                                       (const char *) llcf->access_chunkname);
    return ngx_http_lua_access_by_chunk(L, r);
}
```

下面我们看看 ngx_http_lua_access_by_chunk 函数的细节：

```c
static ngx_int_t
ngx_http_lua_access_by_chunk(lua_State *L, ngx_http_request_t *r)
{
    int                  co_ref;
    ngx_int_t            rc;
    lua_State           *co;
    ngx_connection_t    *c;
    ngx_http_lua_ctx_t  *ctx;
    ngx_http_cleanup_t  *cln;

    ngx_http_lua_loc_conf_t     *llcf;

    // 为这个请求创建新协程
    co = ngx_http_lua_new_thread(r, L, &co_ref);

    // 把要执行的 Lua 代码块从 Lua 虚拟机 L ，交换到新的协程 Lua 虚拟机 co 上
    lua_xmove(L, co, 1);

    // 把 co 虚拟机的全局变量数据 table 复制入栈
    ngx_http_lua_get_globals_table(co);
    
    // 奇怪了，为毛是 -2 ，把 XXX 设置为 co 虚拟机的执行环境数据 table
    lua_setfenv(co, -2);

    // 很熟悉了吧，把 request 请求结构体入栈 co 虚拟机
    ngx_http_lua_set_req(co, r);

    // 清理 ctx 上下文结构体数据
    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    ngx_http_lua_reset_ctx(r, L, ctx);

    // 配置 ctx 数据
    ctx->entered_access_phase = 1;
    ctx->cur_co_ctx = &ctx->entry_co_ctx;
    ctx->cur_co_ctx->co = co;
    ctx->cur_co_ctx->co_ref = co_ref;
#ifdef NGX_LUA_USE_ASSERT
    ctx->cur_co_ctx->co_top = 1;
#endif
    ctx->context = NGX_HTTP_LUA_CONTEXT_ACCESS;



    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);

    if (llcf->check_client_abort) {
        r->read_event_handler = ngx_http_lua_rd_check_broken_connection;

    } else {
        r->read_event_handler = ngx_http_block_reading;
    }

    // 把 Lua 代码块放在新协程虚拟机上运行
    rc = ngx_http_lua_run_thread(L, r, ctx, 0);

    if (rc == NGX_ERROR || rc > NGX_OK) {
        return rc;
    }

    c = r->connection;

    if (rc == NGX_AGAIN) {
        rc = ngx_http_lua_run_posted_threads(c, L, r, ctx);

        if (rc == NGX_ERROR || rc == NGX_DONE || rc > NGX_OK) {
            return rc;
        }

        if (rc != NGX_OK) {
            return NGX_DECLINED;
        }

    } else if (rc == NGX_DONE) {
        ngx_http_lua_finalize_request(r, NGX_DONE);

        rc = ngx_http_lua_run_posted_threads(c, L, r, ctx);

        if (rc == NGX_ERROR || rc == NGX_DONE || rc > NGX_OK) {
            return rc;
        }

        if (rc != NGX_OK) {
            return NGX_DECLINED;
        }
    }

#if 1
    if (rc == NGX_OK) {
        if (r->header_sent) {
            dd("header already sent");

            /* response header was already generated in access_by_lua*,
             * so it is no longer safe to proceed to later phases
             * which may generate responses again */

            if (!ctx->eof) {
                dd("eof not yet sent");

                rc = ngx_http_lua_send_chain_link(r, ctx, NULL
                                                  /* indicate last_buf */);
                if (rc == NGX_ERROR || rc > NGX_OK) {
                    return rc;
                }
            }

            return NGX_HTTP_OK;
        }

        return NGX_OK;
    }
#endif

    return NGX_DECLINED;
}

lua_State *
ngx_http_lua_new_thread(ngx_http_request_t *r, lua_State *L, int *ref)
{
    int              base;
    lua_State       *co;

    // 记一个 top 的索引
    base = lua_gettop(L);

    // 使用 ngx_http_lua_coroutines_key 的地址作为索引，在 register 全局注册表中找到 corutine_table 入栈
    lua_pushlightuserdata(L, &ngx_http_lua_coroutines_key);
    lua_rawget(L, LUA_REGISTRYINDEX);

    // 用 Lua 虚拟机 L 创建一条新协程，并将其压栈，并返回维护这个线程的 lua_State 赋值给 co
    co = lua_newthread(L);

    // 在 co 虚拟机重新设定全局变量表，并出栈
    ngx_http_lua_create_new_globals_table(co, 0, 0);
    lua_createtable(co, 0, 1);
    ngx_http_lua_get_globals_table(co);
    lua_setfield(co, -2, "__index");
    lua_setmetatable(co, -2);
    ngx_http_lua_set_globals_table(co);

    // 回到 Lua 虚拟机 L ，把 co 这个栈顶协程引用到 corutine_table ，并得到 ref 作为引用索引
    *ref = luaL_ref(L, -2);
    
    // 完成所有操作了，删除 base 之上的数据，恢复栈状态
    lua_settop(L, base);
    return co;
}



```





看来已经差不多了解整个流程了，但是像 request 请求结构体以及 chain 数据链结构体，流程中只是使用 lua_pushlightuserdata 函数把它们作为指针入栈到 Lua 虚拟机而已，在 Lua 代码的执行过程中，这些数据会怎么样使用呢？
