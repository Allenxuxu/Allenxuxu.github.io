---
title: "go chan 实用示例"
date: 2020-05-30T11:22:38+08:00
draft: false
tags: ["Go"]
---

- 尝试发送

```go
select {
case c <- struct{}{}:
default:
	fmt.Println("chan 已满，发送不成功")
}
```

- 尝试接收

```go
select {
case v := <- c:
default:
	fmt.Println("chan 中没有信息，接收不成功")
}
```

> 标准编译器对尝试发送和尝试接收代码块做了特别的优化，使得它们的执行效率比多 `case`分支的普通 `select`代码块执行效率高得多。

- 无阻塞的检查一个 chan 是否关闭

假设我们可以保证没有任何协程会向一个通道发送数据，则我们可以使用下面的代码来（并发安全地）检查此通道是否已经关闭，此检查不会阻塞当前协程。

```go
func IsClosed(c chan struct{}) bool {
	select {
	case <-c:
		return true
	default:
	}
	return false
}
```

- 最快回应

```go
package main

import (
	"fmt"
	"math/rand"
	"time"
)

func source(c chan<- int, id int) {
	 rb := rand.Intn(3)+1
	// 休眠1秒/2秒/3秒
	time.Sleep(time.Duration(rb) * time.Second)
	 // 使用尝试放松，防止阻塞
	select {
	case c <- id:
	default:
	}
}

func main() {
	rand.Seed(time.Now().UnixNano())

	c := make(chan int, 1) // 此通道容量必须至少为1
	for i := 0; i < 5; i++ {
		go source(c, i)
	}
	id := <-c // 只采用第一个成功发送的回应数据
	fmt.Println(id)
}
```

- 超时机制

```go
func doRequest(data chan int) {
	time.Sleep(time.Second * 10)
	data <- 1
}

func requestWithTimeout(timeout time.Duration) (int, error) {
	data := make(chan int)
	go doRequest(data) // 可能需要超出预期的时长回应

	select {
	case data := <-data:
		return data, nil
	case <-time.After(timeout):
		return 0, errors.New("超时了！")
	}
}
```

- 防止重复 close chan

```go
func Stop() error {
	select {
	case <-w.exit:
	default:
		close(w.exit)g
	}
	return nil
}
```

