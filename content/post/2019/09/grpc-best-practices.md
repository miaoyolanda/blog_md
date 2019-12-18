---
title: "Grpc Best Practices"
date: 2019-09-28T21:30:01+08:00
keywords: []
draft: true
---

http://beckjin.com/2019/09/25/grpc-docker-swarm/

在 Linux 的内核参数中，有 TCP 的 keepalive 默认设置，时间是 7200s

如果不希望修改内核参数，也可以在 gRPC 服务代码中通过修改 `grpc.keepalive_time_ms`，参考：[Keepalive User Guide for gRPC Core](https://github.com/grpc/grpc/blob/master/doc/keepalive.md#defaults-values) 和 [Grpc_arg_keys](https://grpc.github.io/grpc/core/group__grpc__arg__keys.html)，服务端默认 `grpc.keepalive_time_ms` 也是 7200s，和内核参数一样，以下是 .NET 代码例子（其他语言类似）：

https://www.youtube.com/watch?v=Z_yD7YPL2oE



## RateLimit

根据我有限的观察，在对外部服务进行调用的时候，几乎所有人都会加上重试机制。即使你这边不会，但也不能保证你的调用方不会不断重试，如果这条调用链路上所有的服务都触发了重试，那原来的请求量将会被**指数级别**的放大，这无异于对最底层的服务发起了一次 DDOS 攻击。