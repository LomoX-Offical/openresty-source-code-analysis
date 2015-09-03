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

## 总结
从 set 和 get 系列的代码分析可以看出：
*  由于使用了共享内存，所以内存分配用了 slab 内存分配机制。
*  由于要实现字典功能，所以内存的组织结构使用了红黑树作为底层实现。
*  但除了红黑树，节点其实还需要插入到一个队列中，这是未解之谜 1 。
*  除了使用了红黑树作为基础，为了实现有效期功能，添加了 ngx_http_lua_shdict_expire 以及 ngx_http_lua_shdict_lookup 等函数。


*  有 lua_shared_dict 这条指令的存在，预示着共享内存字典的初始化可以发生在配置文件读取阶段。
*  应该不存在 