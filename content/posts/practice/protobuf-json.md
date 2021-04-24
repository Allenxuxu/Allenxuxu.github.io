---
title: "golang protobuf 字段为零值时 json 序列化忽略问题"
date: 2020-06-02T11:22:38+08:00
draft: false
tags: ["protobuf","go"]
---

protoc 编译生成的 pb.go 文件，默认情况下 tag 中会设置 json 忽略零值的返回属性 `omitempty`。

```go
type Message struct {
	Header               map[string]string `protobuf:"bytes,1,rep,name=header,proto3" json:"header,omitempty" protobuf_key:"bytes,1,opt,name=key,proto3" protobuf_val:"bytes,2,opt,name=value,proto3"`
	Body                 []byte            `protobuf:"bytes,2,opt,name=body,proto3" json:"body,omitempty"`
	XXX_NoUnkeyedLiteral struct{}          `json:"-"`
	XXX_unrecognized     []byte            `json:"-"`
	XXX_sizecache        int32             `json:"-"`
}
```

一个比较 hack 的方式，是在 pb.go 文件生成后，手动去删掉 `omitempty` 。每次手动去删除，比较麻烦且容易出错，下面提供一个 Makefile ，每次生成 pb.go 的时候就去删除 `omitempty` 。

```makefile
proto:
	protoc --proto_path=. --go_out=. --micro_out=. config/config.proto
	ls config/*.pb.go | xargs -n1 -IX bash -c 'sed s/,omitempty// X > X.tmp && mv X{.tmp,}'
```

proto 目标的第一个命令是调用 protoc 根据 config/config.proto 生成 pb.go 文件；

第二行命令就是将 config/*.pb.go 中的 `omitempty` 删除。

```go
type Message struct {
	Header               map[string]string `protobuf:"bytes,1,rep,name=header,proto3" json:"header" protobuf_key:"bytes,1,opt,name=key,proto3" protobuf_val:"bytes,2,opt,name=value,proto3"`
	Body                 []byte            `protobuf:"bytes,2,opt,name=body,proto3" json:"body"`
	XXX_NoUnkeyedLiteral struct{}          `json:"-"`
	XXX_unrecognized     []byte            `json:"-"`
	XXX_sizecache        int32             `json:"-"`
}
```

使用时，根据需要修改 `config/config.proto` 和 `config/*.pb.go` 即可。

