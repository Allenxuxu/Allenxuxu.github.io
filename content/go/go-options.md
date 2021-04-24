---
title: "Golang实现默认参数"
date: 2019-06-27T11:22:38+08:00
draft: false
tags: ["Go"]
---

在golang 中是不支持默认参数的，micro中有一种优雅的实现方法(并非 micro 首创)，叫做 Functional Options Patter。Functional Options 可以用来实现简洁的支持默认参数的函数方法。

## options

```go
package server


import (
	"time"
)

type Options struct {
	ConnectTimeOut time.Duration
	Name           string
	Address        string
}

type  Option  func(*Options)

func newOptions(opt ...Option) Options {
	opts := Options{}

	for _, o := range opt {
		o(&opts)
	}

	if len(opts.Address) == 0 {
		opts.Address = DefaultAddress
	}

	if len(opts.Name) == 0 {
		opts.Name = DefaultName
	}

	if opts.ConnectTimeOut == time.Duration(0) {
		opts.ConnectTimeOut = DefaultConnectTimeOut
	}

	return opts
}

// Name server name
func Name(n string) Option {
	return func(o *Options) {
		o.Name = n
	}
}

// Address server address
func Address(a string) Option {
	return func(o *Options) {
		o.Address = a
	}
}

// ConnectTimeOut 连接超时时间
func ConnectTimeOut(t time.Duration) Option {
	return func(o *Options) {
		o.ConnectTimeOut = t
	}
}
```

## server

```go
package server

import  "sync"

var (
    DefaultAddress  =  ":0"
    DefaultName  =  "server"
    DefaultConnectTimeOut  = time.Second *  4
)

type  Server  struct {
    sync.RWMutex
    opts Options
}

func  NewServer(opts ...Option) Server {
    options  :=  newOptions(opts...)
    return  &Server{
        opts: options,
    }
}

func (s *Server) Options() Options {
    s.RLock()
    opts  := s.opts
    s.RUnlock()
    return opts
}

func (s *Server) Init(opts ...Option) error {
    s.Lock()
    for  _, opt  :=  range opts {
        opt(&s.opts)
    }
    s.Unlock()
    return  nil
}

func (s *Server) Start() error {
    return  nil
}
  
func (s *Server) Stop() error {
    return  nil
}
```

## 使用

```go
server := NewServer( 
	Name("test Name"), 
	Address("test Address"), 
 )
```

