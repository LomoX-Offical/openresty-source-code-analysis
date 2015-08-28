# ndk api

## 功能

我们翻查 Openresty 官方说明文档，发现这个 API 非常精妙，他通过 ndk_set_var_value 的调用，可以调用其他模块提供的允许使用变量的指令，具体的实例例如：

```lua
    local res = ndk.set_var.set_escape_uri('a/b');
```

 从上面对功能的描述，我们看 ndk.set_var.set_escape_uri 的功能就是通过 ndk.set_var 这个 table 查找 set_escape_uri 指令，并传入参数 'a/b' 执行这个指令 


