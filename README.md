OpenResty源码解析
=======

OpenResty 在我们公司有了3-4年的测试，由于我们的业务是私有云方案，所以用户基本是自己搭建服务器，我们提供软件包部署到用户服务器上，所以我们已经有数以万计的服务器部署 OpenResty 的经验。

OpenResty 最大特点是继承了 Nginx 的异步特性，加以 Lua-Jit 使得 服务端开发开发效率大幅提高。

本书把 OpenResty 是 结合 Lua-jit 并且实现异步的特点做了代码级的分析解读，希望借此对大家深入理解 OpenResty 提供帮助。 

本书源码在 Github 上维护，欢迎参与：
[github地址](https://github.com/LomoX-Offical/openresty-source-code-analysis)。

