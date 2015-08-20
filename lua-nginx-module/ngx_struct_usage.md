# Nginx Struct Usage

## 概述

在前文中，我们了解到 lua-nginx-module 是如何把 Lua 代码植入到 nginx 的每个请求处理阶段中， Lua 代码块在运行结束后，nignx 根据其返回值进行逻辑处理，决定要读取内容，还是关闭请求，还是让另一个 handler 处理。
然而，我们使用 Lua 编写的算法，需要根据输入参数的来计算出返回值的，那么 Lua 代码需要用到的输入数据有哪些数据呢：
*  request 请求结构体
*  cycle 进程循环结构体
*  chain 数据链结构体


在各个阶段的 Lua 虚拟机执行前的初始化步骤中，都执行过如下的一些函数：

```c
#define ngx_http_lua_req_key  "__ngx_req"

static ngx_inline void
ngx_http_lua_set_req(lua_State *L, ngx_http_request_t *r)
{
	// request 结构体入栈的代码
    lua_pushlightuserdata(L, r);
    lua_setglobal(L, ngx_http_lua_req_key);
}

#define ngx_http_lua_chain_key  "__ngx_cl"

static void
ngx_http_lua_body_filter_by_lua_env(lua_State *L, ngx_http_request_t *r,
    ngx_chain_t *in)
{
	// 省略了很多代码，只剩下 chain 结构体入栈的代码了
    lua_pushlightuserdata(L, in);
    lua_setglobal(L, ngx_http_lua_chain_key);
}

static void
ngx_http_lua_init_globals(lua_State *L, ngx_cycle_t *cycle,
    ngx_http_lua_main_conf_t *lmcf, ngx_log_t *log)
{
	// 省略了很多代码，只剩下 cycle 结构体入栈的代码了
    lua_pushlightuserdata(L, cycle);
    lua_setglobal(L, "__ngx_cycle");
}

```

从 lua_pushlightuserdata 的函数功能看，它只把指针到 Lua 虚拟机入栈并设置到全局变量中，也就是说， Lua 虚拟机内执行的 Lua 代码只能从全局变量 table 中按变量名找到这个结构体的指针，结构体本身还不能直接被 Lua 代码所使用。

那么 lua-nginx-module 是怎么让 Lua 代码使用上这些结构体的呢？我们行啊分析一下 ngx_chain_t 这个个案。

## ngx_http_request_t 使用

从上文对 ngx_http_lua_set_req 函数的代码阅读可以得知， ngx_http_request_t *r 最终会被塞进 __ngx_req 全局变量中，那么我们应该可以从 Lua 代码中找到一些蛛丝马迹吧。

为此，我们找到了 resty/core/request.lua 源文件，其实 resty/core 大量的使用了 __ngx_req 这个全局变量，我们来看一个最简单的函数，从中去体会其 ngx_http_request_t *r 的使用方法：

```lua

function ngx.req.start_time()
    local r = getfenv(0).__ngx_req
    if not r then
        return error("no request found")
    end

    return tonumber(C.ngx_http_lua_ffi_req_start_time(r))
end

```

我们看到 local r = getfenv(0).__ngx_req ，获取 __ngx_req 并保存在变量 r 中，在函数的最后，调用了 C.ngx_http_lua_ffi_req_start_time(r) ，这个函数
