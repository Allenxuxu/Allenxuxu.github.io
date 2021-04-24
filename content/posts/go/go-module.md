---
title: "拥抱 Go module"
date: 2019-08-20T11:22:38+08:00
draft: false
tags: ["Go"]
---

go get 拉包一直时国内选手头疼的问题，虽然梯子可以解决问题，但是总是有很慢的时候，而且需要每台电脑都配置，特别是 CI 的服务器等，很烦人。

七牛云开源了 goproxy ，还免费提供 https://goproxy.cn 作为代理来拉包。

不过 GOPROXY 只有在 Go module 下才能使用，索性全面拥抱 Go module 一劳永逸。

修改一下配置文件，即可：

```bash
sudo vi /etc/profile
```

在最后添加如下内容，开启 Go module 和代理：

```bash
export GO111MODULE=on
export GOPROXY=https://goproxy.cn
```

让配置文件立即生效

```bash
source /etc/profile
```

接下来就可以畅快 Go 了！



PS： Go 1.16 已经默认开启 go moudle了。