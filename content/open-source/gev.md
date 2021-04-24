---
title: "开源 gev: Go 实现基于 Reactor 模式的非阻塞 TCP 网络库"
date: 2019-09-19T11:22:38+08:00
draft: false
tags: ["Go", "Open Source"]
---

# [gev](https://github.com/Allenxuxu/gev)    轻量、快速的 Golang 网络库

[gev](https://github.com/Allenxuxu/gev)  是一个轻量、快速的基于 Reactor 模式的非阻塞 TCP 网络库，支持自定义协议，轻松快速搭建高性能服务器。

## 为什么有 gev

Golang 的 goroutine 虽然非常轻量，但是每启动一个 goroutine 仍需要 4k 左右的内存。读了鸟窝大佬的文章【[百万 Go TCP 连接的思考: epoll方式减少资源占用](https://colobu.com/2019/02/23/1m-go-tcp-connection/)】后，便去研究了了下 [evio](https://github.com/tidwall/evio)。

evio 虽然非常快，但是仍然存在一些问题，便尝试去优化它，于是有了 [eviop](https://github.com/Allenxuxu/eviop) 项目。关于 evio 的问题可以看我的另一篇博文 【[Golang 网络库evio一些问题/bug和思考](https://hacpai.com/article/1565926947655)】。在优化 evio 完成 eviop 的过程中，因为其网络模型的缘故，愈加感觉修改它非常麻烦，成本比重新搞一个还高。

最终决定自己重搞一个，更加轻量，不需要的全去掉。加上大学时学习过 [muduo](https://github.com/chenshuo/muduo) ，便参考 muduo 的使用的 Reactor 模型实现 gev 。

在 linux 环境下，gev 底层使用 epoll ，这是 gev 会专注优化的地方。在 mac 下底层使用 kqueue，可能不会过多关注这部分的优化，毕竟很少有用 mac 做服务器的（Windows 环境"暂"不支持）。

## 特点

- 基于 epoll 和 kqueue 实现的高性能事件循环
- 支持多核多线程
- 动态扩容 Ring Buffer 实现的读写缓冲区
- 异步读写
- SO_REUSEPORT 端口重用支持
- 支持 WebSocket
- 支持定时任务，延时任务
- 支持自定义协议，处理 TCP 粘包

## 网络模型

`gev` 只使用极少的 goroutine, 一个 goroutine 负责监听客户端连接，其他 goroutine （work 协程）负责处理已连接客户端的读写事件，work 协程数量可以配置，默认与运行主机 CPU 数量相同。

![image.png](https://img.hacpai.com/file/2019/09/image-38c61bae.png)

## 性能测试

> 测试环境 Ubuntu18.04

- gnet   
- eviop   
- evio   
- net (标准库)

### 吞吐量测试

![null](https://raw.githubusercontent.com/Allenxuxu/gev/master/benchmarks/out/gev11.png)

![null](https://raw.githubusercontent.com/Allenxuxu/gev/master/benchmarks/out/gev44.png)

### evio 压测方式:

限制 GOMAXPROCS=1，1 个 work 协程

![image.png](https://img.hacpai.com/file/2019/09/image-c3303366.png)

限制 GOMAXPROCS=1，4 个 work 协程

![image.png](https://img.hacpai.com/file/2019/09/image-6eb2e9a9.png)

限制 GOMAXPROCS=4，4 个 work 协程

![image.png](https://img.hacpai.com/file/2019/09/image-85dbdde8.png)

