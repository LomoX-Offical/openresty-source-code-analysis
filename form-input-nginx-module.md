#form-input-nginx-module

##概述：
这是 Openresty 中一个用于处理 HTTP 请求的 POST 以及 PUT 方法，在协议头 Content-Type 是 application/x-www-form-urlencoded 的情况下，解析请求实体内容并按 nginx 变量存储的模块。

这个模块的开发和编译需要依赖 ngx_devel_kit 模块 (NDK)。

整个模块只有一个 c 文件 ngx_http_form_input_module.c，我们看看他里边的内容：

##指令集：
咱们看看指令结构体的定义：

```c
	
	static ngx_command_t ngx_http_form_input_commands[] = {
	
	    { ngx_string("set_form_input"),
	      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE12,
	      ngx_http_set_form_input_conf_handler,
	      NGX_HTTP_LOC_CONF_OFFSET,
	      0,
	      NULL },
	
	    { ngx_string("set_form_input_multi"),
	      NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE12,
	      ngx_http_set_form_input_conf_handler,
	      NGX_HTTP_LOC_CONF_OFFSET,
	      0,
	      NULL },
	
	      ngx_null_command
	};
```

从代码可以看出：

set_form_input 指令：
可以使用于 http ，server 以及 location 三种上下文中
可以使用1个或者2个参数
指令参数解析函数使用 ngx_http_set_form_input_conf_handler

set_form_input_multi 指令：
可以使用于 http ，server 以及 location 三种上下文中
可以使用1个或者2个参数
指令参数解析函数使用 ngx_http_set_form_input_conf_handler


##指令参数处理

下面看看 ngx_http_set_form_input_conf_handler 的代码摘要：
```c
static char *
ngx_http_set_form_input_conf_handler(ngx_conf_t *cf, ngx_command_t *cmd,
    void *conf)
{
    ndk_set_var_t                            filter;
    ngx_str_t                               *value, s;
    u_char                                  *p;

    ngx_http_form_input_used = 1;

    filter.type = NDK_SET_VAR_MULTI_VALUE;
    filter.size = 1;

    value = cf->args->elts;

    if ((value->len == sizeof("set_form_input_multi") - 1) &&
        ngx_strncmp(value->data, "set_form_input_multi", value->len) == 0)
    {
        dd("use ngx_http_form_input_multi");
        filter.func = (void *) ngx_http_set_form_input_multi;

    } else {
        filter.func = (void *) ngx_http_set_form_input;
    }

    value++;

    if (cf->args->nelts == 2) {
        p = value->data;
        p++;
        s.len = value->len - 1;
        s.data = p;

    } else if (cf->args->nelts == 3) {
        s.len = (value + 1)->len;
        s.data = (value + 1)->data;
    }

    return ndk_set_var_multi_value_core (cf, value,  &s, &filter);
}

```

函数中，关键点在于构建 filter 这个变量，按 指令名称 设置了不同的 filter.func，
并根据指令实参数量（cf->args->nelts），指定了 value 作为变量名，s 作为变量值，并调用 ndk_set_var_multi_value_core，这是一个 ndk 提供的函数，用于简化开发过程中在 http 请求中变量的的设置的流程。

那么 关键的代码就在于 ngx_http_set_form_input_multi 和 ngx_http_set_form_input 了，我们来看看他们的代码摘要：

```c
static ngx_int_t
ngx_http_set_form_input(ngx_http_request_t *r, ngx_str_t *res,
    ngx_http_variable_value_t *v)
{
    ngx_http_form_input_ctx_t           *ctx;
    ngx_int_t                            rc;

    ngx_str_set(res, "");

    ctx = ngx_http_get_module_ctx(r, ngx_http_form_input_module);

    rc = ngx_http_form_input_arg(r, v->data, v->len, res, 0);

    return rc;
}
```
从函数参数列表可以看出，ngx_http_request_t *r 这个参数的出现就预示着 ngx_http_set_form_input 就是在http请求到来的时候执行的函数，
ngx_http_set_form_input 和 ngx_http_set_form_input_multi 函数几乎一样，
都是使用 ngx_http_get_module_ctx 获取请求的上下文，然后调用 ngx_http_form_input_arg，
区别在于 ngx_http_form_input_arg 的最后一个传入的实参的值，我们再看看 ngx_http_form_input_arg 函数的代码摘要：

```c
static ngx_int_t
ngx_http_form_input_arg(ngx_http_request_t *r, u_char *arg_name, size_t arg_len,
    ngx_str_t *value, ngx_flag_t multi)
{
    u_char              *p, *v, *last, *buf;
    ngx_chain_t         *cl;
    size_t               len = 0;
    ngx_array_t         *array = NULL;
    ngx_str_t           *s;
    ngx_buf_t           *b;

    /*
        multi 的真正分支代码，multi 的情况下会创建一个数组存储变量。
    */
    if (multi) {
        array = ngx_array_create(r->pool, 1, sizeof(ngx_str_t));
        if (array == NULL) {
            return NGX_ERROR;
        }
        value->data = (u_char *)array;
        value->len = sizeof(ngx_array_t);

    } else {
        ngx_str_set(value, "");
    }

    /*
        这里从 r->request_body->bufs 中获取 body 内容，并使用 buf 存储起来。
    */
    if (r->request_body->bufs->next != NULL) {
        len = 0;
        for (cl = r->request_body->bufs; cl; cl = cl->next) {
            b = cl->buf;
            len += b->last - b->pos;
        }

        buf = ngx_palloc(r->pool, len);
        if (buf == NULL) {
            return NGX_ERROR;
        }

        p = buf;
        last = p + len;

        for (cl = r->request_body->bufs; cl; cl = cl->next) {
            p = ngx_copy(p, cl->buf->pos, cl->buf->last - cl->buf->pos);
        }
    } else {

        b = r->request_body->bufs->buf;
        if (ngx_buf_size(b) == 0) {
            return NGX_OK;
        }

        buf = b->pos;
        last = b->last;
    }

    /*
        这里对 buf 做解析，把 key 是 arg_name 数据块找到，并把后面的数据块 value 的值复制过来。 
    */
    for (p = buf; p < last; p++) {
        p = ngx_strlcasestrn(p, last - 1, arg_name, arg_len - 1);
        if (p == NULL) {
            return NGX_OK;
        }

        if ((p == buf || *(p - 1) == '&') && *(p + arg_len) == '=') {
            v = p + arg_len + 1;

            p = ngx_strlchr(v, last, '&');
            if (p == NULL) {
                p = last;
            }

            if (multi) {
                s = ngx_array_push(array);
                if (s == NULL) {
                    return NGX_ERROR;
                }
                s->data = v;
                s->len = p - v;

            } else {
                value->data = v;
                value->len = p - v;
                return NGX_OK;
            }
        }
    }

    return NGX_OK;
}
```
请求处理过程中的变量赋值操作就这样完成了，下面我们来看看模块的其他函数。

##模块定义

```c
static ngx_http_module_t ngx_http_form_input_module_ctx = {
    NULL,                                   /* preconfiguration */
    ngx_http_form_input_init,               /* postconfiguration */

    NULL,                                   /* create main configuration */
    NULL,                                   /* init main configuration */

    NULL,                                   /* create server configuration */
    NULL,                                   /* merge server configuration */

    NULL,                                   /* create location configuration */
    NULL                                   /* merge location configuration */
};

ngx_module_t ngx_http_form_input_module = {
    NGX_MODULE_V1,
    &ngx_http_form_input_module_ctx,        /* module context */
    ngx_http_form_input_commands,           /* module directives */
    NGX_HTTP_MODULE,                        /* module type */
    NULL,                                   /* init master */
    NULL,                                   /* init module */
    NULL,                                   /* init process */
    NULL,                                   /* init thread */
    NULL,                                   /* exit thread */
    NULL,                                   /* exit precess */
    NULL,                                   /* exit master */
    NGX_MODULE_V1_PADDING
};

```
从 ngx_http_form_input_module 的定义可以看到，模块是 NGX_HTTP_MODULE 类型，并且定义了配置文件读取完毕后的处理函数 ngx_http_form_input_init。

```c

/* register a new rewrite phase handler */
static ngx_int_t
ngx_http_form_input_init(ngx_conf_t *cf)
{

    ngx_http_handler_pt             *h;
    ngx_http_core_main_conf_t       *cmcf;


    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    h = ngx_array_push(&cmcf->phases[NGX_HTTP_REWRITE_PHASE].handlers);

    if (h == NULL) {
        return NGX_ERROR;
    }

    *h = ngx_http_form_input_handler;

    return NGX_OK;
}
```
ngx_http_form_input_init 向 rewrite 阶段的处理函数列表添加了 ngx_http_form_input_handler。

ngx_http_form_input_handler 函数会在请求到来的时候，在rewrite 阶段调用，代码摘要如下：

```c
static ngx_int_t
ngx_http_form_input_handler(ngx_http_request_t *r)
{
    ngx_http_form_input_ctx_t       *ctx;
    ngx_str_t                        value;
    ngx_int_t                        rc;

    /*
        获取 上下文 于 ctx
    */
    ctx = ngx_http_get_module_ctx(r, ngx_http_form_input_module);

    /*
        只处理 POST 和 PUT 方法的请求
    */
    if (r->method != NGX_HTTP_POST && r->method != NGX_HTTP_PUT) {
        return NGX_DECLINED;
    }

    /*
        协议头没定义 content-type 的请求也不处理。
    */
    if (r->headers_in.content_type == NULL
        || r->headers_in.content_type->value.data == NULL)
    {
        return NGX_DECLINED;
    }

    /*
        协议头 content-type 不是 "application/x-www-form-urlencoded" 的请求也不处理。
    */
    value = r->headers_in.content_type->value;
    if (value.len < form_urlencoded_type_len
        || ngx_strncasecmp(value.data, (u_char *) form_urlencoded_type,
                           form_urlencoded_type_len) != 0)
    {
        return NGX_DECLINED;
    }

    /*
        创建上下文 ngx_http_form_input_ctx_t，并保存于 r 的 ngx_http_form_input_module 块。
    */
    ctx = ngx_pcalloc(r->pool, sizeof(ngx_http_form_input_ctx_t));
    if (ctx == NULL) {
        return NGX_ERROR;
    }
    ngx_http_set_ctx(r, ctx, ngx_http_form_input_module);

    /*
        发送网络读取命令，使用 ngx_http_form_input_post_read 处理。
    */
    rc = ngx_http_read_client_request_body(r, ngx_http_form_input_post_read);
    if (rc == NGX_ERROR || rc >= NGX_HTTP_SPECIAL_RESPONSE) {
        return rc;
    }

    if (rc == NGX_AGAIN) {
        ctx->waiting_more_body = 1;

        return NGX_DONE;
    }


    return NGX_DECLINED;
}
```

这样我们看看  ngx_http_form_input_post_read 干了什么事情。

```c

static void
ngx_http_form_input_post_read(ngx_http_request_t *r)
{
    ngx_http_form_input_ctx_t     *ctx;
    ctx = ngx_http_get_module_ctx(r, ngx_http_form_input_module);

    ctx->done = 1;
    r->main->count--;

    if (ctx->waiting_more_body) {
        ctx->waiting_more_body = 0;

        ngx_http_core_run_phases(r);
    }
}
```
可见这个函数也不做啥，判断有新数据可以读取，即进入这个函数就调用 ngx_http_core_run_phases 继续下一个 本阶段或者下阶段的处理函数。

#总结

模块具体流程如下：
配置文件读取阶段处理两个指令 set_form_input 和 set_form_input_multi，并设置 ngx_http_set_form_input_multi 作为变量函数。
配置文件读取结束阶段往 rewrite 阶段处理函数列表添加了模块自己定义的 ngx_http_form_input_handler。
请求到来阶段，在 rewrite 阶段执行 ngx_http_form_input_handler，用于判断请求是否需要本模块处理。
在 ？？ 阶段，依据 ngx_http_set_form_input 或者 ngx_http_set_form_input_multi 读取 body 内容并解析赋值于指定的 nginx 变量中。







这里解释一下几个基础结构体：

ngx_command_t 是一个指令结构体有如下定义：

```c
	
	typedef struct ngx_command_s     ngx_command_t;
	struct ngx_command_s {
	    ngx_str_t             name;
	    ngx_uint_t            type;
	    char               *(*set)(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);
	    ngx_uint_t            conf;
	    ngx_uint_t            offset;
	    void                 *post;
	};

```
name：是一个字符串，指令名称，可作为唯一标识。
type: 是指令的类型，基本上用于定义指令的格式规范，在程序运行时能够鉴别出指令是否正确使用了。type 大概有如下这些值：
```c

#define NGX_HTTP_MAIN_CONF        0x02000000
#define NGX_HTTP_SRV_CONF         0x04000000
#define NGX_HTTP_LOC_CONF         0x08000000
#define NGX_HTTP_UPS_CONF         0x10000000
#define NGX_HTTP_SIF_CONF         0x20000000
#define NGX_HTTP_LIF_CONF         0x40000000
#define NGX_HTTP_LMT_CONF         0x80000000

#define NGX_CONF_NOARGS      0x00000001
#define NGX_CONF_TAKE1       0x00000002
#define NGX_CONF_TAKE2       0x00000004
#define NGX_CONF_TAKE3       0x00000008
#define NGX_CONF_TAKE4       0x00000010
#define NGX_CONF_TAKE5       0x00000020
#define NGX_CONF_TAKE6       0x00000040
#define NGX_CONF_TAKE7       0x00000080

#define NGX_CONF_MAX_ARGS    8

#define NGX_CONF_TAKE12      (NGX_CONF_TAKE1|NGX_CONF_TAKE2)
#define NGX_CONF_TAKE13      (NGX_CONF_TAKE1|NGX_CONF_TAKE3)

#define NGX_CONF_TAKE23      (NGX_CONF_TAKE2|NGX_CONF_TAKE3)

#define NGX_CONF_TAKE123     (NGX_CONF_TAKE1|NGX_CONF_TAKE2|NGX_CONF_TAKE3)
#define NGX_CONF_TAKE1234    (NGX_CONF_TAKE1|NGX_CONF_TAKE2|NGX_CONF_TAKE3   \
                              |NGX_CONF_TAKE4)

#define NGX_CONF_ARGS_NUMBER 0x000000ff
#define NGX_CONF_BLOCK       0x00000100
#define NGX_CONF_FLAG        0x00000200
#define NGX_CONF_ANY         0x00000400
#define NGX_CONF_1MORE       0x00000800
#define NGX_CONF_2MORE       0x00001000
#define NGX_CONF_MULTI       0x00000000  /* compatibility */

#define NGX_DIRECT_CONF      0x00010000

#define NGX_MAIN_CONF        0x01000000
#define NGX_ANY_CONF         0x0F000000

```
type 可以作与操作同时声明几个值，例如：
	NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_CONF_TAKE12

set： 是一个用于处理指令参数的一个函数指针，在 nginx 源码中已经定义了一些不同类型的指令参数处理函数可以赋值给 set 指针，例如：：

```c
```


