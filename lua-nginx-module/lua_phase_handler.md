#Phase Handler

## Lua 在请求中

Lua 在初始化过程中的调用咱们都有大致的了解了，但仅仅是这些只是，我们对 Lua 在模块中执行的理解还不够，接下来咱们来深入了解请求到来时的各个 Phase 被插入的 handler：

## rewrite_by_lua

指令定义：

```c
    { ngx_string("rewrite_by_lua"),
      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF
                        |NGX_CONF_TAKE1,
      ngx_http_lua_rewrite_by_lua,
      NGX_HTTP_LOC_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_rewrite_handler_inline },
```
能看出来 rewrite_by_lua 跟 init_by_lua 不一样的地方是，允许在 main ，server ， location ， if 上下文中使用，可以看出 rewrite_by_lua 的使用将会比 init_by_lua* 更加灵活。
从 ngx_http_lua_init 函数代码分析我们可以得知， rewrite_by_lua 指令定义的 Lua 代码，首先是把 ngx_http_lua_rewrite_handler 插入到 cmcf->phases[NGX_HTTP_REWRITE_PHASE].handlers 列表中，在一个请求处理的 rewrite 阶段被执行。
那么被插入的 ngx_http_lua_rewrite_handler 函数具体的实现了什么：

```c
ngx_int_t
ngx_http_lua_rewrite_handler(ngx_http_request_t *r)
{
    ngx_http_lua_loc_conf_t     *llcf;
    ngx_http_lua_ctx_t          *ctx;
    ngx_int_t                    rc;

    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);

	// 如果没有设置 llcf->rewrite_handler 回调，那就让下一个 handler 处理吧。
    if (llcf->rewrite_handler == NULL) {
        return NGX_DECLINED;
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);

    if (ctx->entered_rewrite_phase) {
        rc = ctx->resume_handler(r);

        if (rc == NGX_OK) {
            rc = NGX_DECLINED;
        }

        if (rc == NGX_DECLINED) {
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
        }

        return rc;
    }

    if (ctx->waiting_more_body) {
        return NGX_DONE;
    }

    if (llcf->force_read_body && !ctx->read_body_done) {
        r->request_body_in_single_buf = 1;
        r->request_body_in_persistent_file = 1;
        r->request_body_in_clean_file = 1;

        rc = ngx_http_read_client_request_body(r,
                                       ngx_http_lua_generic_phase_post_read);

        if (rc == NGX_ERROR || rc >= NGX_HTTP_SPECIAL_RESPONSE) {
#if (nginx_version < 1002006) ||                                             \
        (nginx_version >= 1003000 && nginx_version < 1003009)
            r->main->count--;
#endif

            return rc;
        }

        if (rc == NGX_AGAIN) {
            ctx->waiting_more_body = 1;
            return NGX_DONE;
        }
    }

    dd("calling rewrite handler");
    return llcf->rewrite_handler(r);
}
```





 