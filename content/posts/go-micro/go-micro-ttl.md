---
title: "Go Micro 服务健康检查"
date: 2019-06-27T11:22:38+08:00
draft: false
tags: ["Go-Micro"]
---

# 服务健康检查

在微服务架构中，每个服务都会存在多个实例，可能部署在不同的主机中。因为网络或者主机等不确定因素，每个服务都可能会出现故障。我们需要能够监控每个服务实例的健康状态，当一个服务故障时，及时将它从注册中心删除。

# 实现

micro提供两个方法可以直接实现健康检查功能

```
micro.RegisterTTL(time.Second*30),
micro.RegisterInterval(time.Second*20),
```

Interval就是间隔多久服务会重新注册
TTL就是注册服务的过期时间，如果服务挂了，超过过期时间后，注册中心也会将服务删除

## micro内部服务注册的流程

当我们执行service.Run() 时内部会执行Start()
![image-20210424112941936](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/image-20210424112941936.png)

在Start函数中又会执行s.opts.Server.Start()，方法的实现在go-micro/server/rpc_server.go中。
我们跳转到内部server的Start方法
![image-20210424113002309](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/image-20210424113002309.png)

可以发现micro使用一个定时器按照间隔时间去自动重新注册。当服务意外故障，无法向注册中心重新注册时，如果超过了设定的TTL时间，注册中心就会将服务删除。

## 修改源码

```
	service := grpc.NewService(
		micro.Name("go.micro.srv.hello"),
		micro.WrapHandler(ocplugin.NewHandlerWrapper(t)),
+		micro.RegisterTTL(time.Second*15),
+		micro.RegisterInterval(time.Second*10),
		// micro.Version("latest"),
	)
```

```
service := web.NewService(
		web.Name(name),
		web.Version("lastest"),
+		web.RegisterTTL(time.Second*15),
+		web.RegisterInterval(time.Second*10),
		web.MicroService(grpc.NewService()),
	)
```

