#lua-nginx-module

##概述
lua-nginx-module 是 Openresty 最核心的模块，主要功能是在每个 nginx 的请求处理阶段以及后台定时事件中植入要运行的 lua 代码。凭借 Lua 的开发效率以及性能优势，为服务器开发提供了非常优质的解决方案。在阅读代码的过程中，我们主要需要了解如下方面：

*  lua-nginx-module 是怎么做到异步高并发的。
*  lua-nginx-module 是怎么把 Lua 代码植入到每一个请求处理阶段（phase）的。
*  lua-nginx-module 是怎么提供定时事件的 Lua 代码调用的。
*  等等

##模块定义

一切从最基本的层面入手，也就是咱们的模块定义

```c
ngx_http_module_t ngx_http_lua_module_ctx = {
    NULL,                             /*  preconfiguration */
    ngx_http_lua_init,                /*  postconfiguration */

    ngx_http_lua_create_main_conf,    /*  create main configuration */
    ngx_http_lua_init_main_conf,      /*  init main configuration */

    NULL,                             /*  create server configuration */
    NULL,                             /*  merge server configuration */

    ngx_http_lua_create_loc_conf,     /*  create location configuration */
    ngx_http_lua_merge_loc_conf       /*  merge location configuration */
};


ngx_module_t ngx_http_lua_module = {
    NGX_MODULE_V1,
    &ngx_http_lua_module_ctx,   /*  module context */
    ngx_http_lua_cmds,          /*  module directives */
    NGX_HTTP_MODULE,            /*  module type */
    NULL,                       /*  init master */
    NULL,                       /*  init module */
    ngx_http_lua_init_worker,   /*  init process */
    NULL,                       /*  init thread */
    NULL,                       /*  exit thread */
    NULL,                       /*  exit process */
    NULL,                       /*  exit master */
    NGX_MODULE_V1_PADDING
};
```

有了前面的nginx 模块代码阅读和开发基础之后，我们对这块代码应该不会陌生，我们对此有如下的基本认识：

*  lua-nginx-module 模块注册了 init process 过程回调 ngx_http_lua_init_worker
*  ngx_http_lua_cmds 结构是注册模块若干指令的数组
*  在 ngx_http_lua_module_ctx 中，注册了 postconfiguration 过程回调  ngx_http_lua_init
*  在 ngx_http_lua_module_ctx 中，注册了 create main configuration 过程回调  ngx_http_lua_create_main_conf
*  注册了 init main configuration 过程调用 ngx_http_lua_init_main_conf
*  注册了 create location configuration 过程调用 ngx_http_lua_create_loc_conf
*  注册了 merge location configuration 过程调用 ngx_http_lua_merge_loc_conf
*  没有注册 preconfiguration， create server configuration, merge server configuration 这三个过程的回调

## 指令集

看看所支持的指令集：

```c

static ngx_command_t ngx_http_lua_cmds[] = {

    { ngx_string("lua_max_running_timers"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      NGX_HTTP_MAIN_CONF_OFFSET,
      offsetof(ngx_http_lua_main_conf_t, max_running_timers),
      NULL },

    { ngx_string("lua_max_pending_timers"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      NGX_HTTP_MAIN_CONF_OFFSET,
      offsetof(ngx_http_lua_main_conf_t, max_pending_timers),
      NULL },

    { ngx_string("lua_shared_dict"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE2,
      ngx_http_lua_shared_dict,
      0,
      0,
      NULL },

#if (NGX_PCRE)
    { ngx_string("lua_regex_cache_max_entries"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      NGX_HTTP_MAIN_CONF_OFFSET,
      offsetof(ngx_http_lua_main_conf_t, regex_cache_max_entries),
      NULL },

    { ngx_string("lua_regex_match_limit"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      NGX_HTTP_MAIN_CONF_OFFSET,
      offsetof(ngx_http_lua_main_conf_t, regex_match_limit),
      NULL },
#endif

    { ngx_string("lua_package_cpath"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_http_lua_package_cpath,
      NGX_HTTP_MAIN_CONF_OFFSET,
      0,
      NULL },

    { ngx_string("lua_package_path"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_http_lua_package_path,
      NGX_HTTP_MAIN_CONF_OFFSET,
      0,
      NULL },

    { ngx_string("lua_code_cache"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_FLAG,
      ngx_http_lua_code_cache,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, enable_code_cache),
      NULL },

    { ngx_string("lua_need_request_body"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, force_read_body),
      NULL },

    { ngx_string("lua_transform_underscores_in_response_headers"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, transform_underscores_in_resp_headers),
      NULL },

     { ngx_string("lua_socket_log_errors"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, log_socket_errors),
      NULL },

    { ngx_string("init_by_lua"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_http_lua_init_by_lua,
      NGX_HTTP_MAIN_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_init_by_inline },

    { ngx_string("init_by_lua_file"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_http_lua_init_by_lua,
      NGX_HTTP_MAIN_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_init_by_file },

    { ngx_string("init_worker_by_lua"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_http_lua_init_worker_by_lua,
      NGX_HTTP_MAIN_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_init_worker_by_inline },

    { ngx_string("init_worker_by_lua_file"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_http_lua_init_worker_by_lua,
      NGX_HTTP_MAIN_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_init_worker_by_file },

#if defined(NDK) && NDK
    /* set_by_lua $res <inline script> [$arg1 [$arg2 [...]]] */
    { ngx_string("set_by_lua"),
      NGX_HTTP_SRV_CONF|NGX_HTTP_SIF_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                       |NGX_CONF_2MORE,
      ngx_http_lua_set_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_filter_set_by_lua_inline },

    /* set_by_lua_file $res rel/or/abs/path/to/script [$arg1 [$arg2 [..]]] */
    { ngx_string("set_by_lua_file"),
      NGX_HTTP_SRV_CONF|NGX_HTTP_SIF_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                       |NGX_CONF_2MORE,
      ngx_http_lua_set_by_lua_file,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_filter_set_by_lua_file },
#endif

    /* rewrite_by_lua <inline script> */
    { ngx_string("rewrite_by_lua"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_lua_rewrite_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_rewrite_handler_inline },

    /* access_by_lua <inline script> */
    { ngx_string("access_by_lua"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_lua_access_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_access_handler_inline },

    /* content_by_lua <inline script> */
    { ngx_string("content_by_lua"),
      NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF|NGX_CONF_TAKE1,
      ngx_http_lua_content_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_content_handler_inline },

    /* log_by_lua <inline script> */
    { ngx_string("log_by_lua"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_lua_log_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_log_handler_inline },

    { ngx_string("rewrite_by_lua_file"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_lua_rewrite_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_rewrite_handler_file },

    { ngx_string("rewrite_by_lua_no_postpone"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      NGX_HTTP_MAIN_CONF_OFFSET,
      offsetof(ngx_http_lua_main_conf_t, postponed_to_rewrite_phase_end),
      NULL },

    { ngx_string("access_by_lua_file"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_lua_access_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_access_handler_file },

    /* content_by_lua_file rel/or/abs/path/to/script */
    { ngx_string("content_by_lua_file"),
      NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF|NGX_CONF_TAKE1,
      ngx_http_lua_content_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_content_handler_file },

    { ngx_string("log_by_lua_file"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_lua_log_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_log_handler_file },

    /* header_filter_by_lua <inline script> */
    { ngx_string("header_filter_by_lua"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_lua_header_filter_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_header_filter_inline },

    { ngx_string("header_filter_by_lua_file"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_lua_header_filter_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_header_filter_file },

    { ngx_string("body_filter_by_lua"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_lua_body_filter_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_body_filter_inline },

    { ngx_string("body_filter_by_lua_file"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_lua_body_filter_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_body_filter_file },

    { ngx_string("lua_socket_keepalive_timeout"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF
          |NGX_HTTP_LIF_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_msec_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, keepalive_timeout),
      NULL },

    { ngx_string("lua_socket_connect_timeout"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF
          |NGX_HTTP_LIF_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_msec_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, connect_timeout),
      NULL },

    { ngx_string("lua_socket_send_timeout"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF
          |NGX_HTTP_LIF_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_msec_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, send_timeout),
      NULL },

    { ngx_string("lua_socket_send_lowat"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF
          |NGX_HTTP_LIF_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_size_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, send_lowat),
      &ngx_http_lua_lowat_post },

    { ngx_string("lua_socket_buffer_size"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF
          |NGX_HTTP_LIF_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_size_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, buffer_size),
      NULL },

    { ngx_string("lua_socket_pool_size"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF
                        |NGX_HTTP_LIF_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, pool_size),
      NULL },

    { ngx_string("lua_socket_read_timeout"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF
          |NGX_HTTP_LIF_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_msec_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, read_timeout),
      NULL },

    { ngx_string("lua_http10_buffering"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, http10_buffering),
      NULL },

    { ngx_string("lua_check_client_abort"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, check_client_abort),
      NULL },

    { ngx_string("lua_use_default_type"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_FLAG,
      ngx_conf_set_flag_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, use_default_type),
      NULL },

#if (NGX_HTTP_SSL)

#   if defined(nginx_version) && nginx_version >= 1001013

    { ngx_string("lua_ssl_protocols"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_1MORE,
      ngx_conf_set_bitmask_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, ssl_protocols),
      &ngx_http_lua_ssl_protocols },

#   endif

    { ngx_string("lua_ssl_ciphers"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_str_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, ssl_ciphers),
      NULL },

    { ngx_string("lua_ssl_verify_depth"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_num_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, ssl_verify_depth),
      NULL },

    { ngx_string("lua_ssl_trusted_certificate"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_str_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, ssl_trusted_certificate),
      NULL },

    { ngx_string("lua_ssl_crl"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE1,
      ngx_conf_set_str_slot,
      NGX_HTTP_LOC_CONF_OFFSET,
      offsetof(ngx_http_lua_loc_conf_t, ssl_crl),
      NULL },

#endif  /* NGX_HTTP_SSL */

    ngx_null_command
};

```
这里可以看出指令可以简单分成三类：
*  向各个阶段植入 Lua 代码的指令，例如：
    *  init_by_lua
    *  init_by_lua_file
    *  init_worker_by_lua
    *  init_worker_by_lua_file
    *  set_by_lua
    *  set_by_lua_file
    *  rewrite_by_lua
    *  rewrite_by_lua_file
    *  access_by_lua
    *  access_by_lua_file
    *  content_by_lua
    *  content_by_lua_file
    *  log_by_lua
    *  log_by_lua_file
    *  header_filter_by_lua
    *  header_filter_by_lua_file
    *  body_filter_by_lua
    *  body_filter_by_lua_file
    
    
*  向模块的 main 上下文配置结构体设置参数的指令，例如：
    *  lua_max_running_timers
    *  max_pending_timers
    *  ngx_http_lua_shared_dict
    *  regex_cache_max_entries
    *  regex_match_limit
    *  ngx_http_lua_package_cpath
    *  ngx_http_lua_package_path
    *  rewrite_by_lua_no_postpone
    
*  向模块的 location 或者 server 上下文配置结构体设置参数的指令，例如：
    *  lua_code_cache
    *  lua_need_request_body
    *  lua_transform_underscores_in_response_headers
    *  lua_socket_log_errors
    *  lua_socket_keepalive_timeout
    *  lua_socket_connect_timeout
    *  lua_socket_send_timeout
    *  lua_socket_send_lowat
    *  lua_socket_buffer_size
    *  lua_socket_pool_size
    *  lua_socket_read_timeout
    *  lua_http10_buffering
    *  lua_check_client_abort
    *  lua_use_default_type
    *  lua_ssl_protocols
    *  lua_ssl_ciphers
    *  lua_ssl_verify_depth
    *  lua_ssl_trusted_certificate
    *  lua_ssl_crl

我们先从植入 Lua 代码的指令说起，请往下一节阅读 lua_code_execution.md

