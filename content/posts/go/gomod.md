---
title: "Go mod 小结"
date: 2021-05-12T11:22:38+08:00
draft: false
tags: ["Go"] 
---

## go.mod 文件

```
module example.com/foobar



go 1.13



require (

    example.com/apple v0.1.2

    example.com/banana v1.2.3

    example.com/banana/v2 v2.3.4

    example.com/pineapple v0.0.0-20190924185754-1b0db40df49a

)



exclude example.com/banana v1.2.4

replace example.com/apple v0.1.2 => example.com/rda v0.1.0 

replace example.com/banana => example.com/hugebanana
```

- module：用于定义当前项目的模块路径。

- go：用于设置预期的 Go 版本。

- require：用于设置一个特定的模块版本。

- exclude：用于从使用中排除一个特定的模块版本。

- replace：用于将一个模块版本替换为另外一个模块版本。

### 版本表示方式

- 基于某一个commit的伪版本号
  - 基本版本前缀-commit的UTC时间-commit的hash前12位
    - vX.0.0-yyyymmddhhmmss-abcdefabcdef

- vX.Y.Z-pre.0.yyyymmddhhmmss-abcdefabcdef

- vX.Y.(Z+1)-0.yyyymmddhhmmss-abcdefabcdef  



需要注意的是，同一个仓库的 v2.x.x 和之前小于 v2 大版本的代码被认为是两个不同的仓库。

**Go modules 规定主版本号不是 v0 或者 v1 时，那么主版本号必须显式地出现在模块路径的尾部。**例如，上面示例 go.mod 文件中的 ：

```
    example.com/banana v1.2.3

    example.com/banana/v2 v2.3.4
```



## Go mod 版本选择算法

在go mod中，项目依赖了A、B两个项目，且A、B分别依赖了C项目的v1.3、v1.3两个版本。

最终会选择最高的那个版本 v1.4.

![img](https://cdn.jsdelivr.net/gh/Allenxuxu/blog/img/(null))

**对每个依赖项，选择其所有被依赖版本中最高的那个版本。**

## Go mod 的使用方法和工具

- 用 go get 拉取新的依赖
  - 拉取最新的版本(优先择取 tag)：go get golang.org/x/text@latest

- 拉取 master 分支的最新 commit：go get golang.org/x/text@master

- 拉取 tag 为 v0.3.2 的 commit：go get golang.org/x/text@v0.3.2

- 拉取 hash 为 342b231 的 commit，最终会被转换为 v0.3.2：go get golang.org/x/text@342b2e

- 用 go get -u 更新现有的依赖

- 用 go mod download 下载 go.mod 文件中指明的所有依赖

- 用 go mod tidy 整理现有的依赖

- 用 go mod graph 查看现有的依赖结构

- 用 go mod init 生成 go.mod 文件 

- 用 go mod edit 编辑 go.mod 文件

- 用 go mod vendor 导出现有的所有依赖 (事实上 Go modules 正在淡化 Vendor 的概念)

- 用 go mod verify 校验一个模块是否被篡改过

## Go mod 的常见问题

- 主版本号
  - Go get -u 不会更新主版本号，如果需要更新，需要手动修改其导入路径
    - **Go modules 规定主版本号不是 v0 或者 v1 时，那么主版本号必须显式地出现在模块路径的尾部。** 