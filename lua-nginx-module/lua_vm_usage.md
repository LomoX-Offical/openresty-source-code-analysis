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
    // 从 registry 全局注册表，找 ngx_http_lua_code_cache_key 的地址作为索引的值，并且把值入栈
    lua_pushlightuserdata(L, &ngx_http_lua_code_cache_key);
    lua_rawget(L, LUA_REGISTRYINDEX);   /*  sp++ */

    // 这个值其实是一个 table ， 用于存储 Lua 代码块数据，现在以 key 为索引对 table 进行查找并把结果入栈
    lua_getfield(L, -1, key);    /*  sp++ */

    if (lua_isfunction(L, -1)) {
        /*  call closure factory to gen new closure */
        rc = lua_pcall(L, 0, 1, 0);
        if (rc == 0) {
            /*  remove cache table from stack, leave code chunk at
             *  top of stack */
            lua_remove(L, -2);   /*  sp-- */
            return NGX_OK;
        }

        if (lua_isstring(L, -1)) {
            err = (u_char *) lua_tostring(L, -1);

        } else {
            err = (u_char *) "unknown error";
        }

        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "lua: failed to run factory at key \"%s\": %s",
                      key, err);
        lua_pop(L, 2);
        return NGX_ERROR;
    }

    /*  remove cache table and value from stack */
    lua_pop(L, 2);                                /*  sp-=2 */

    return NGX_DECLINED;
}


```
这个是一个获取保存在 Lua 虚拟机的 C 函数使用全局变量的方法，key 是 ngx_http_lua_code_cache_key 的地址（独一无二）


然而其实我们还不知道 ngx_http_lua_body_filter_by_chunk 里边究竟发生了什么：

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

        /*  error occured */
        err_msg = (u_char *) lua_tolstring(L, -1, &len);

        if (err_msg == NULL) {
            err_msg = (u_char *) "unknown reason";
            len = sizeof("unknown reason") - 1;
        }

        ngx_log_error(NGX_LOG_ERR, r->connection->log, 0,
                      "failed to run body_filter_by_lua*: %*s", len, err_msg);

        lua_settop(L, 0);    /*  clear remaining elems on stack */

        return NGX_ERROR;
    }

    rc = (ngx_int_t) lua_tointeger(L, -1);
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


