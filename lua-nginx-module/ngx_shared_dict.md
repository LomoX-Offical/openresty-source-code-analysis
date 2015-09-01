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

# set
ngx.shared.DICT 的其中一个常用的 api 就是 set 了，但除了set api ，其实还包括 add ， safe_add ， replace ， add ， add ， 

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


static int
ngx_http_lua_shdict_set_helper(lua_State *L, int flags)
{
    int                          i, n;
    ngx_str_t                    key;
    uint32_t                     hash;
    ngx_int_t                    rc;
    ngx_http_lua_shdict_ctx_t   *ctx;
    ngx_http_lua_shdict_node_t  *sd;
    ngx_str_t                    value;
    int                          value_type;
    double                       num;
    u_char                       c;
    lua_Number                   exptime = 0;
    u_char                      *p;
    ngx_rbtree_node_t           *node;
    ngx_time_t                  *tp;
    ngx_shm_zone_t              *zone;
    int                          forcible = 0;
                         /* indicates whether to foricibly override other
                          * valid entries */
    int32_t                      user_flags = 0;

    n = lua_gettop(L);

    if (n != 3 && n != 4 && n != 5) {
        return luaL_error(L, "expecting 3, 4 or 5 arguments, "
                          "but only seen %d", n);
    }

    zone = lua_touserdata(L, 1);
    if (zone == NULL) {
        return luaL_error(L, "bad \"zone\" argument");
    }

    ctx = zone->data;

    if (lua_isnil(L, 2)) {
        lua_pushnil(L);
        lua_pushliteral(L, "nil key");
        return 2;
    }

    key.data = (u_char *) luaL_checklstring(L, 2, &key.len);

    if (key.len == 0) {
        lua_pushnil(L);
        lua_pushliteral(L, "empty key");
        return 2;
    }

    if (key.len > 65535) {
        lua_pushnil(L);
        lua_pushliteral(L, "key too long");
        return 2;
    }

    hash = ngx_crc32_short(key.data, key.len);

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

    if (n >= 4) {
        exptime = luaL_checknumber(L, 4);
        if (exptime < 0) {
            exptime = 0;
        }
    }

    if (n == 5) {
        user_flags = (uint32_t) luaL_checkinteger(L, 5);
    }

    ngx_shmtx_lock(&ctx->shpool->mutex);

#if 1
    ngx_http_lua_shdict_expire(ctx, 1);
#endif

    rc = ngx_http_lua_shdict_lookup(zone, hash, key.data, key.len, &sd);

    dd("shdict lookup returned %d", (int) rc);

    if (flags & NGX_HTTP_LUA_SHDICT_REPLACE) {

        if (rc == NGX_DECLINED || rc == NGX_DONE) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushboolean(L, 0);
            lua_pushliteral(L, "not found");
            lua_pushboolean(L, forcible);
            return 3;
        }

        /* rc == NGX_OK */

        goto replace;
    }

    if (flags & NGX_HTTP_LUA_SHDICT_ADD) {

        if (rc == NGX_OK) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushboolean(L, 0);
            lua_pushliteral(L, "exists");
            lua_pushboolean(L, forcible);
            return 3;
        }

        if (rc == NGX_DONE) {
            /* exists but expired */

            dd("go to replace");
            goto replace;
        }

        /* rc == NGX_DECLINED */

        dd("go to insert");
        goto insert;
    }

    if (rc == NGX_OK || rc == NGX_DONE) {

        if (value_type == LUA_TNIL) {
            goto remove;
        }

replace:

        if (value.data && value.len == (size_t) sd->value_len) {

            ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                           "lua shared dict set: found old entry and value "
                           "size matched, reusing it");

            ngx_queue_remove(&sd->queue);
            ngx_queue_insert_head(&ctx->sh->queue, &sd->queue);

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

            dd("setting value type to %d", value_type);

            sd->value_type = (uint8_t) value_type;

            p = ngx_copy(sd->data, key.data, key.len);
            ngx_memcpy(p, value.data, value.len);

            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushboolean(L, 1);
            lua_pushnil(L);
            lua_pushboolean(L, forcible);
            return 3;
        }

        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                       "lua shared dict set: found old entry bug value size "
                       "NOT matched, removing it first");

remove:

        ngx_queue_remove(&sd->queue);

        node = (ngx_rbtree_node_t *)
                   ((u_char *) sd - offsetof(ngx_rbtree_node_t, color));

        ngx_rbtree_delete(&ctx->sh->rbtree, node);

        ngx_slab_free_locked(ctx->shpool, node);

    }

insert:

    /* rc == NGX_DECLINED or value size unmatch */

    if (value.data == NULL) {
        ngx_shmtx_unlock(&ctx->shpool->mutex);

        lua_pushboolean(L, 1);
        lua_pushnil(L);
        lua_pushboolean(L, 0);
        return 3;
    }

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                   "lua shared dict set: creating a new entry");

    n = offsetof(ngx_rbtree_node_t, color)
        + offsetof(ngx_http_lua_shdict_node_t, data)
        + key.len
        + value.len;

    node = ngx_slab_alloc_locked(ctx->shpool, n);

    if (node == NULL) {

        if (flags & NGX_HTTP_LUA_SHDICT_SAFE_STORE) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushboolean(L, 0);
            lua_pushliteral(L, "no memory");
            return 2;
        }

        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                       "lua shared dict set: overriding non-expired items "
                       "due to memory shortage for entry \"%V\"", &key);

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

        ngx_shmtx_unlock(&ctx->shpool->mutex);

        lua_pushboolean(L, 0);
        lua_pushliteral(L, "no memory");
        lua_pushboolean(L, forcible);
        return 3;
    }

allocated:

    sd = (ngx_http_lua_shdict_node_t *) &node->color;

    node->key = hash;
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

    dd("setting value type to %d", value_type);

    sd->value_type = (uint8_t) value_type;

    p = ngx_copy(sd->data, key.data, key.len);
    ngx_memcpy(p, value.data, value.len);

    ngx_rbtree_insert(&ctx->sh->rbtree, node);

    ngx_queue_insert_head(&ctx->sh->queue, &sd->queue);

    ngx_shmtx_unlock(&ctx->shpool->mutex);

    lua_pushboolean(L, 1);
    lua_pushnil(L);
    lua_pushboolean(L, forcible);
    return 3;
}

```

## get

ngx.shared.DICT 的另一个常用的 api 就是 get 了，他的作用是从共享内存字典中取出指定 key 所对应 value ，其功能入上文所述，是对应上一个 C 函数 ngx_http_lua_shdict_get ，代码如下：

```c
static int
ngx_http_lua_shdict_get(lua_State *L)
{
  return ngx_http_lua_shdict_get_helper(L, 0 /* stale */);
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




*  有 lua_shared_dict 这条指令的存在，预示着共享内存字典的初始化可以发生在配置文件读取阶段。
*  应该不存在 