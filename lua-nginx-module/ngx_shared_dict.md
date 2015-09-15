# 共享内存字典

##功能

## 指令

lua_shared_dict 这个指令要实现的功能是初始化一块指令指定大小共享内存区域，初始化成一个共享内存字典的数据结构，并绑定指令指定的名字。我们看看官方doc说明的例子:

```nginx
	http {
        lua_shared_dict dogs 10m;
        server {
            location /set {
                content_by_lua '
                    local dogs = ngx.shared.dogs
                    dogs:set("Jim", 8)
                    ngx.say("STORED")
                ';
            }
            location /get {
                content_by_lua '
                    local dogs = ngx.shared.dogs
                    ngx.say(dogs:get("Jim"))
                ';
            }
        }
    }
```

从上面例子可以看出， 
*  指令 lua_shared_dict 初始化了一个名为 dogs 的，大小为 10MB 的共享内存字典
*  在 content_by_lua 命令定义的 Lua 代码中， Lua 代码可以通过访问 ngx.shared.dogs 访问到这个共享内存字典，其实还有另外一种形式 ngx.shared['dogs']

我们对 lua_shared_dict 有了一定的了解之后，我们可以对其代码做一些阅读。

```c
    { ngx_string("lua_shared_dict"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE2,
      ngx_http_lua_shared_dict,
      0,
      0,
      NULL },

```

从上面 lua_shared_dict 指令定义的代码片段可以看出：
*  lua_shared_dict 只能在 http 上下文中出现
*  lua_shared_dict 有且仅有两个参数
*  指令配置解析函数是 ngx_http_lua_shared_dict

那么咱们看看 ngx_http_lua_shared_dict 函数：

```c


char *
ngx_http_lua_shared_dict(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
	// 初始化 shm_zones 数组
    if (lmcf->shm_zones == NULL) {
        lmcf->shm_zones = ngx_palloc(cf->pool, sizeof(ngx_array_t));
        if (ngx_array_init(lmcf->shm_zones, cf->pool, 2,
                           sizeof(ngx_shm_zone_t *))
            != NGX_OK)
        {
            return NGX_CONF_ERROR;
        }
    }

    value = cf->args->elts;
	
	// 共享内存的名字从参数读取
    name = value[1];

	// 内存大小参数读取，不能小于 8k
    size = ngx_parse_size(&value[2]);
    if (size <= 8191) {
        return NGX_CONF_ERROR;
    }

    ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_lua_shdict_ctx_t));

    ctx->name = name;
    ctx->main_conf = lmcf;
    ctx->log = &cf->cycle->new_log;

	// 按参数创建共享内存
    zone = ngx_shared_memory_add(cf, &name, (size_t) size,
                                 &ngx_http_lua_module);
	
	// 设置这块共享内存的初始化函数与参数
    zone->init = ngx_http_lua_shdict_init_zone;
    zone->data = ctx;

	// 把共享内存插入共享内存数组
    zp = ngx_array_push(lmcf->shm_zones);
    *zp = zone;

	// 设置标志位，需要延后执行 init_by_lua 代码的标志
    lmcf->requires_shm = 1;

    return NGX_CONF_OK;
}


```

这里要说明一下的执行顺序，我们在 nginx 的代码上可以见到在 ngx_init_cycle 以及 ngx_conf_handler 函数中一些关键代码的执行顺序：

*  先调用 ngx_conf_parse(&conf, &cycle->conf_file) 函数读取配置文件并执行指令解析，在这里 ngx_http_lua_shared_dict 被执行了。
*  之后在读取配置文件并指令解析结束后，会使用 for 循环遍历 ngx_modules 并且执行每个元素的 preconfiguration ，在这里 ngx_http_lua_init 被执行了。
*  在 ngx_init_cycle 的最后会对 cycle->shared_memory 数组做遍历，并对其各个元素执行 shm_zone[i].init 回调函数，在这里 ngx_http_lua_shdict_init_zone 被执行了。

在这里我们可以看出，流程上 ngx_http_lua_init 比 ngx_http_lua_shdict_init_zone 先执行。但我们知道在 ngx_http_lua_init 上会执行 init_by_lua\* 的 Lua 代码，如果按这样的流程意味着 init_by_lua\* 无法使用 ngx.shared ，因为在这个时间点上， ngx.shared 的初始化函数 ngx_http_lua_shdict_init_zone 还没被执行。

所以这里，作者添加了一个机制，以保证 init_by_lua\* 的 Lua 代码也能使用 ngx.shared ，这就是 init_handler 延后执行：

*  首先指令解析函数 ngx_http_lua_shared_dict 最后解析成功后，添加 lmcf->requires_shm = 1; 标志位。
*  后面 preconfiguration 阶段被调用的 ngx_http_lua_init 函数，在执行之前先判断 requires_shm 是否被标志，下面有 ngx_http_lua_init 代码块。
*  在最后对 cycle->shared_memory 进行遍历并挨个执行 shm_zone[i].init 时， ngx_http_lua_shdict_init_zone 会被调用，该函数中会在最后一个 shared_dict 被执行的最后执行 init_handler ，请看下面 ngx_http_lua_shdict_init_zone 代码块。

```c
// ngx_http_lua_init 判断 requires_shm 被设置为1，就不执行 init_handler 了。
if (!lmcf->requires_shm && lmcf->init_handler) {
	if (lmcf->init_handler(cf->log, lmcf, lmcf->lua) != NGX_OK) {
		/* an error happened */
		return NGX_ERROR;
	}
}

```

```c
ngx_int_t
ngx_http_lua_shdict_init_zone(ngx_shm_zone_t *shm_zone, void *data)
{
    ngx_http_lua_shdict_ctx_t  *octx = data;

    size_t                      len;
    ngx_http_lua_shdict_ctx_t  *ctx;
    ngx_http_lua_main_conf_t   *lmcf;

    ctx = shm_zone->data;

    if (octx) {
        // 如果执行 reload 命令，则会继承已经初始化过的数据
        ctx->sh = octx->sh;
        ctx->shpool = octx->shpool;
        goto done;
    }

    ctx->shpool = (ngx_slab_pool_t *) shm_zone->shm.addr;

    if (shm_zone->shm.exists) {
        // master 会把初始化搞定，子进程执行到这里就会发现已经初始化完成了。
        ctx->sh = ctx->shpool->data;
        goto done;
    }

    // 一般只有 master 才会到达这里，先用共享内存创建共享内存字典的内存分配表结构 ngx_http_lua_shdict_shctx_t 
    ctx->sh = ngx_slab_alloc(ctx->shpool, sizeof(ngx_http_lua_shdict_shctx_t));
    ctx->shpool->data = ctx->sh;

    // 把 内存分配表结构 初始化为 红黑树
    ngx_rbtree_init(&ctx->sh->rbtree, &ctx->sh->sentinel,
                    ngx_http_lua_shdict_rbtree_insert_value);

    // 元素队列初始化
    ngx_queue_init(&ctx->sh->queue);

    // 日志信息初始化
    len = sizeof(" in lua_shared_dict zone \"\"") + shm_zone->shm.name.len;
    ctx->shpool->log_ctx = ngx_slab_alloc(ctx->shpool, len);
    if (ctx->shpool->log_ctx == NULL) {
        return NGX_ERROR;
    }

    ctx->shpool->log_nomem = 0;

done:
    lmcf = ctx->main_conf;
    lmcf->shm_zones_inited++;

    // 判断已经完成最后一个共享内存字典的初始化之后，执行 lmcf->init_handler
    if (lmcf->shm_zones_inited == lmcf->shm_zones->nelts
        && lmcf->init_handler)
    {
        if (lmcf->init_handler(ctx->log, lmcf, lmcf->lua) != NGX_OK) {
            return NGX_ERROR;
        }
    }

    return NGX_OK;
}


```

从函数 ngx_http_lua_shdict_init_zone 可见：
*  共享内存字典实质数据结构是 nginx 本身实现的红黑树。
*  共享内存的内存分配使用 ngx_slab_alloc 系列函数。
*  函数的最后会判断已经完成最后一个共享内存字典的初始化之后，执行 lmcf->init_handler ，以保证 init_by_lua\* 传入的 Lua 代码能在所有共享内存字典初始化之后使用，从而可以使用 ngx.shared apis。

## ngx.shared

我们知道 APIs 的注入动作在 postconfigure 阶段函数执行， 而 ngx.shared 的注册是这样的：

```c
void
ngx_http_lua_inject_shdict_api(ngx_http_lua_main_conf_t *lmcf, lua_State *L)
{
    ngx_http_lua_shdict_ctx_t   *ctx;
    ngx_uint_t                   i;
    ngx_shm_zone_t             **zone;

    if (lmcf->shm_zones != NULL) {
        
        // 一个 table ，取名字 t1
        lua_createtable(L, 0, lmcf->shm_zones->nelts /* nrec */);

        // 创建了一个 shared metatable
        lua_createtable(L, 0 /* narr */, 13 /* nrec */);

        // 往 metatable 插入函数，一个 13  个
        lua_pushcfunction(L, ngx_http_lua_shdict_get);
        lua_setfield(L, -2, "get");

        lua_pushcfunction(L, ngx_http_lua_shdict_get_stale);
        lua_setfield(L, -2, "get_stale");

        lua_pushcfunction(L, ngx_http_lua_shdict_set);
        lua_setfield(L, -2, "set");

        lua_pushcfunction(L, ngx_http_lua_shdict_safe_set);
        lua_setfield(L, -2, "safe_set");

        lua_pushcfunction(L, ngx_http_lua_shdict_add);
        lua_setfield(L, -2, "add");

        lua_pushcfunction(L, ngx_http_lua_shdict_safe_add);
        lua_setfield(L, -2, "safe_add");

        lua_pushcfunction(L, ngx_http_lua_shdict_replace);
        lua_setfield(L, -2, "replace");

        lua_pushcfunction(L, ngx_http_lua_shdict_incr);
        lua_setfield(L, -2, "incr");

        lua_pushcfunction(L, ngx_http_lua_shdict_delete);
        lua_setfield(L, -2, "delete");

        lua_pushcfunction(L, ngx_http_lua_shdict_flush_all);
        lua_setfield(L, -2, "flush_all");

        lua_pushcfunction(L, ngx_http_lua_shdict_flush_expired);
        lua_setfield(L, -2, "flush_expired");

        lua_pushcfunction(L, ngx_http_lua_shdict_get_keys);
        lua_setfield(L, -2, "get_keys");

        // 把这个 metatable 复制一份入栈
        lua_pushvalue(L, -1);
        // 把刚复制的 table 设置进 metatable 的 __index 中
        lua_setfield(L, -2, "__index");

        zone = lmcf->shm_zones->elts;

        // 枚举共享内存字典
        for (i = 0; i < lmcf->shm_zones->nelts; i++) {
            ctx = zone[i]->data;

            // 名字作为 key
            lua_pushlstring(L, (char *) ctx->name.data, ctx->name.len);

            // 指针作为 value
            lua_pushlightuserdata(L, zone[i]);
            
            // 把 metatable 复制一份入栈
            lua_pushvalue(L, -3);
            
            // 把刚复制出来的 table 作为 value 的 metatable
            lua_setmetatable(L, -2);
            
            //  t1[key] = value
            lua_rawset(L, -4);
        }

        // 把 metatable 出栈
        lua_pop(L, 1);

    } else {
        lua_newtable(L); 
    }

    // 外部的 ngx table， ngx["shared"] = t1
    lua_setfield(L, -2, "shared");
}

```

这里的 api 实现并不是直接放在 ngx.shared 上，而是把 shared 作为一个 table ，里边有各种带名字的共享内存字典对象，这些对象存储了一个共享内存数据结构地址，并且继承了 __index 可以索引到 13 个函数 api 。
然后后面对会围绕着这个共享内存数据结构地址 zone[i] 展开进行操作。

# struct

在代码分析重要的一环就是分析数据结构，在 shared dict 共享内存字典的实现中，上下文结构体 ngx_http_lua_shdict_ctx_t 用于维护共享内存以及字典结构，代码如下：


```c
typedef struct {
    ngx_http_lua_shdict_shctx_t  *sh;
    ngx_slab_pool_t              *shpool;
    ngx_str_t                     name;
    ngx_http_lua_main_conf_t     *main_conf;
    ngx_log_t                    *log;
} ngx_http_lua_shdict_ctx_t;

```

在这里的结构成员变量，都在 ngx_http_lua_shdict_init_zone 函数中初始化：

*  ngx_http_lua_shdict_shctx_t *sh 结构是用于维护字典实现数据的结构
*  ngx_slab_pool_t *shpool 是 nginx 提供的 slab 内存分配机制实现的内存池，这里实际操作的是一块共享内存，提供在共享内存中分配内存的功能
*  ngx_str_t name 是字符串类型，用于保存本共享内存字典的名字，作为唯一识别码存在
*  ngx_http_lua_main_conf_t  *main_conf 这里保存了 main 配置的指针，目前只用于判断是否需要延后调用 lmcf->init_handler
*  ngx_log_t *log 写日志数据指针

咱们看看 ngx_http_lua_shdict_shctx_t 结构：

```c
typedef struct {
    ngx_rbtree_t                  rbtree;
    ngx_rbtree_node_t             sentinel;
    ngx_queue_t                   queue;
} ngx_http_lua_shdict_shctx_t;

```

ngx_http_lua_shdict_shctx_t 包含：

*  ngx_rbtree_t  rbtree 一个红黑树结构
*  ngx_rbtree_node_t  sentinel 一个红黑树哨兵节点
*  ngx_queue_t  queue 还有一个队列结构

在这里，包括 ngx_http_lua_shdict_shctx_t 都是从 shpool 分配出来的，里边的红黑树节点，哨兵节点，队列节点都是也都是从 shpool 分配出来。

这里有共享内存字典节点结构的声明：

```c

typedef struct {
    u_char                       color;
    u_char                       dummy;
    u_short                      key_len;
    ngx_queue_t                  queue;
    uint64_t                     expires;
    uint8_t                      value_type;
    uint32_t                     value_len;
    uint32_t                     user_flags;
    u_char                       data[1];
} ngx_http_lua_shdict_node_t;

typedef struct ngx_rbtree_node_s  ngx_rbtree_node_t;

struct ngx_rbtree_node_s {
    ngx_rbtree_key_t       key;
    ngx_rbtree_node_t     *left;
    ngx_rbtree_node_t     *right;
    ngx_rbtree_node_t     *parent;
    u_char                 color;
    u_char                 data;
};

typedef struct ngx_queue_s  ngx_queue_t;

struct ngx_queue_s {
    ngx_queue_t  *prev;
    ngx_queue_t  *next;
};


```

在 nginx 的红黑树使用手法上，这个结构作为用户数据结构而存在，共享内存字典在红黑树结构使用手法基本参考了 ngx_http_limit_req_module 模块的手法，我们来看看下面若干代码片段：

```c
    ngx_rbtree_node_t           *node;
    ngx_http_lua_shdict_node_t  *sd;
    ngx_queue_t                 *q;

    // 1、为节点内存申请，摘自 ngx_http_lua_shdict_set_helper
    n = offsetof(ngx_rbtree_node_t, color)
        + offsetof(ngx_http_lua_shdict_node_t, data)
        + key.len
        + value.len;
    node = ngx_slab_alloc_locked(ctx->shpool, n);

    // 2、从红黑树节点获取 ngx_http_lua_shdict_node_t 节点，摘自 ngx_http_lua_shdict_set_helper
    sd = (ngx_http_lua_shdict_node_t *) &node->color;

    // 3、从 ngx_http_lua_shdict_node_t 节点获取红黑树节点，摘自 ngx_http_lua_shdict_expire
    node = (ngx_rbtree_node_t *)
                ((u_char *) sd - offsetof(ngx_rbtree_node_t, color));


    // 4、从队列节点获取 ngx_http_lua_shdict_node_t 节点，摘自 ngx_http_lua_shdict_expire
    sd = ngx_queue_data(q, ngx_http_lua_shdict_node_t, queue);


```

从上面代码片段可以看出这些结构的包含关系：

*  ngx_rbtree_node_t 结构体实际上包含了 ngx_http_lua_shdict_node_t ，所以分配内存的阶段会多分配 ngx_http_lua_shdict_node_t 的大小，当然没有那么简单，还要去除双方的共同占用的成员变量，有 color 和 data 两个。
*  ngx_http_lua_shdict_node_t 实际上也包含 ngx_queue_t ，这在 ngx_http_lua_shdict_node_t 上就直接有体现了
*  他们之间的结构转换，是直接做地址偏移加减完成的。



# set
ngx.shared.DICT 的其中一个常用的 api 就是 set 了，但除了set ，其实还包括 add ， safe_add ， replace ， safe_set 都在 set api 集合里边。

```c

static int
ngx_http_lua_shdict_add(lua_State *L)
{
    return ngx_http_lua_shdict_set_helper(L, NGX_HTTP_LUA_SHDICT_ADD);
}

static int
ngx_http_lua_shdict_safe_add(lua_State *L)
{
    return ngx_http_lua_shdict_set_helper(L, NGX_HTTP_LUA_SHDICT_ADD
                                          |NGX_HTTP_LUA_SHDICT_SAFE_STORE);
}

static int
ngx_http_lua_shdict_replace(lua_State *L)
{
    return ngx_http_lua_shdict_set_helper(L, NGX_HTTP_LUA_SHDICT_REPLACE);
}

static int
ngx_http_lua_shdict_set(lua_State *L)
{
    return ngx_http_lua_shdict_set_helper(L, 0);
}

static int
ngx_http_lua_shdict_safe_set(lua_State *L)
{
    return ngx_http_lua_shdict_set_helper(L, NGX_HTTP_LUA_SHDICT_SAFE_STORE);
}

```

这里可以看出，这些 api 的 C 函数的实现都调用到 ngx_http_lua_shdict_set_helper 函数，但第二个参数却各不一样。

```c
static int
ngx_http_lua_shdict_set_helper(lua_State *L, int flags)
{
    // 获取栈内有多少数据，判断数量是否有错
    n = lua_gettop(L);
    if (n != 3 && n != 4 && n != 5) {
        return luaL_error(L, "expecting 3, 4 or 5 arguments, "
                          "but only seen %d", n);
    }
    
    // 取得 Lua 虚拟机的栈底数据，在这个环境下他不是第一个传入参数，而是 ngx.shared.DICT 本身，它的 value 其实就是共享内存结构体指针 
    zone = lua_touserdata(L, 1);

    ctx = zone->data;

    // 从 Lua 虚拟机的第 1 个传入参数获取 key
    key.data = (u_char *) luaL_checklstring(L, 2, &key.len);
    // 计算个 hash
    hash = ngx_crc32_short(key.data, key.len);
    
    
    // 获取第 2 个传入参数的类型，根据类型做类型获取
    value_type = lua_type(L, 3);
    switch (value_type) {
    case LUA_TSTRING:
        value.data = (u_char *) lua_tolstring(L, 3, &value.len);
        break;
    case LUA_TNUMBER:
        value.len = sizeof(double);
        num = lua_tonumber(L, 3);
        value.data = (u_char *) &num;
        break;
    case LUA_TBOOLEAN:
        value.len = sizeof(u_char);
        c = lua_toboolean(L, 3) ? 1 : 0;
        value.data = &c;
        break;
    case LUA_TNIL:
        if (flags & (NGX_HTTP_LUA_SHDICT_ADD|NGX_HTTP_LUA_SHDICT_REPLACE)) {
            lua_pushnil(L);
            lua_pushliteral(L, "attempt to add or replace nil values");
            return 2;
        }
        ngx_str_null(&value);
        break;
    default:
        lua_pushnil(L);
        lua_pushliteral(L, "bad value type");
        return 2;
    }

    // 如果有第 4 个参数，取出来作为 exptime 过期时间参数
    if (n >= 4) {
        exptime = luaL_checknumber(L, 4);
        if (exptime < 0) {
            exptime = 0;
        }
    }

    // 如果有第 5 个参数，取出来作为 user_flags 参数
    if (n == 5) {
        user_flags = (uint32_t) luaL_checkinteger(L, 5);
    }

    // 到了操作步骤了，先对这块共享内存加锁
    ngx_shmtx_lock(&ctx->shpool->mutex);

    // 这个函数的功能是找到 1 个已经过期的元素做删除操作
    ngx_http_lua_shdict_expire(ctx, 1);

    // 在共享内存的数据结构内查找 key 为索引的存储空间
    rc = ngx_http_lua_shdict_lookup(zone, hash, key.data, key.len, &sd);
    
    // replace api 情况下
    if (flags & NGX_HTTP_LUA_SHDICT_REPLACE) {
        
        // 没找到或者过期了，解锁，返回值传失败，没找到
        if (rc == NGX_DECLINED || rc == NGX_DONE) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushboolean(L, 0);
            lua_pushliteral(L, "not found");
            lua_pushboolean(L, forcible);
            return 3;
        }

        // 找到了，对这个空间做 value 替换
        goto replace;
    }

    // add 和 safe_add api 情况下
    if (flags & NGX_HTTP_LUA_SHDICT_ADD) {
        // 找到了，解锁，返回值传失败，已存在
        if (rc == NGX_OK) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushboolean(L, 0);
            lua_pushliteral(L, "exists");
            lua_pushboolean(L, forcible);
            return 3;
        }

        // 过期了，做替换
        if (rc == NGX_DONE) {
            goto replace;
        }

        // 没找到，做插入
        goto insert;
    }

    // set 和 safe_set api 的情况下
    if (rc == NGX_OK || rc == NGX_DONE) {
        
        // value 类型是 null 情况下，做删除
        if (value_type == LUA_TNIL) {
            goto remove;
        }
        
        // 这里代码被精心设计了，不同的条件进入会有不同的代码路径
replace:
        // replace 的第一步，如果 value 大小不变，修改 value 即可
        if (value.data && value.len == (size_t) sd->value_len) {

            // 队列里把找到的元素给删除掉，并把元素从队列头插入
            ngx_queue_remove(&sd->queue);
            ngx_queue_insert_head(&ctx->sh->queue, &sd->queue);

            // 设置 sd 的数据值
            sd->key_len = (u_short) key.len;
            if (exptime > 0) {
                tp = ngx_timeofday();
                sd->expires = (uint64_t) tp->sec * 1000 + tp->msec
                              + (uint64_t) (exptime * 1000);

            } else {
                sd->expires = 0;
            }

            sd->user_flags = user_flags;
            sd->value_len = (uint32_t) value.len;
            sd->value_type = (uint8_t) value_type;

            p = ngx_copy(sd->data, key.data, key.len);
            ngx_memcpy(p, value.data, value.len);

            // 解锁
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushboolean(L, 1);
            lua_pushnil(L);
            lua_pushboolean(L, forcible);
            return 3;
        }

remove:
        // 第二步，队列里把找到的元素给删除掉
        ngx_queue_remove(&sd->queue);

        node = (ngx_rbtree_node_t *)
                   ((u_char *) sd - offsetof(ngx_rbtree_node_t, color));

        // 红黑树上也把元素给删除掉
        ngx_rbtree_delete(&ctx->sh->rbtree, node);

        // 最后还把元素占用的内存释放掉
        ngx_slab_free_locked(ctx->shpool, node);

    }

insert:

    // 第三步，判断为空，即判断操作为删除，直接返回并退出即可
    if (value.data == NULL) {
        ngx_shmtx_unlock(&ctx->shpool->mutex);

        lua_pushboolean(L, 1);
        lua_pushnil(L);
        lua_pushboolean(L, 0);
        return 3;
    }

    // 计算新增节点所需空间大小，并按空间大小给节点分配内存
    n = offsetof(ngx_rbtree_node_t, color)
        + offsetof(ngx_http_lua_shdict_node_t, data)
        + key.len
        + value.len;
    node = ngx_slab_alloc_locked(ctx->shpool, n);

    // 分配失败，空间不足
    if (node == NULL) {
        
        if (flags & NGX_HTTP_LUA_SHDICT_SAFE_STORE) {
            // 如果是 safe 系列 api，需要保护已有数据，返回内存不足
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushboolean(L, 0);
            lua_pushliteral(L, "no memory");
            return 2;
        }

        // 尝试 30 次对字典中的过期元素进行内存回收
        for (i = 0; i < 30; i++) {

            if (ngx_http_lua_shdict_expire(ctx, 0) == 0) {
                break;
            }

            forcible = 1;

            node = ngx_slab_alloc_locked(ctx->shpool, n);
            if (node != NULL) {
                goto allocated;
            }
        }

        // 尝试失败，还是没内存，认命了返回内存不足
        ngx_shmtx_unlock(&ctx->shpool->mutex);

        lua_pushboolean(L, 0);
        lua_pushliteral(L, "no memory");
        lua_pushboolean(L, forcible);
        return 3;
    }

allocated:
    // 第四步，分配好内存之后，获取 sd
    sd = (ngx_http_lua_shdict_node_t *) &node->color;
    
    node->key = hash;
    
    // 设置 sd 的数据值
    sd->key_len = (u_short) key.len;
    if (exptime > 0) {
        tp = ngx_timeofday();
        sd->expires = (uint64_t) tp->sec * 1000 + tp->msec
                      + (uint64_t) (exptime * 1000);
    } else {
        sd->expires = 0;
    }

    sd->user_flags = user_flags;
    sd->value_len = (uint32_t) value.len;
    sd->value_type = (uint8_t) value_type;

    p = ngx_copy(sd->data, key.data, key.len);
    ngx_memcpy(p, value.data, value.len);

    // 把节点插入红黑树中
    ngx_rbtree_insert(&ctx->sh->rbtree, node);
    
    // 把节点插入队列中
    ngx_queue_insert_head(&ctx->sh->queue, &sd->queue);

    // 解锁并返回成功
    ngx_shmtx_unlock(&ctx->shpool->mutex);

    lua_pushboolean(L, 1);
    lua_pushnil(L);
    lua_pushboolean(L, forcible);
    return 3;
}

```
代码注释可能并没有很好的表达里边的逻辑，特别是加了若干 goto 跳转之后更是理解起来比较费劲。
这里我们分别对 add ， replace ， set 三个 api 做一下关键逻辑梳理，我们先看看 set 系列 api：

*  传入参数的数量和类型检查
*  调用 ngx_http_lua_shdict_lookup 查找节点
*  找到节点，比较 value 大小是否一样，如果一样直接替换value，返回成功
*  如果找不到节点或者 value 大小不一致，删除节点
*  重新分配节点，如果分配失败，返回失败
*  对节点进行插入，返回插入结果

set 函数是关键路径走得最完整的，我们再看看 add ：

*  传入参数的数量和类型检查
*  调用 ngx_http_lua_shdict_lookup 查找节点
*  找到节点，返回失败
*  如果节点过期，比较 value 大小是否一样，如果一样直接替换value，返回成功
*  如果找不到节点或者节点过期并且大小不一致，删除节点
*  重新分配节点，如果分配失败，返回失败
*  对节点进行插入，返回插入结果

从上述可见与 set 相比， add 的执行路径是不一样的特别在于找到节点后的处理方面，我们再看看 replace ：

*  传入参数的数量和类型检查
*  调用 ngx_http_lua_shdict_lookup 查找节点
*  如果找不到节点或节点过期，返回失败
*  如果找到节点，比较 value 大小是否一样，如果一样直接替换value，返回成功
*  如果大小不一致，删除节点
*  重新分配节点，如果分配失败，返回失败
*  对节点进行插入，返回插入结果

可以看出 replace 与 add 的找到节点之后的处理方法刚好相反，这可以说就是他们要实现的功能的区别所在。

## get

ngx.shared.DICT 的另一个常用的 api 就是 get 和 get_stale api 了，他的作用是从共享内存字典中取出指定 key 所对应 value ，其功能入上文所述，是对应上一个 C 函数 ngx_http_lua_shdict_get ，代码如下：

```c
static int
ngx_http_lua_shdict_get(lua_State *L)
{
  return ngx_http_lua_shdict_get_helper(L, 0 /* stale */);
}

static int
ngx_http_lua_shdict_get_stale(lua_State *L)
{
  return ngx_http_lua_shdict_get_helper(L, 1 /* stale */);
}

static int
ngx_http_lua_shdict_get_helper(lua_State *L, int get_stale)
{
    n = lua_gettop(L);

    // 参数仅能是2个
    if (n != 2) {
        return luaL_error(L, "expecting exactly two arguments, "
                          "but only seen %d", n);
    }

    // zone 把 ngx.shared.DICT 的唯一元素给取出来了，就是共享内存结构指针
    zone = lua_touserdata(L, 1);
    ctx = zone->data;
    name = ctx->name;

    // 检查参数 2 ，并赋值给 key
    key.data = (u_char *) luaL_checklstring(L, 2, &key.len);

    // 这里算了一次 hash ，但这里只是为了提高红黑树的查找速度
    hash = ngx_crc32_short(key.data, key.len);

    //  获取共享内存互斥锁
    ngx_shmtx_lock(&ctx->shpool->mutex);

    // 判断一下 过期了没
    if (!get_stale) {
        ngx_http_lua_shdict_expire(ctx, 1);
    }

    // 这里是查找算法
    rc = ngx_http_lua_shdict_lookup(zone, hash, key.data, key.len, &sd);

    // 找不到返回 null
    if (rc == NGX_DECLINED || (rc == NGX_DONE && !get_stale)) {
        ngx_shmtx_unlock(&ctx->shpool->mutex);
        lua_pushnil(L);
        return 1;
    }

    // 结果赋值到 value
    value_type = sd->value_type;
    value.data = sd->data + sd->key_len;
    value.len = (size_t) sd->value_len;

    // 结果类型处理部分，还要做各种错误判断，这里省略一些了。
    switch (value_type) {
    case LUA_TSTRING:
        lua_pushlstring(L, (char *) value.data, value.len);
        break;
    case LUA_TNUMBER:
        ngx_memcpy(&num, value.data, sizeof(double));
        lua_pushnumber(L, num);
        break;
    case LUA_TBOOLEAN:
        c = *value.data;
        lua_pushboolean(L, c ? 1 : 0);
        break;
    }

    user_flags = sd->user_flags;

    // 解锁， 返回查找结果
    ngx_shmtx_unlock(&ctx->shpool->mutex);

    if (get_stale) {
        if (user_flags) {
            lua_pushinteger(L, (lua_Integer) user_flags);

        } else {
            lua_pushnil(L);
        }

        lua_pushboolean(L, rc == NGX_DONE);
        return 3;
    }

    if (user_flags) {
        lua_pushinteger(L, (lua_Integer) user_flags);
        return 2;
    }

    return 1;
}

```

## 红黑树操作相关函数

我们看看 ngx_http_lua_shdict_expire 这个函数，他的主要功能就是把 n 个已经过期的内存释放掉，降低内存使用：

```c
static int
ngx_http_lua_shdict_expire(ngx_http_lua_shdict_ctx_t *ctx, ngx_uint_t n)
{
    // 获取当前时间
    tp = ngx_timeofday();
    now = (uint64_t) tp->sec * 1000 + tp->msec;

    /*
     * n == 1 删除一个或者两个过期元素
     * n == 0 强制删除一个最老的元素，
     *        然后删除一个或者两个过期元素
     */

    while (n < 3) {
        
        // 队列中无元素，不做任何事情退出
        if (ngx_queue_empty(&ctx->sh->queue)) {
            return freed;
        }

        // 取下一个队列中的字典元素 sd
        q = ngx_queue_last(&ctx->sh->queue);
        sd = ngx_queue_data(q, ngx_http_lua_shdict_node_t, queue);

        // 如果 n = 0 则忽略有效期逻辑，直接进入删除步骤，实现强制删除逻辑
        if (n++ != 0) {
            
            // 如果 sd 没有设置有效期，直接退出
            if (sd->expires == 0) {
                return freed;
            }

            // 如果有效期没过，直接退出
            ms = sd->expires - now;
            if (ms > 0) {
                return freed;
            }
        }

        // 的确过期了，队列中删除此元素，红黑树也删除此元素，并且占用的内存空间归还给 slab
        ngx_queue_remove(q);
        node = (ngx_rbtree_node_t *)
                   ((u_char *) sd - offsetof(ngx_rbtree_node_t, color));
        ngx_rbtree_delete(&ctx->sh->rbtree, node);

        ngx_slab_free_locked(ctx->shpool, node);

        freed++;
    }

    return freed;
}
```



这个函数的实现很大程度上参考了 ngx_http_limit_req_module 模块中的 ngx_http_limit_req_expire 函数实现，流程基本一样，除了有效期各自模块有各自的定义，所以它们的实现也有所区别，其具体区别就是共享内存字典需要对 expires 为 0 的情况，也就是没设置有效期的情况做跳过流程的处理。

在这里，聪明的读者能理解队列的结尾为什么是有效时间最老的元素呢？

好吧，聪明的你应该不用我多解释，那么下面我们看看 ngx_http_lua_shdict_lookup 函数吧：

```c

static ngx_int_t
ngx_http_lua_shdict_lookup(ngx_shm_zone_t *shm_zone, ngx_uint_t hash,
    u_char *kdata, size_t klen, ngx_http_lua_shdict_node_t **sdp)
{
    // 取共享内存字典的上下文，红黑树根节点和哨兵节点
    ctx = shm_zone->data;
    node = ctx->sh->rbtree.root;
    sentinel = ctx->sh->rbtree.sentinel;

    while (node != sentinel) {
        
        // 按 hash 比较大小找到相等节点
        if (hash < node->key) {
            node = node->left;
            continue;
        }

        if (hash > node->key) {
            node = node->right;
            continue;
        }

        // 相等的找到了，结构体小技巧，把 sd 从 node 中取出来
        sd = (ngx_http_lua_shdict_node_t *) &node->color;

        // hash 相等不代表 key 一样啊，这时候还得判断一下 key 是否相等
        rc = ngx_memn2cmp(kdata, sd->data, klen, (size_t) sd->key_len);

        if (rc == 0) {
            // key 也一样的情况下，从队列中取出，再重新插入到队列头中，这个是重新计算有效期的意思
            ngx_queue_remove(&sd->queue);
            ngx_queue_insert_head(&ctx->sh->queue, &sd->queue);

            // 设置可以返回的 sdp
            *sdp = sd;

            // 计算有效期，并返回相应结果
            if (sd->expires != 0) {
                tp = ngx_timeofday();
                now = (uint64_t) tp->sec * 1000 + tp->msec;
                ms = sd->expires - now;
                if (ms < 0) {
                    return NGX_DONE;
                }
            }

            return NGX_OK;
        }

        // key 比较结果不相等的情况下，根据结果往左右节点继续找
        node = (rc < 0) ? node->left : node->right;
    }

    // 哨兵节点，直接返回 null ，找不到
    *sdp = NULL;
    return NGX_DECLINED;
}

```
这个函数要实现的功能就是查找到 key 所在节点，其实现与 ngx_http_limit_req_module 模块中的 ngx_http_limit_req_lookup 有点相似。


在 nginx 代码中，ngx_rbtree.c 只实现了 rbtree 的插入删除等操作，并没有实现查找操作，都是各个使用 rbtree 的模块自己实现的，其典型实现是在 ngx_http_limit_req_module 中，它其中的函数 ngx_http_limit_conn_lookup 实现了查找操作，但也有其模块本身的一些特殊逻辑在其中，而 ngx_http_lua_shdict_lookup 的基本逻辑是与 ngx_http_limit_conn_lookup 类似的，又有共享内存字典特有的有效期逻辑在其中：


然后，在插入流程中， ngx_rbtree_t 的插入函数可以做部分定制，这是通过实现 insert 回调函数指针实现的，在 ngx_rbtree_insert 插入操作函数流程中会多次使用自定义 insert 回调函数，已完成插入操作，并实现高度灵活定制。但其实nginx 还是实现了 insert 回调函数的默认定义 ngx_rbtree_insert_value ，各个 nginx 模块除了可以使用默认 insert 回调函数之外，也可以自己实现定制的 insert 回调。

在共享内存字典的实现中，没有使用默认插入函数 ngx_rbtree_insert_value ，而是自己实现了 ngx_http_lua_shdict_rbtree_insert_value ，我们先回头看看 ngx_http_lua_shdict_init_zone 函数对其使用的 rbtree 的定义：

```c
#define ngx_rbtree_init(tree, s, i)                                           \
    ngx_rbtree_sentinel_init(s);                                              \
    (tree)->root = s;                                                         \
    (tree)->sentinel = s;                                                     \
    (tree)->insert = i


    ngx_rbtree_init(&ctx->sh->rbtree, &ctx->sh->sentinel,
                    ngx_http_lua_shdict_rbtree_insert_value);

```

在这个代码片段中，我们可以看出 rbtree 的 insert 回调函数指针被定义为 ngx_http_lua_shdict_rbtree_insert_value ， 这意味着插入操作里边会使用 ngx_http_lua_shdict_rbtree_insert_value 做插入操作基础，在插入完毕之后红黑树会做标准的平衡操作。

```c
void
ngx_http_lua_shdict_rbtree_insert_value(ngx_rbtree_node_t *temp,
    ngx_rbtree_node_t *node, ngx_rbtree_node_t *sentinel)
{
    // 在 temp 节点上插入 node ， sentinel 是哨兵节点
    for ( ;; ) {
        // 通过对 hash 的比较，决定往左右查找
        if (node->key < temp->key) {
            p = &temp->left;
        } else if (node->key > temp->key) {
            p = &temp->right;
        } else { 
            // 比较 hash 发现 hash 已经存在节点，按 key 比较结果再决定左右节点查找
            sdn = (ngx_http_lua_shdict_node_t *) &node->color;
            sdnt = (ngx_http_lua_shdict_node_t *) &temp->color;

            p = ngx_memn2cmp(sdn->data, sdnt->data, sdn->key_len,
                             sdnt->key_len) < 0 ? &temp->left : &temp->right;
        }

        // 直到找到的节点等于哨兵节点，也就是这个节点不存在退出查找循环。
        if (*p == sentinel) {
            break;
        }

        temp = *p;
    }

    // 把节点插入到 temp 节点中，并初始化其父节点左右节点
    *p = node;
    node->parent = temp;
    node->left = sentinel;
    node->right = sentinel;
    ngx_rbt_red(node);
}
```
与 ngx_rbtree_insert_value 比较，差别在于对 hash 相等情况下的处理，在 ngx_rbtree_insert_value 函数中相等情况下会统一向右节点查找，而 ngx_http_lua_shdict_rbtree_insert_value 函数中，hash 相等情况下会用 key 原来数据做比较并根据其结果决定向左右查找，而这个实现其实与 ngx_http_limit_req_rbtree_insert_value 函数实现基本一致。

## get_keys

这里还实现了一个获取所有 key 的 api ，不过需要注意的是这个 api 的性能消耗，作者在函数开头的注释中写道，这个 api 用时间换空间，有可能比较慢，最坏情况 O(2n)

```c

/*
 * This trades CPU for memory. This is potentially slow. O(2n)
 */

static int
ngx_http_lua_shdict_get_keys(lua_State *L)
{
    // 照例，判断函数参数数量
    n = lua_gettop(L);

    // this 应该是一个轻量用户数据
    luaL_checktype(L, 1, LUA_TLIGHTUSERDATA);

    // 把 zone 从 this 取出
    zone = lua_touserdata(L, 1);

    // 第一个参数， max_count 默认 1024
    if (n == 2) {
        attempts = luaL_checkint(L, 2);
    }

    // 对共享内存加锁
    ctx = zone->data;
    ngx_shmtx_lock(&ctx->shpool->mutex);

    // 获取当前时间
    tp = ngx_timeofday();
    now = (uint64_t) tp->sec * 1000 + tp->msec;

    /* first run through: get total number of elements we need to allocate */

    // 第一个 O(n) 枚举队列节点，统计 max_count 数量的未过期节点，得到 total 应该输出的节点数量
    q = ngx_queue_last(&ctx->sh->queue);
    while (q != ngx_queue_sentinel(&ctx->sh->queue)) {
        prev = ngx_queue_prev(q);

        sd = ngx_queue_data(q, ngx_http_lua_shdict_node_t, queue);

        if (sd->expires == 0 || sd->expires > now) {
            total++;
            if (attempts && total == attempts) {
                break;
            }
        }

        q = prev;
    }

    // 利用 total 事先创建一个大小恰当的输出 table
    lua_createtable(L, total, 0);

    // 第二个 O(n) 枚举队列节点，遍历 max_count 数量的未过期节点，并把 key 作为字符串插入 table
    total = 0;
    q = ngx_queue_last(&ctx->sh->queue);

    while (q != ngx_queue_sentinel(&ctx->sh->queue)) {
        prev = ngx_queue_prev(q);

        sd = ngx_queue_data(q, ngx_http_lua_shdict_node_t, queue);

        if (sd->expires == 0 || sd->expires > now) {
            lua_pushlstring(L, (char *) sd->data, sd->key_len);
            lua_rawseti(L, -2, ++total);
            if (attempts && total == attempts) {
                break;
            }
        }

        q = prev;
    }

    // 解锁共享内存，并退出
    ngx_shmtx_unlock(&ctx->shpool->mutex);
    return 1;
}

```

由此可见， ngx_http_lua_shdict_get_keys 为了确定输出 table 的元素数量上，多做了一次遍历，输出的 table 因此有了确定的元素数量而不会自动扩展内存，这就是作者注释当中描述的时间换空间的技术细节了，由此可见作者在技术偏好上，更倾向于使用更小的内存完成这个操作，对 CPU 消耗会更多，可见这个 API 的使用场景在数据量不大的场景下使用会比较合适。

共享内存字典的 API 其实还有 flush_all ， delete ， flush_expired ， incr 等，这些 API 相对简单，而且基本流程在上述的 API 中都有描述，这里就不一一做解析了。


## 总结
从 set 和 get 等系列的代码分析可以看出：
*  由于使用了共享内存，所以内存分配用了 slab 内存分配机制。
*  由于要实现字典功能，所以内存的组织结构使用了红黑树作为底层实现。
*  但除了红黑树，节点其实还需要插入到一个队列中，这是未解之谜 1 。
*  红黑树使用的 key 是一个整数，在共享内存字典的索引字符串先被 CRC 算法计算一次得到 hash 然后作为红黑树的索引 key 使用，这个使用导致其实 红黑树对原本的索引字符串而言并没有排序功能，不能实现条件范围搜索的功能。
*  除了使用了红黑树作为基础，为了实现有效期功能，添加了有效期相关的 ngx_http_lua_shdict_expire 、 ngx_http_lua_shdict_rbtree_insert_value 以及 ngx_http_lua_shdict_lookup 等函数。
*  在 get_keys api 中使用了两次遍历循环，以便时间换空间，在数据量小的场景使用比较合适。

从实现上不难看出，共享内存字典的实现很大程度上参考了 ngx_http_limit_req_module 对红黑树使用的代码实现，事实上这两处功能其实基本相同，作者参考成熟度较高的 ngx_http_limit_req_module 模块代码也为其本身成熟度奠定了良好的基础。
