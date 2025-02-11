---
date: "2025-02-07"
draft: false
title: "Redis 网络模型"
cover:
  image: img/redis-network-model.png
  alt: "This is a post image"
  caption: "Redis Network Model"
---

# Redis 网络模型

https://www.sobyte.net/post/2022-03/redis-multi-threaded-network-model/

# 1 Redis 网络模型

# 2 Redis 6.0 启用了多线程

通过对 Redis 的网络模型进行分析我们可以知道，Redis 网络模型的性能瓶颈在于（1）对客户端 socket 的 IO 解析；（2）对客户端 socket 的 IO 回写数据。 瓶颈都是出现在 IO 上
因此，Redis 针对（1）客户端命令解析；（2）写响应结果这两个环节采用了多线程进行并发的处理
过程是这样的

1. 在 epoll_wait 的过程中遍历得到待处理的客户端 socket
2. 一个客户端 socket 在一次请求中可能会传递多个命令
3. 多线程分配一个线程从这个客户端 socket 请求中解析命令，这一个特定线程解析完所有命令后会将结果依次添加到一个全局队列中
4. Redis 主线程从全局队列中依次执行，并且将结果写到 client 的响应缓冲区中
5. Redis 在下一次 epoll_wait 循环中监听到客户端 socket 的写事件就绪，再通过多线程的方式，每个 socket 分配一个线程进行串行的回写数据
   需要注意的是，Redis 开启多线程后不保证多个客户端之间命令的串行性，只保证单个客户端内部命令执行以及响应的串行性（通过为一个客户端分配一个单独的线程串行执行来完成），这种设计满足大部分场景的要求，因为客户端通常只关注自身操作的原子性。

