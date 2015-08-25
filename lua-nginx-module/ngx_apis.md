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

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, log, 0, "lua initialize the "
                   "global Lua VM %p", L);

    /* register cleanup handler for Lua VM */
    cln->handler = ngx_http_lua_cleanup_vm;

    state = ngx_alloc(sizeof(ngx_http_lua_vm_state_t), log);
    if (state == NULL) {
        return NULL;
    }
    state->vm = L;
    state->count = 1;

    cln->data = state;

    if (pcln) {
        *pcln = cln;
    }

    if (lmcf->preload_hooks) {

        /* register the 3rd-party module's preload hooks */

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
