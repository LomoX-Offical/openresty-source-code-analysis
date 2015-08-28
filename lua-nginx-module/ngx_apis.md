# ngx api

## 概述

在前文我们了解基本的 nginx 阶段植入 Lua 的原理和代码，那么 Lua 代码为了实现服务器业务逻辑，除了基本的 Lua 语法支持之外，lua-nginx-module 还提供了一套能让 Lua 代码调用的内置 API ，并且都封装在 ngx 这个包（ package ）之中，这套 API 包含了几个方面的内容：

*  读写 request 参数的 arg api，包括
	*  arg 数组的 \_\_index 和 \_\_newindex

*  关于日志输出的 log api，包括
	*  log
	*  全局函数 print

*  向客户端发送数据的 output api，包括
	*  send_headers
	*  print
	*  say
	*  flush
	*  eof

*  有关时间的 time api ，包括：
	*  utctime
	*  get_now_ts
	*  get_now
	*  localtime
	*  time
	*  now
	*  update_time
	*  get_today
	*  today
	*  cookie_time
	*  http_time
	*  parse_http_time
	
*  字符串辅助操作相关的 string api，包括：
	*  escape_uri
	*  unescape_uri
	*  encode_args
	*  decode_args
	*  quote_sql_str
	*  decode_base64
	*  encode_base64
	*  md5_bin
	*  md5
	*  sha1_bin
	*  crc32_short
	*  crc32_long
	*  hmac_sha1
	
*  流程控制相关的 control api，包括：
	*  redirect
	*  exec
	*  throw_error
	*  exit
	*  on_abort

*  子请求操作相关的 subrequest api ，他们都封装在 location 包中，包括：
	*  capture
	*  capture_multi

*  提供休眠操作的 sleep api ，它只包含一个函数：
	*  sleep
	
*  提供阶段信息的 phase api ，它目前只包含一个函数：
	*  get_phase
	
*  提供正则操作的 regex api，他们被封装在 re 包内，包括：
	*  find
	*  match
	*  gmatch
	*  sub
	*  gsub
	
*  提供请求相关数据的 req api ，他们都封装在 req 包中，其中又包括好几个系列的 api：
	*  访问和操作 header 信息的 req header api
		*  http_version
		*  raw_header
		*  clear_header
		*  set_header
		*  get_headers 

	*  修改 uri 信息的 req_uri_api
		*  set_uri

	*  访问和操作请求的 arg 的 req args api
		*  set_uri_args
		*  get_uri_args
		*  get_query_args
		*  get_post_args
		
	*  访问和操作请求 body 的 req body api
		*  read_body
		*  discard_body
		*  get_body_data
		*  get_body_file
		*  set_body_data
		*  set_body_file
		*  init_body
		*  append_body
		*  finish_body
	
	*  获取请求相关的 socket 套接字封装 cosocket 的 req socket api ，它目前只包含一个函数：
		*  socket
		
	*  访问和操作请求方法的 req method api
		*  get_method
		*  set_method
		
	*  访问请求开始时间的 req time api
		*  start_time
		
*  提供应答 header 相关数据操作的 resp header api ，他们都封装在 resp 包中，包括：
	*  名为 header 的 table，包含 \_\_index 以及 \_\_newindex 的重载
	*  get_headers
	
*  提供访问变量相关数据操作的 variable api ，包括：
	*  名为 var 的 table，包含 \_\_index 以及 \_\_newindex 的重载
	
*  提供共享内存相关操作的 shdict api，他们都封装在 shared 包中，包括：
	*  get
	*  get_stale
	*  set
	*  safe_set
	*  add
	*  safe_add
	*  replace
	*  incr
	*  delete
	*  flush_all
	*  flush_expired
	*  get_keys
	*  get_keys
	*  重载了 __index 操作
	
*  提供创建 cosocket 的 socket tcp api ，他们都封装在 socket 包中，包括
	*  connect
	*  tcp
	*  connect 和 tcp 创建出来的 cosocket ，有如下函数提供：
		*  connect
		*  sslhandshake
		*  receive
		*  receiveuntil
		*  send
		*  close
		*  setoption
		*  settimeout
		*  getreusedtimes
		*  setkeepalive

	*  udp
	*  udp 创建出来的 cosocket ，有如下函数提供：
		*  setpeername
		*  send
		*  receive
		*  close
		*  settimeout
		
*  提供操作用户轻量线程的 uthread api ，他们都封装在 thread 包中，包括
	*  spawn
	*  wait
	*  kill

*  提供定时器操作的 timer api ，只包含一个函数，它封装在 timer 包中：
	*  at
	
*  提供获取编译配置信息的 config api ，他们封装在 config 包中，包括：
	*  debug
	*  prefix
	*  nginx_version
	*  ngx_lua_version
	*  nginx_configure
	
*  提供工作进程相关信息的 worker api ，他们封装在 worker 包中，包括：
	*  exiting
	*  pid

*  提供 coroutine 协程操作的 coroutine api，他们封装在 coroutine 包之中，包括：
	*  create
	*  resume
	*  yield
	*  wrap
	*  running
	*  status

*  另外还有一些杂项接口 misc api
	*  对 ngx 重载了 \_\_index 和 \_\_newindex
	*  status
	*  ctx
	*  is_subrequest
	*  headers_sent

除了 API 之外，其实还有很多常量也在里边定义了，包括：
*  核心常量 core consts，包括：
	*  OK
	*  AGAIN
	*  DONE
	*  DECLINED
	*  ERROR
	*  null

*  http 常量 http consts ，包括：
	*  HTTP_GET
	*  HTTP_POST
	*  HTTP_PUT
	*  HTTP_HEAD
	*  HTTP_DELETE
	*  HTTP_OPTIONS
	*  HTTP_MKCOL
	*  HTTP_COPY
	*  HTTP_MOVE
	*  HTTP_PROPFIND
	*  HTTP_PROPPATCH
	*  HTTP_LOCK
	*  HTTP_UNLOCK
	*  HTTP_PATCH
	*  HTTP_TRACE
	*  HTTP_OK
	*  HTTP_CREATED
	*  HTTP_SPECIAL_RESPONSE
	*  HTTP_MOVED_PERMANENTLY
	*  HTTP_MOVED_TEMPORARILY
	*  HTTP_SEE_OTHER
	*  HTTP_NOT_MODIFIED
	*  HTTP_BAD_REQUEST
	*  HTTP_UNAUTHORIZED
	*  HTTP_FORBIDDEN
	*  HTTP_NOT_FOUND
	*  HTTP_NOT_ALLOWED
	*  HTTP_GONE
	*  HTTP_INTERNAL_SERVER_ERROR
	*  HTTP_METHOD_NOT_IMPLEMENTED
	*  HTTP_SERVICE_UNAVAILABLE
	*  HTTP_GATEWAY_TIMEOUT
	
*  日志级别常量 log consts ，包括：
	*  STDERR
	*  EMERG
	*  ALERT
	*  CRIT
	*  ERR
	*  WARN
	*  NOTICE
	*  INFO
	*  DEBUG


这些 API 以及常量可以使得 Lua 能渗透到 nginx 业务逻辑的方方面面，我们利用这些 API 以及常量能轻松对请求的信息以及应答的信息进行操作。
那么这些 API 以及常量是怎么提供的呢？

## inject

回顾一下我们之前分析过 ngx_http_lua_init 函数吧，它在模块 postconfiguration 阶段被调用，它的功能除了事先就往各个阶段插入 handler、调用了 lmcf->init_handler 回调函数之外，它还调用了 ngx_http_lua_init_vm 函数去初始化 Lua 虚拟机，这里我们对初始化 Lua 虚拟机的这个 ngx_http_lua_init_vm 函数进行展开。

```c

lua_State *
ngx_http_lua_init_vm(lua_State *parent_vm, ngx_cycle_t *cycle,
    ngx_pool_t *pool, ngx_http_lua_main_conf_t *lmcf, ngx_log_t *log,
    ngx_pool_cleanup_t **pcln)
{
    // 创建新的 Lua 虚拟机
    L = ngx_http_lua_new_state(parent_vm, cycle, lmcf, log);
    if (L == NULL) {
        return NULL;
    }

    // Lua 虚拟机清理句柄设置
    cln->handler = ngx_http_lua_cleanup_vm;

    state = ngx_alloc(sizeof(ngx_http_lua_vm_state_t), log);
    state->vm = L;
    state->count = 1;

    cln->data = state;
    if (pcln) {
        *pcln = cln;
    }

    if (lmcf->preload_hooks) {
		// 第三方模块加载
        lua_getglobal(L, "package");
        lua_getfield(L, -1, "preload");
        hook = lmcf->preload_hooks->elts;
        for (i = 0; i < lmcf->preload_hooks->nelts; i++) {
            ngx_http_lua_probe_register_preload_package(L, hook[i].package);
            lua_pushcfunction(L, hook[i].loader);
            lua_setfield(L, -2, (char *) hook[i].package);
        }
        lua_pop(L, 2);
    }

    return L;
}

```

这个函数大致做了三个事情，
*  调用 ngx_http_lua_new_state 创建新的 Lua 虚拟机;
*  给内存池设置清理 Lua 虚拟机的回调函数;
*  对已注册的 preload_hooks 第三方模块进行加载

其他的暂且不说，我们先说 ngx_http_lua_new_state 这个函数，其实他也没有想象中单纯，咱们看看代码：

```c
static lua_State *
ngx_http_lua_new_state(lua_State *parent_vm, ngx_cycle_t *cycle,
    ngx_http_lua_main_conf_t *lmcf, ngx_log_t *log)
{
	// Lua API ：创建 Lua 虚拟机
    L = luaL_newstate();

    // Lua API ：打开指定状态机中的所有 Lua 标准库。
    luaL_openlibs(L);

	// 把全局的 package table 入栈，是一个模块导出函数列表
    lua_getglobal(L, "package");

	// 初始化 package table 的 path 和 cpath ， 也就是搜索路径
    if (parent_vm) {
		// 如果有 parent_vm ，这里选择继承过来。
        lua_getglobal(parent_vm, "package");
        lua_getfield(parent_vm, -1, "path");
        old_path = lua_tolstring(parent_vm, -1, &old_path_len);
        lua_pop(parent_vm, 1);

        lua_pushlstring(L, old_path, old_path_len);
        lua_setfield(L, -2, "path");

        lua_getfield(parent_vm, -1, "cpath");
        old_path = lua_tolstring(parent_vm, -1, &old_path_len);
        lua_pop(parent_vm, 2);

        lua_pushlstring(L, old_path, old_path_len);
        lua_setfield(L, -2, "cpath");

    } else {
		// 这个是用 lua_package_path 以及 lua_package_cpath 指令设置的路径初始化 package 的搜索路径
        if (lmcf->lua_path.len != 0) {
            lua_getfield(L, -1, "path"); /* get original package.path */
            old_path = lua_tolstring(L, -1, &old_path_len);

            lua_pushlstring(L, (char *) lmcf->lua_path.data,
                            lmcf->lua_path.len);
            new_path = lua_tostring(L, -1);

            ngx_http_lua_set_path(cycle, L, -3, "path", new_path, old_path,
                                  log);
            lua_pop(L, 2);
        }

        if (lmcf->lua_cpath.len != 0) {
            lua_getfield(L, -1, "cpath"); /* get original package.cpath */
            old_cpath = lua_tolstring(L, -1, &old_cpath_len);

            lua_pushlstring(L, (char *) lmcf->lua_cpath.data,
                            lmcf->lua_cpath.len);
            new_cpath = lua_tostring(L, -1);

            ngx_http_lua_set_path(cycle, L, -3, "cpath", new_cpath, old_cpath,
                                  log);
           lua_pop(L, 2);
        }
    }

	// package table 操作完毕，出栈
    lua_pop(L, 1);

	// 初始化注册表
    ngx_http_lua_init_registry(L, log);

	// 初始化全局变量
    ngx_http_lua_init_globals(L, cycle, lmcf, log);

    return L;
}

```

这个函数干了比较多的事情，除了创建新的 Lua 虚拟机之外，还对虚拟机加载 Lua 标准库，对 package 设置了搜索路径，最后还调用了两函数：
*  ngx_http_lua_init_registry 初始化注册表
*  ngx_http_lua_init_globals 初始化全局变量

在看看 ngx_http_lua_init_registry ：

```c
static void
ngx_http_lua_init_registry(lua_State *L, ngx_log_t *log)
{
	// 创建协程信息专用的 table ，以 ngx_http_lua_coroutines_key 地址作为索引插入注册表
    lua_pushlightuserdata(L, &ngx_http_lua_coroutines_key);
    lua_createtable(L, 0, 32 /* nrec */);
    lua_rawset(L, LUA_REGISTRYINDEX);

    // 创建存储请求处理的上下文信息的 table ，以 ngx_http_lua_ctx_tables_key 地址作为索引插入注册表
    lua_pushliteral(L, ngx_http_lua_ctx_tables_key);
    lua_createtable(L, 0, 32 /* nrec */);
    lua_rawset(L, LUA_REGISTRYINDEX);

    // 创建存储 cosocket 连接池信息的 table ，，以 ngx_http_lua_socket_pool_key 地址作为索引插入注册表
    lua_pushlightuserdata(L, &ngx_http_lua_socket_pool_key);
    lua_createtable(L, 0, 8 /* nrec */);
    lua_rawset(L, LUA_REGISTRYINDEX);

#if (NGX_PCRE)
    // 创建预编译正则专用的 table ，以 ngx_http_lua_regex_cache_key 地址作为索引插入注册表
    lua_pushlightuserdata(L, &ngx_http_lua_regex_cache_key);
    lua_createtable(L, 0, 16 /* nrec */);
    lua_rawset(L, LUA_REGISTRYINDEX);
#endif

    // 创建存储代码块缓存的 table ，以 ngx_http_lua_code_cache_key 地址作为索引插入注册表
    lua_pushlightuserdata(L, &ngx_http_lua_code_cache_key);
    lua_createtable(L, 0, 8 /* nrec */);
    lua_rawset(L, LUA_REGISTRYINDEX);
}

```
从代码分析看， ngx_http_lua_init_registry 在注册表创建了若干个 table 分别对应不同的业务，提供全局访问。

我们继续往下一个函数进发，阅读 ngx_http_lua_init_globals 的代码：

```c

static void
ngx_http_lua_init_globals(lua_State *L, ngx_cycle_t *cycle,
    ngx_http_lua_main_conf_t *lmcf, ngx_log_t *log)
{
	// 把 cycle 数据以轻量用户数据方式塞入了全局变量 __ngx_cycle 中。
    lua_pushlightuserdata(L, cycle);
    lua_setglobal(L, "__ngx_cycle");

	// 注入 ndk api
    ngx_http_lua_inject_ndk_api(L);
	
	// 注入 ngx api
    ngx_http_lua_inject_ngx_api(L, lmcf, log);
}

void
ngx_http_lua_inject_ndk_api(lua_State *L)
{
    lua_createtable(L, 0, 1 /* nrec */);    /* ndk.* */

    lua_newtable(L);    /* .set_var */

    lua_createtable(L, 0, 2 /* nrec */); /* metatable for .set_var */
    lua_pushcfunction(L, ngx_http_lua_ndk_set_var_get);
    lua_setfield(L, -2, "__index");
    lua_pushcfunction(L, ngx_http_lua_ndk_set_var_set);
    lua_setfield(L, -2, "__newindex");
    lua_setmetatable(L, -2);

    lua_setfield(L, -2, "set_var");

    lua_getglobal(L, "package"); /* ndk package */
    lua_getfield(L, -1, "loaded"); /* ndk package loaded */
    lua_pushvalue(L, -3); /* ndk package loaded ndk */
    lua_setfield(L, -2, "ndk"); /* ndk package loaded */
    lua_pop(L, 2);

    lua_setglobal(L, "ndk");
}
```

代码分析到这里终于见到了本节的重点，向 Lua 虚拟机进行 api 的注入。
从代码可见， ngx_http_lua_init_globals 干了三件事情：
*  向 Lua 虚拟机塞入了 cycle 数据。
*  调用 ngx_http_lua_inject_ndk_api ，注入了 ndk_api ，允许 Lua 代码调用 ndk_api。
*  调用 ngx_http_lua_inject_ngx_api ，注入了 ngx_api ，允许 Lua 代码调用 ngx_api。

我们分析一下 ngx_http_lua_inject_ndk_api ，他的主要功能就是：
*  创建了 ndk 这个模块。
*  创建了 table ndk.set_var ，并重载了  __index 和 __newindex 这两个函数。
*  把 ndk 注册到 package 中，设置已加载状态。
*  把 ndk 设置到全局变量中。


## ngx api

我们继续关注 lua-nginx-module 提供的 API：

```c
static void
ngx_http_lua_inject_ngx_api(lua_State *L, ngx_http_lua_main_conf_t *lmcf,
    ngx_log_t *log)
{
    lua_createtable(L, 0 /* narr */, 99 /* nrec */);    /* ngx.* */

    lua_pushcfunction(L, ngx_http_lua_get_raw_phase_context);
    lua_setfield(L, -2, "_phase_ctx");

    ngx_http_lua_inject_arg_api(L);

    ngx_http_lua_inject_http_consts(L);
    ngx_http_lua_inject_core_consts(L);

    ngx_http_lua_inject_log_api(L);
    ngx_http_lua_inject_output_api(L);
    ngx_http_lua_inject_time_api(L);
    ngx_http_lua_inject_string_api(L);
    ngx_http_lua_inject_control_api(log, L);
    ngx_http_lua_inject_subrequest_api(L);
    ngx_http_lua_inject_sleep_api(L);
    ngx_http_lua_inject_phase_api(L);

#if (NGX_PCRE)
    ngx_http_lua_inject_regex_api(L);
#endif

    ngx_http_lua_inject_req_api(log, L);
    ngx_http_lua_inject_resp_header_api(L);
    ngx_http_lua_create_headers_metatable(log, L);
    ngx_http_lua_inject_variable_api(L);
    ngx_http_lua_inject_shdict_api(lmcf, L);
    ngx_http_lua_inject_socket_tcp_api(log, L);
    ngx_http_lua_inject_socket_udp_api(log, L);
    ngx_http_lua_inject_uthread_api(log, L);
    ngx_http_lua_inject_timer_api(L);
    ngx_http_lua_inject_config_api(L);
    ngx_http_lua_inject_worker_api(L);

    ngx_http_lua_inject_misc_api(L);

    lua_getglobal(L, "package"); /* ngx package */
    lua_getfield(L, -1, "loaded"); /* ngx package loaded */
    lua_pushvalue(L, -3); /* ngx package loaded ngx */
    lua_setfield(L, -2, "ngx"); /* ngx package loaded */
    lua_pop(L, 2);

    lua_setglobal(L, "ngx");

    ngx_http_lua_inject_coroutine_api(log, L);
}

```
从这个函数可以看出， 大概会类似 ngx_http_lua_inject_ndk_api 这样向 Lua 虚拟机插入各种 C 函数并注册到 ngx 模块内，这里可以看到所有 ngx api 的注册流程了。
最后，往 package table 插入 ngx 模块，允许 Lua 通过 ngx.XXX 的方式访问这里注入的各种 api ，里边包含的各种类型的 api 集我们往后会分别展开。我们这里先展开其中一个小 api 集。

## ngx_http_lua_inject_time_api

咱们看看基本注入流程是咋样的：

```c

void
ngx_http_lua_inject_time_api(lua_State *L)
{
    lua_pushcfunction(L, ngx_http_lua_ngx_utctime);
    lua_setfield(L, -2, "utctime");

    lua_pushcfunction(L, ngx_http_lua_ngx_time);
    lua_setfield(L, -2, "get_now_ts"); /* deprecated */

    lua_pushcfunction(L, ngx_http_lua_ngx_localtime);
    lua_setfield(L, -2, "get_now"); /* deprecated */

    lua_pushcfunction(L, ngx_http_lua_ngx_localtime);
    lua_setfield(L, -2, "localtime");

    lua_pushcfunction(L, ngx_http_lua_ngx_time);
    lua_setfield(L, -2, "time");

    lua_pushcfunction(L, ngx_http_lua_ngx_now);
    lua_setfield(L, -2, "now");

    lua_pushcfunction(L, ngx_http_lua_ngx_update_time);
    lua_setfield(L, -2, "update_time");

    lua_pushcfunction(L, ngx_http_lua_ngx_today);
    lua_setfield(L, -2, "get_today"); /* deprecated */

    lua_pushcfunction(L, ngx_http_lua_ngx_today);
    lua_setfield(L, -2, "today");

    lua_pushcfunction(L, ngx_http_lua_ngx_cookie_time);
    lua_setfield(L, -2, "cookie_time");

    lua_pushcfunction(L, ngx_http_lua_ngx_http_time);
    lua_setfield(L, -2, "http_time");

    lua_pushcfunction(L, ngx_http_lua_ngx_parse_http_time);
    lua_setfield(L, -2, "parse_http_time");
}
```
Oh，原来流程是这么的简单的：

*  先调用 lua_pushcfunction 函数把 C 函数指针入栈。
*  再调用 lua_setfield 把刚入栈的这个 C 函数对象作为 table 的某个 key 插入。

然后注入 C 函数的流程可就完成了。
这里我们简单看一个函数的实现：

```c
static int
ngx_http_lua_ngx_now(lua_State *L)
{
    ngx_time_t              *tp;
    tp = ngx_timeofday();
    lua_pushnumber(L, (lua_Number) (tp->sec + tp->msec / 1000.0L));
    return 1;
}

```

在这个注入的函数 ngx_http_lua_ngx_now 中，他要实现的是返回一个当前时间，这个函数也实现得相当的简单，通过调用 ngx_timeofday 获取当前时间，然后各种计算，并通过 lua_pushnumber 向 Lua 虚拟机返回一个数字对象就完成了。


## 总结
在这里我们需要记住整个注册流程：

*  调用 luaL_newstate 创建新的 Lua 虚拟机；
*  在 Lua 虚拟机打开指定状态机中的所有 Lua 标准库；
*  初始化 package table 的 path 和 cpath 这两个搜索路径；
*  创建存储 coroutine、ctx tables 、 cosocket pool 、 regex cache 、 code cache  的 table ，以 ngx_http_lua_coroutines_key 地址作为索引插入注册表；
*  向全局变量添加 ndk ，注册 ndk.set_var 的 C 函数；
*  向全局变量添加 ngx ，注册 ndk.XXXXX 的 C 函数；
*  对已注册的 preload_hooks 第三方模块进行加载。

这个流程在 ngx_http_lua_init 函数实现，该函数在 postconfiguration 阶段被调用，并且在这里 创建并初始化出来的 Lua 虚拟机会作为整个 nginx 进程运行过程中的 Lua 虚拟机母体存在，当有请求处理流程进入 lua-nginx-module 的 handler ，会在执行 Lua 代码前，从 Lua 虚拟机母体以各种方式 copy 一份新的 Lua 虚拟机出来执行。

