# Nginx Struct Usage

## 概述

在前文中，我们了解到 lua-nginx-module 是如何把 Lua 代码植入到 nginx 的每个请求处理阶段中， Lua 代码块在运行结束后，nignx 根据其返回值进行逻辑处理，决定要读取内容，还是关闭请求，还是让另一个 handler 处理。
然而，我们使用 Lua 编写的算法，需要根据输入参数的来计算出返回值的，那么 Lua 代码需要用到的输入数据有哪些数据呢：
*  request 请求结构体
*   