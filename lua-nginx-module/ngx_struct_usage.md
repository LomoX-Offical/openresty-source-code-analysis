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

我们看到 local r = getfenv(0).__ngx_req ，获取 __ngx_req 并保存在变量 r 中，在函数的最后，调用了 C.ngx_http_lua_ffi_req_start_time(r) 。
C 这个变量的定义在这里：

```lua

local C = ffi.C

ffi.cdef[[
    typedef struct {
        ngx_http_lua_ffi_str_t   key;
        ngx_http_lua_ffi_str_t   value;
    } ngx_http_lua_ffi_table_elt_t;

    int ngx_http_lua_ffi_req_get_headers_count(ngx_http_request_t *r,
        int max);

    int ngx_http_lua_ffi_req_get_headers(ngx_http_request_t *r,
        ngx_http_lua_ffi_table_elt_t *out, int count, int raw);

    int ngx_http_lua_ffi_req_get_uri_args_count(ngx_http_request_t *r,
        int max);

    size_t ngx_http_lua_ffi_req_get_querystring_len(ngx_http_request_t *r);

    int ngx_http_lua_ffi_req_get_uri_args(ngx_http_request_t *r,
        unsigned char *buf, ngx_http_lua_ffi_table_elt_t *out, int count);

    double ngx_http_lua_ffi_req_start_time(ngx_http_request_t *r);

    int ngx_http_lua_ffi_req_get_method(ngx_http_request_t *r);

    int ngx_http_lua_ffi_req_get_method_name(ngx_http_request_t *r,
        char *name, size_t *len);

    int ngx_http_lua_ffi_req_set_method(ngx_http_request_t *r, int method);

    int ngx_http_lua_ffi_req_header_set_single_value(ngx_http_request_t *r,
        const unsigned char *key, size_t key_len, const unsigned char *value,
        size_t value_len);
]];

```

这里要解释一下 ffi.C 的功能。在 http://luajit.org/ 官方API文档上，我们可以得知，ffi.C 是一个 C 语言基础库的命名空间， Lua 通过访问 ffi.C 可以调用 C 语言的基础库提供的函数，不同的系统实现和最终提供的函数列表又有一些区别：
*  在 POSIX 系统上（比如 Linux 或者大多数 Unix 、BSD 系统），通过 ffi.C 可以访问到：
    *  libc, libm, libdl (仅在 Linux ), libgcc (如果 Luajit 是使用 GCC 编译的话) 等基础依赖库的 API 函数，
    *  宿主程序本身的所有导出函数，
    *  以及 Luajit 自身提供的 API 函数。
*  在 windows 系统上，通过 ffi.C 可以访问到：
    *  宿主程序本身的所有导出函数，
    *  Luajit 自身提供的 API 函数，
    *  Luajit 依赖的基础动态库（比如 msvcrt*.dll, kernel32.dll, user32.dll and gdi32.dll 等）的 API 函数。
    
而在通过 ffi.C 访问这一系列 API 之前，必须通过 ffi.cdef 接口声明 C 函数，在我们上例代码中，可以看到 ngx_http_lua_ffi_req_start_time 出现在 ffi.cdef 语句的函数声明列表之中。

在这样的声明语句之后，我们就可以通过 C.ngx_http_lua_ffi_req_start_time(r) 的方式调用 C 函数 ngx_http_lua_ffi_req_start_time ，并把 r 作为参数传入这个 C 函数中。

那么 Lua 代码中所使用的 ngx_http_lua_ffi_req_start_time 函数到底在哪里定义呢？
我们通过翻查 lua-nginx-module 的代码，又可以找到 ngx_http_lua_time.c 这个文件中的一个函数定义：

```c
double
ngx_http_lua_ffi_req_start_time(ngx_http_request_t *r)
{
    return r->start_sec + r->start_msec / 1000.0;
}

```

由此可见，是 lua-nginx-module 导出并提供了这个 ngx_http_lua_ffi_req_start_time 函数给 Lua 代码调用。

整理一下顺序：

*  我们通过在 Lua 虚拟机执行 Lua 代码块之前，先对 Lua 虚拟机执行 ngx_http_lua_set_req(L, r) 把 request 请求结构体以指针方式设置进全局变量 __ngx_req 中，
*  在 Lua 虚拟机执行 Lua 代码块的时候， Lua 代码可以不可以直接访问 request 的成员，但可以通过调用 ngx.req.start_time() 等一系列 Lua 函数间接使用了刚赋值到全局变量 __ngx_req 的 reqeust 请求结构体指针，
*  在 ngx.req.start_time() 函数执行过程中， 会通过 ffi.C 调用 C 函数 ngx_http_lua_ffi_req_start_time ，并把 request 请求结构体指针作为参数传入该 C 函数中。
*  在 lua-nginx-module 的 ngx_http_lua_ffi_req_start_time 函数通过实参 request 结构体的指针，计算出返回值完成这个获取请求开始时间戳的功能。

其他的功能实现如果需要访问 request 请求结构体去完成功能，都是以这种类似 ngx.req.start_time() 的方法去间接使用 request 请求结构体。


## todo 是在没办法编下去了