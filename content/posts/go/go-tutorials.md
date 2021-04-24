---
title: "Golang 极简入门教程"
date: 2019-08-04T11:22:38+08:00
draft: false
tags: ["Go"]
---

# Hello World

我们以传统的“hello	world”案例开始吧。

```go
package main

import "fmt"

func main() {
    fmt.Println("Hello World")
}
```

Go的源文件以 **.go** 为后缀名，这些文件名均由小写字母（推荐做法）组成且不包含空格和其他特殊字符，如 main.go 。如果文件名由多个部分组成，则使用下划线 _ 对它们进行分隔，如 main_test.go 。

Go是一门编译型语言,Go语言的工具链将源代码及其依赖转换成计算机的机器指令。Go语言提供的工具都通过一个单独的命令 **go**	调用，**go** 命令有一系列子命令。

```
$ go help
Go is a tool for managing Go source code.

Usage:

        go <command> [arguments]

The commands are:

        bug         start a bug report
        build       compile packages and dependencies
        clean       remove object files and cached files
        doc         show documentation for package or symbol
        env         print Go environment information
        fix         update packages to use new APIs
        fmt         gofmt (reformat) package sources
        generate    generate Go files by processing source
        get         download and install packages and dependencies
        install     compile and install packages and dependencies
        list        list packages or modules
        mod         module maintenance
        run         compile and run Go program
        test        test packages
        tool        run specified go tool
        version     print Go version
        vet         report likely mistakes in packages

...
```

我们通过 **go run** 命令编译 main.go 文件并且运行它。

```
$ go run main.go
Hello World
```

前面已经说过了GO语言是一门编译型语言，所以通过 **go** 工具同样可以编译生成二进制文件保存下来。

```
$ go build main.go
```

执行后，会在当前目录生成一个可执行文件 **main** （Windows平台是 main.exe）。我们可以直接在命令行运行它，就像执行 C/C++ 静态编译出来的可执行文件一样。

```
$ ./main
Hello World
```

Go语言的代码通过包组织,包机制类似于其它语言里的库或者模块。一个包由位于单个目录下的一个或多个 **.go** 源代码文件组成。每个源文件都以一条 **package** 声明语句开始，这个例子里就是 **package main**，表示该文件属于哪个包，紧跟着一系列导入 (**import**) 的包。

```go
import "fmt"
```

接下来是这个文件的程序代码，在本例中是 **main** 函数。

main 包是一个比较特殊的包，它定义了一个独立可执行的程序，而不是一个库。在 main 包里的 main 函数是整个程序执行时的入口，就像 C/C++ 里一样。

Go的标准库提供了100多个包， fmt	包含有格式化输出、接收输入等方法。Println 函数是其中一个基础函数,可以打印以空格间隔的一个或多个值,并在最后添加一个换行符。

**func** 是Go语言的关键字之一，用于声明一个函数。一个函数的声明由 func 关键字、函数名、参数列表、返回值列表以及包含在大括号里的函数体组成。本例中的 main 函数参数列表和返回值都是空的，意思就是没有参数和返回值，无需像 C/C++ 中那样再手动添加 void，也不会存在隐式的默认参数。

Go语言不需要在语句或者声明的末尾添加分号,除非一行上有多条语句。但是,实际上编译器会帮我们添加分号。

Go语言在代码格式上强制统一，比如函数作左括号 **{** 必须另起一行，否则会编译报错。这样省去了很多口水仗，也统一了代码风格，提高了代码可读性。Go语言提供 gofmt (go fmt)	工具把代码格式化为标准格式。

```
gofmt  -w main.go 
```

该命令会格式化该源文件的代码并且将格式化后的代码覆盖原始内容，如果不加参数 -w 则只会打印格式化后的结果而不重写文件。在实际开发中，我们可以使用IDE或者编辑器插件自动格式化，无需每次执行命令来格式化代码。



# Golang 的主要特点

> 我发现我花了四年时间锤炼自己用 C 语言构建系统的能力，试图找到一个规范，可以更好的编写软件。结果发现只是对 Go 的模仿。缺乏语言层面的支持，只能是一个拙劣的模仿。 --云风

## 极简设计

Go 语言给人的第一感觉便是简洁。Go 语言通过减少关键字的数量（25 个，截止至发稿日期）来简化编码过程中的复杂度。这些关键字在编译过程中少到不需要符号表来协助解析，这也是Go语言的编译速度也是非常快的原因之一。极少的关键字，极简的语法都极大减少开发者编码的工作量，也提高了代码的可读性。

Go 语言的强类型系统禁止一切隐式类型转换，让代码更加容易阅读，减少犯错的机会。

defer 实现 RAII 也比 C++ 中通过对象生命周期和析构函数的实现方式更加容易理解和简洁明了。

```go
file, err := os.Open("test.txt")
if err != nil {
    panic(err)
}
defer file.Close()
```

Go 语言默认所有类型 zero 初始化，省去了很多无意义的初始化操作，也降低了开发者出错的几率。

Go 语言部署非常简单，编译出来一个静态可执行文件，除了 glibc 外没有其他外部依赖便可以直接运行。并且 Go 语言支持交叉编译，使用自带的工具 **go build** 可以直接将源代码编译成不同平台上的可执行程序。

比如，我们在Mac或者Windows上为Linux编译应用：

```
GOOS=linux GOARCH=amd64 go build main.go
```

只需要声明目标系统（GOOS）与CPU架构（GOARCH）即可。

Go 语言从设计上就坚持极简理念，并且极力给作者提供简单高效的开发体验。

## 开发效率与运行效率齐飞

“少即是多” 这就是 Go 语言贯穿始终的哲学。极简的语法，语言级别的并发管理，自动垃圾回收让开发者可以用最少的代码实现功能强大的程序。Go 语言没有隐式转换，没有构造函数和析构函数,没有运算符重载,没有继承...，极大的降低了开发者的心智负担。完善的类型系统让 Go 语言可以避免动态语言那种粗心的类型错误，同时又没有 C++ 那样繁杂的具体类型属性需要考量。Go 语言强制统一代码风格，减少了不少口水战，也让代码的可读性，可维护性更高了，这也是提高开发效率的关键。Go 语言致力于提供更少的语言特性，通过简洁的设计，减少代码出错的机会，让开发者更容易写出更高质量的代码。

Go 语言出现之前，各种语言在运行效率和开发效率上都不能兼备。Python开发效率高，但是性能差强人意; C/C++ 运行效率毋庸置疑，但是开发效率略低。Go 语言运行效率高是因为 Go 语言是编译型的静态语言，它在执行速度上比解释型语言具有先天的优势，但是同时其简洁的语法又让开发者有种写动态语言的轻松感。Go 语言的运行效率直逼 C/C++ ，之所以稍逊于 C/C++ 主要还是因为 GC（自动垃圾回收机制），考虑到开发效率上的提升，这一点性能损失还是值得的。

## 强大的内置类型和标准库

Go 语言除了几乎所有语言都支持的简单内置类型(比如整型和浮点型等)外， 也内置了一些比较新的语言中内置的高级类型，比如数组、字符串、字典类型(map)。Go语言的标准库覆盖网络、系统、加密、编码、图形等各个方面，可以直接使用标准库的 http 包进行 HTTP 协议的收发处理;网络库基于高性能的操作系统通信模型(Linux 的 epoll、Windows 的 IOCP);所有的加密、编码都内建支持，不需要再从第三方开发者处获取。

| Go语言标准库包名 |	功  能 
|  ----         | ----  
| bufio 	    | 带缓冲的 I/O 操作
| bytes         | 实现字节操作
| container 	| 封装堆、列表和环形列表等容器
| crypto        | 加密算法
| database 	    | 数据库驱动和接口
| debug         | 各种调试文件格式访问及调试功能
| encoding 	    | 常见算法如 JSON、XML、Base64 等
| flag 	        | 命令行解析
| fmt 	        | 格式化操作
| go 	        | Go 语言的词法、语法树、类型等。可通过这个包进行代码信息提取和修改
| html 	        | HTML 转义及模板系统
| image 	    | 常见图形格式的访问及生成
| io 	        | 实现 I/O 原始访问接口及访问封装
| math 	        | 数学库
| net 	        | 网络库，支持 Socket、HTTP、邮件、RPC、SMTP 等
| os 	        | 操作系统平台不依赖平台操作封装
| path 	        | 兼容各操作系统的路径操作实用函数
| plugin 	    | Go 1.7 加入的插件系统。支持将代码编译为插件，按需加载
| reflect 	    | 语言反射支持。可以动态获得代码中的类型信息，获取和修改变量的值
| regexp 	    | 正则表达式封装
| runtime 	    | 运行时接口
| sort 	        | 排序接口
| strings 	    | 字符串转换、解析及实用函数
| time 	        | 时间接口
| text 	        | 文本模板及 Token 词法器
| ...           | ...

## 并发

并发编程可以充分发挥多核处理器的性能。在 C/C++ 中，可以通过编写多线程程序来实现并发，但是滥用线程会加重系统负担，所以更优的做法是使用通过 epoll 等方式来实现IO多路复用，以及使用各种协程库。除此之外，多个线程之间肯定还需要传递数据，可以通过 shared_ptr 来做，但是也需要小心翼翼，整个编码过程非常容易犯错。

goroutine 是 Go 语言并发设计的核心。goroutine 其实就是协程，比线程更轻量，是一种运行在用户态的用户线程。goroutine 并不是对应于内核线程，一个内核线程会调度若干个协程，goroutine 是在语言层面提供了调度器，并且对网络IO库进行了封装，屏蔽了复杂的细节，对外提供统一的语法关键字支持，简化了并发程序编写的成本。channel 是设计来在 goroutine 之间传递数据，channel 在实现原理上其实是一个阻塞的消息队列。在一个 goroutine 中将消息发送到 channel 中，然后在监听这个 channel 的 goroutine 处理，实现了不同 goroutine 的解耦。

## 接口设计

接口类型是对其它类型行为的抽象和概括;因为接口类型不会和特定的实现细节绑定在一起,通过这种抽象的方式我们可以让我们的函数更加灵活和更具有适应能力。

Go语言的主要设计者之一 Rob Pike 曾经说过，如果只能选择一个Go语言的特性移植到其他语言中，他会选择接口。可见接口在Go 语言中的地位，及其对gloang这门语言所带来的活力。

C++,Java 中使用侵入式接口，实现类需要明确声明自己实现了某个接口。这种强制性的接口继承方式是面向对象编程思想发展过程中一个争议颇多的特性。

Go语言采用的是非侵入式接口,只要某类型的公开方法完全满足接口的要求，就可以把此类型的对象用在需要该接口的地方。满足接口的要求，即是指实现了接口所规定的一组成员(方法)。Go 语言的接口实现者无需指明实现了哪一个接口，编译器会去完成这项工作并发现错误。



# 控制结构

Go 程序和大多数编程语言一样从 main() 函数开始执行，然后按顺序执行该函数体中代码。代码中必然需要进行条件判断，Go 中提供如下分支结构：

- if-else 
- switch
- select

Go 中同样有循环结构来重复执行某段代码：

- for(range)

## if-else 结构

if 是用于测试某个条件（布尔型或逻辑型）的语句，如果该条件成立，则会执行 if 后由大括号括起来的代码块。else 这个代码块中的代码只有在 if 条件不满足时才会执行。if 和 else 后的两个代码块是相互独立的分支，永远只会执行其中一个。

```go
if condition {
	// do something	
} else {
	// do something	
}
```

如果需要 增加更多分支选择，可以使用 else if 。else-if 分支的数量是没有限制的，但是当选择条件过多时，应该使用 switch 。

```go
if condition1 {
	// do something	
} else if condition2 {
	// do something else	
} else {
	// catch-all or default
}
```

if 可以包含一个初始化语句，常用于 err 的条件判断。

```go
if initialization; condition {
	// do something
}
```

例如：

```go
if err := fun(); err != nil {
    // do something
}
```

## switch 结构

相比较 C/C++ 等其他语言而言，Go 语言中的 switch 结构使用非常灵活, 并且不需要 breake 语句来跳出。

```go
switch value {
	case v1:
		...
	case v2:
		...
	default:
		...
}
```

value 变量可以是任何类型，v1 和 v2 是同类型的任意值或者是最终结果为相同类型的表达式，但不限于常量和整数。
同一个 case 可以匹配多个可能符合条件的值，通过逗号分割：

```go
switch i {
	case 0: 
	case 1,2,3:
		f() // 当 i == 1 或者 i == 2 或者 i == 3 则执行 f()
}
```

switch 语句可以不提供任何被判断的值，然后在每个 case 分支中进行测试不同的条件。当任一分支的测试结果为 true 时，该分支的代码会被执行。这看起来非常像链式的 if-else 语句。

```go
switch {
	case i < 0:
		f1()
	case i == 0:
		f2()
	case i > 0:
		f3()
}
```

switch 语句还可以包含一个初始化语句：

```go
switch result := calculate(); {
	case result < 0:
		...
	case result > 0:
		...
	default:
		// 0
}
```

```go
switch a, b := x[i], y[j]; {
	case a < b: t = -1
	case a == b: t = 0
	case a > b: t = 1
}
```

因为 Go 的 switch 相当于每个case最后都自带一个 break ，匹配成功后就不会向下执行其他 case ，所以如果需要接着执行下一个 case 的可以使用 fallthrough 关键字。

```go 
	i:= 2
    switch i {
    case 1:
            fmt.Println("1")
            fallthrough
    case 2:
            fmt.Println("2")
            fallthrough
    case 3:
            fmt.Println("3")
            fallthrough
    case 4:
			fmt.Println("4")
			fallthrough
    default:
			fmt.Println("default")
			
    }
```

输出：

```bash
2
3
4
default
```

fallthrough 只会强制执行下一个 case 。

```go
	i:= 2
    switch i {
    case 1:
            fmt.Println("1")
            fallthrough
    case 2:
            fmt.Println("2")
            fallthrough
    case 3:
            fmt.Println("3")
            // fallthrough
    case 4:
			fmt.Println("4")
			fallthrough
    default:
			fmt.Println("default")
			
    }
```

输出：

```bash
2
3
```

Go 中的 switch 还可以用来做类型判断。

```go
package main

import "fmt"

func Type(v interface{}) {
	switch v.(type) {
	case bool:
		fmt.Println("bool")
	case float64:
		fmt.Println("float64")
	case int:
		fmt.Println("int")
	case nil:
		fmt.Println("nil")
	case string:
		fmt.Println("string")
	default:
		fmt.Println("default")
	}
}

func main() {
	Type(1)	
	Type("")
	Type(true)
	Type(nil)
	i := 1
	Type(&i)	// *int
}
```

输出：

```bash
int
string
bool
nil
default
```

## for 结构

Go 中循环结构只有 for 语句，并没有 while 语句。

for 语句基本用法和其他语言无异：

```go
for i := 0; i < 5; i++ {
    // do
}
```

支持多个变量控制循环：

```go
for i, j := 0, 1; i < j; i, j = i+1, j-1 {
    // do
}
```

for 语句实现 while 语句功能：

```go
var i int = 3

for i >= 0 {
    i--
    // do
}
```

无限循环：

```go
for {

}
```

```go
for i := 0; ; i++ {

} 
```

```go
for ; ; {
    
}
```

Go 中还提供一个关键字用于循环结构 range ，它可以迭代任何一个集合（数组，map）。

```go
for k, v := range s {
    // do
}
```

需要注意的是，v 对于元素的值拷贝，任何对 v 的修改都不会影响集合 s 。

## break 和 continue

和其他语言一样，break 用于跳出整个循环，continue 用于跳出当前循环继续下一次循环。

```go
for i := 0; i < 3; i++ {
    if i == 1 {
        continue
    }
    println(i)
}
```

输出：

```bash
0
2
```

```go
for i := 0; i < 3; i++ {
    if i == 1 {
        break
    }
    println(i)
}
```

输出：

```bash
0
```



# 基本数据类型和要素

## 包的概念

类似其他语言中的库和模块的概念，目的都是为了支持模块化、封装、单独编译和代码重用。每一个 Go 文件都属于且仅属于一个包，每个包可以有多个 Go 文件。每个包中的程序可以使用自身的包或者导入其他包。

当包内的全局变量或者常量标识符以一个大写字母开头，如： Test，那么它就是可以直接被外部包使用的，称为导出，类似于其他面向对象语言中 public。如果是以小写的字母开头，则对包外是不可见的，但是可以在包内直接使用（同一个包内的不同 .go 文件可以直接使用），类似于其他面向对象语言中的 private。

每个包都对应一个独立的名称空间。不同包的导出函数或者变量即使名称相同，也不会有命名冲突。在外部调用时必须显示指定包，例如： fmt.Println 。如果包名有冲突，可以在导入的时候设置别名，如：

```go
package main

import f "fmt"

func main() {
    f.Println("Hello World")
}
```

如果需要导入多个包

```go
import "fmt"
import "os"
```

但是有更简短的做法

```go
import (
   "fmt"
   "os"
)
```

## 注释

Go 提供了 C 样式 /* */ 块注释和 C++ 样式 // 行注释。行注释是标准规范，块注释主要作为包注释出现或者是禁用大量代码时使用。

```go
// 单行注释

/*
   块注释
*/
```

## 常量

Go 的常量使用 const 关键字定义，常量的数据类型只可以是布尔型、数字型和字符串类型。

```go
const a string = "abc"
const b = "abc" 
```

Go 的编译器可以自动推断类型，所以以上两种定义方法都是可以的。

常量的值必须是能够在编译期就能够确定的，可以在其赋值表达式中涉及计算过程，但是所有用于计算的值必须在编译期间就能获得。

```go
const c = 1+3
```

上面是正确的做法，但是下面的 func1 自定义函数无法在编译期求值，因此无法用于常量的赋值，但是 Go 内置的函数是可以的，如： len() 。

```go
const c = func1() 
```

Go 对关键字十分吝啬，对于枚举类型没有专门的关键字，但是常量可以用作枚举。

```go
const (
    a = 0
    b = 1
    c = 2
)
```

Go 语言还提供了 iota 关键字，可以用来简化常量的增长数字的定义。iota 会自增 1 ,每遇到一次 const 关键字，就重置为 0 。

```go
const (
    a = iota  //0
	b         //1
	c         //2
)
```

## 变量

Go 声明变量使用 var 关键字：

```go
var a int = 1
```

```go
var (
	a int    // 0
	b bool   // false
	c string // ""
    d *int   // nil
)
```

声明变量时可以不赋值，默认初始化都会是 ”零“ 值，不会出现 C/C++ 那样的随机值。

Go 的编译器同样可以根据变量的值来自动推断其类型：

```go
var a = 12
```

Go 还提供简短声明语法 := ，不过只可以用于声明函数体内的局部变量，不能用在全局变量的声明与赋值，例如：     

```go
a := 1   //等价于 var a = 1
```

需要注意的是 := 是声明并初始化，所以 := 左边必须是一个新值，否则会出现编译错误。

## 基本类型和运算符

### 布尔类型

```go
var a bool = true
```

布尔类型的值只可以是 true 或者 false ，两个类型相同的值可以使用 == 和 != 运算符来比较并且得到一个布尔类型的值。Go 是一门强类型的语言，所以必须是相同类型的两个值才可以进行比较。如果是一个字面量和一个值比较，值的类型必须和字面量类型兼容。

```go
var a = 10

if a == 1 {    // 可以

}

if a == 3.5 {  // 编译报错

}
```

### 数字类型

Go 支持整型、浮点型数字和复数类型。

#### 整形

- int int8 int16 int32 int64
- uint uint8 uint16 uint32 uint64

int 和 uint 在 32 位系统上是 32 位（4个字节），在64位操作系统上是 64 位（8个字节）。其他如 int8 这种都是与系统无关的类型，有固定的大小，从类型的名称就可以看出其大小。

#### 浮点型

- float32
- float64

float32 精确到小数点后 7 位，float64 精确到小数点后 15 位。应该尽可能地使用 float64，因为 math 包中所有有关数学运算的函数都会要求接收这个类型。

#### 复数

Go 提供以下复数类型：

- complex64
- complex128

复数使用 re+imI 来表示，其中 re 代表实数部分，im 代表虚数部分，I 代表根号负 1 。

```go
var c complex64 = 5 + 10i
```

内置的 complex 函数用于构建复数,内建的 real 和 imag 函数分别返回复数的实部和虚部。

```go
var	x complex128 = complex(1, 2)	// 1+2i
var	y complex128 = complex(3, 4)    // 3+4i
fmt.Println(x*y)                    // "(-5+10i)"
fmt.Println(real(x*y))              // "-5"
fmt.Println(imag(x*y))              // "10"
```

### 字符类型

Go 中字符类型 byte 只是整数的特殊用例，byte 类型是 uint8 的别名。

```go
var ch1 byte = 'a'
var ch2 byte = 65
```

### 关系运算符

Go 中拥有以下逻辑运算符，和其他语言的用法相同，运算结果总是为布尔值。

- !=  、 ==
- <  、 <=  、 >  、 >=
- && 、 ||

### 逻辑运算符

&& 、 || 逻辑与和逻辑或同样支持短路法则。

```

```

### 算术运算符

Go 提供常用的整数和浮点数的二元运算符： + 、 - 、* 、/ 。

```go
var a = 5 / 2  // 2
var b = 5 % 2  // 1
```

对于语句  a = a + 2 ，同样提供  -= 、 *= 、 /= 、 %=  运算符来简化写法。

```go
var a = 1
a = a + 2
a += 3
```

++ 、 -- 一元操作符在 Go 中只能用于后缀，并且只能作为语句而非表达式。

```go
i++       // 正确
++i       // 编译报错，不能用于前缀 

a = i++   // 编译报错，不能作为表达式
```

## 字符串

Go 语言的字符串是一个以UTF8编码的字节序列，并且一旦创建就无法更改。无法像 C/C++ 那样通过索引改变字符串中的某个字符（取字符串某个字节的地址也是非法的 &str[i] ），并且 Go 中的字符串是根据长度限定，不是特殊字符 \0 。

```go
var str string
```

Go 语言的字符串如果声明时未初始化，则默认是 ”零“ 值，即空串 "" ，长度为0

```go
str := "Hello " + "World"
str += " ! "
```

字符串可以通过 **+** 号拼接，也可以使用 **+=** 简写形式。

## 数组

数组是一个有固定长度的且类型唯一的数据序列，

```go
var arr [4]int
fmt.Println(a[0])           // 打印第一个元素  0
fmt.Println(a[len(a)-1])    // 打印最后一个元素 0
```

同样，数组的每个元素都被初始化为元素类型对应的 ”零“ 值，在此处是 0 。

```go
var	a [3]int = [3]int{1, 2, 3}
b := [...]int{ 1, 2, 3}
```

可以在声明数组时给定一组值来初始化数组，在数组长度位置用 "..." 三个点来替代，代表数组的长度根据具体数值的个数来计算。

把一个大数组通过函数传参会消耗大量内存，因为 Go 语言都是值传递，会将数组完整的拷贝一份。可以通过两种方法在避免：

- 传递数组的指针
- 使用切片

## 切片(slice)

切片是一个长度可变的数组，类似 C++ 的动态数组（vector）。切片的语法和数组很像，只是切片没有限定固定长度。

```go
var a []int    // 切片
var b [3]int   // 数组
```

一个切片底层由三部分组成：指针、长度、容量。指针指向切片的第一个元素的地址，长度对应切片中元素的数量，容量是切片底层分配的连续内存空间可容纳元素的数量。切片提供 cap() 函数来计算其容量， len() 函数来计算其长度。

```go
s := []int{0, 1, 2, 3, 4, 5}
fmt.Println(s[:2])  // [0 1]
fmt.Println(s[2:])  // [2 3 4 5]
fmt.Println(s)      // [0 1 2 3 4 5]
```

一个 ”零“ 值的切片是nil，长度和容量都为0。

```go
var s []int     // len(s) == 0, s == nil
s = nil         // len(s) == 0, s == nil
s = []int(nil)  // len(s) == 0, s == nil
s = []int{}     // len(s) == 0, s !=nil
```

我们可以通过 **make** 函数创建以一个指定元素类型、长度和容量的切片。容量参数可以不传，Go 会按照指定的长度和类型初始化。

```go
// make([]T, len, cap)	
make([]int, 3)      // [0,0,0]
make([]int, 3, 5)  // [0,0,0, *, *]
```

Go 内置的 append 函数可以向切片追加元素。

```go
var s []int
s = append(s, 2)
```

append 函数底层在每次操作之前都会先检查切片的容量，如果容量够，就会直接将新添加的元素复制到对应位置并将长度加1;如果容量不够，会先分配一个足够大的内存空间，然后将原来的切片内容和新添加的全部复制过去，再返回这个切片。

## Map

Map 是一个无序的 key/value 的集合，类似于其他编程语言中的字典，哈希表。Map 和 切片一样在使用过程会自动扩容。

```go
var m map[string]int
v := make(map[string]int)
v1 := make(map[string]int, 10)  // 初始化容量为 10
a := map[string]int{
				"abc": 1,
				"def": 2,
                }

m["test"] = 1           // 错误， 此时 m 是 nil
fmt.Println(a["abc"])
delete(a,"abc")

```

Map 可以使用 make 函数创建（可以选择在创建时指定容量），也可以通过map字面值的语法创建，同时还可以指定一些最初的 key/value。需要注意的是，未初始化的 map 的值是 nil，直接访问会出错。Map 中的元素通过key对应的下标语法访问。使用内置的delete函数可以删除元素。

key对应的下标语法访问时，通过如果key在map中是存在的,那么将得到与key对应的value。如果key不存在,那么将得到value对应类型的零值。但是元素类型为 int，就无法区分 0 了。为此，Go 提供了两个返回值来区分。

```go
v, ok := a["test"]
if	!ok	{

}
```

当 ok 为 false 时表示 Map 中找不到 key 等于 "test" 对应的元素。

## 结构体

结构体是一种聚合的数据类型,是由零个或多个任意类型的值聚合成的实体。结构体定义的一般方式如下：

```go
// type identifier struct {
//     field1 type1
//     field2 type2
//     ...
// }

type Abc struct {
    A int
    B int
    C int
}

var s Abc
s.A = 1
s.B =2
```

使用内置的 new 函数可以给一个新的结构体变量分配内存，它返回指向已分配内存的指针。Go 中使用点符号获取结构体中的值：structname.fieldname = value 。实际上，在 Go 中无论是值类型还是指针类型都使用点符号，并没有 C/C++ 中的 -> 符号。

```go
var t *Abc
t = new(Abc)

t2 := new(Abc)

t.A = 2
t2.B = 3
```

结构体初始化主要有两种方法，一种是按照结构体成员定义的顺序为每个成员指定一个面值，这样如果结构体成员顺序又调整就需要改动所有初始化结构体的地方了，所以不太建议着一种;另一种就是，以成员名字和对应值来初始化。

```go
type Abc struct {
    A int
    B int
    C int
}

s := Abc{1, 2, 3}
s2 := Abc{
    A : 1,
    B : 2,
    C : 3,
}
```



# 函数

Go 里面有三种类型的函数：

- 普通的命名函数
- 匿名函数或者lambda函数
- 方法

## 函数参数和返回值

除 main() 、init() 函数外，Go 中其它所有类型的函数都可以有参数与返回值。

函数参数、返回值及它们的类型被统称为函数签名。函数可以返回零个或多个值，相较于 C/C++ 等语言多值返回是 Go 的一大特性。

```go
func Test1(a, b int) int {
    return a + b
}
```

```go
func Test2(a, b int) (int, int) {
    return b, a
}
```

### 命名返回值

命名返回值作为结果形参被初始化为相应类型的零值，当需要返回的时候，我们只需要一条简单的不带参数的return语句。

```go
func Test3() (ret1 int, ret2 int){
    ret1 = 1
    ret2 = 2

    return
}
```

### 按值传递

Go 中默认都是使用按值传递，也就是说函数传参时都会拷贝一个副本出来到函数内部使用。

如果不希望拷贝带来太大的性能开销，或者希望可以改变参数的内容，可以传递指针。

```go
a := 1
f(&a)
```

指针也是一个变量，函数传参时同样时按值传递，只不过拷贝的是指针，也就是变量的地址。指针通常是一个32位或者64位的值，所以性能开销比传递一些结构体要小的多。

在 Go 中也有一些按引用传递的类型：切片（sleice）、字典（mao）、接口（interface）、通道（chan）。其实，这些类型的底层同样是使用指针来实现的。

例如，切片的底层是一个指针指向一片内存的首地址，len 记录已用内存的长度，cap 记录切片的容量。在传递切片时，仅仅会将这三个值拷贝一份，而不会去拷贝切片里的全部数据。所以，我们在使用 Go 自带的这些引用类型时可以直接传参，无需担心性能开销而传递指针。

```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

### 变长参数

Go 中支持变长参数，在函数的最后一个参数采用 ...type 的形式，可以传递 0 个或者多个参数。

```go
func f(a int , args ...int) {
}
...

f(1, 23, 45, 67, 89)
```

如果参数是数组或者切片，可以通过 val... 来自动展开。

```go
sl := []int{1, 2, 3}

f(1, sl...)
```

## defer

关键字 defer 允许我们推迟到函数返回之前（或任意位置执行 return 语句之后）一刻才执行某个语句或函数。

```go
func a() {
	i := 0
	defer fmt.Println(i)    // 1
	i++
	return
}
```

当有多个 defer 行为被注册时，它们会以 defer 的出现顺序逆序执行（类似栈，即后进先出）。
使用 defer 会有一定的性能开销，但是 defer 在程序 panic 的时候，还保证会执行。所以通过我们会使用 defer 进行一些函数执行收尾工作。例如，关闭文件描述符，解锁等。

## 闭包

Go 支持匿名函数，函数在 Go 中是一等公民，可以将函数赋值给变量，在需要时再执行。

```go
f = func() {
    fmt.Println("func")
}

f()

// 直接执行
func() {
 fmt.Println("func")
}()
```

匿名函数同样被称之为闭包，闭包可使得某个函数捕捉到一些外部状态。例如：引用一些外部变量，这些变量可以在闭包中被操作，生命周期延长至和闭包一样。

```go
func fun() func() {
	i := 1
	return func() {
		i++
		fmt.Println(i)
	}
}

func fun1(f func()) {
	f()
}

func main() {
	fmt.Println("Hello World")
	f := fun()
	f()         // 2
	f()         // 3

	fun1(f)     // 4
}
```

## init 函数

init 函数是 Go 中一个特殊函数，每个包都可以有 init 函数，它先于 main 函数执行，用于做一些初始化操作。

init 函数的主要特点：

- init 函数在全局变量初始化之后，main 函数执行前自动执行，不能被手动调用
- init 函数没有参数和返回值
- 每个包可以包含多个 init 函数，同一个包的 init 函数间的执行顺序不确定
- 不同包内的 init 函数按照导入包的顺序执行

```go
package main                                                                                                                     
import (
   "fmt"              
)

var T int = a()

func a() int {
   fmt.Println("var T int64 = a()")
   return 1
}

func init() {
   fmt.Println("init()")
}

func main() {                  
   fmt.Println("main()")     
}
```

输出：

```
var T int64 = a()
init()
main()
```



# 并发编程

## 并发与并行

并发与并行是不同的。一个并发程序可以在一个单核处理器使用多个线程来执行多个任务，就好像这些任务同时执行一样。但是同一时间点只有一个任务在执行，是操作系统内核在调度不同的线程交叉执行使得它们好像在同时执行一样。而并行是指在同一时间点程序同时执行多个任务，是物理上真正的同时执行，而非看着像。

并行是一种利用多处理器提高运行速度的能力。所以并发程序可以是并行的，设计优秀的并发程序运行在多核或者多处理器上也可以实现并行。

多线程程序可以编写出高并发应用，重复利用多核处理器性能，但是编写多线程程序非常容易出错，最主要的问题是内存中的数据共享。多线程程序在多核处理器上的并行执行和操作系统对线程调度的随机性，导致这多个线程中共享的数据会以无法预知的方式进行操作。

传统解决方案是同步不同的线程，即对数据加锁。这样在同一时间点就只有一个线程可以变更数据，但是这使得原来可以在多核处理器上并行执行的程序串行化了，无法重复利用多核处理器的能力。

## Go 提供的并发编程特性

Go 语言原生支持程序的并发执行。Go 语言提供 协程 (goroutine) 与通道 (channel) 来支持并发编程。

Go 的协程和其他语言中的协程是不太一样。Go 的协程意味着并行，或是可以并行，而其他语言的协程一般来说是单线程串形化执行的，需要程序主动让出当前CPU。

### 协程 goroutine

Go 的协程和操作系统线程不是一对一的关系，一个协程对应于一个或多个线程，映射（多路复用，执行于）在它们之上。也就是说一个协程可能会在多个操作系统线程上都运行过，同一个操作系统线程会运行多个 Go 协程，Go 语言的协程调度器负责完成调度。

操作系统线程上的协程时间片让我们可以使用少量的操作系统线程就能运行任意多个协程，而且 Go 运行时可以聪明的意识到哪些协程被阻塞了，暂时搁置它们并处理其他协程。比如，当系统调用（比如等待 I/O）阻塞协程时，当前协程会被挂起，其他协程会继续在其他线程上工作，当 I/O 事件到来，挂起的协程会自动恢复执行。

Go 每个协程创建时占用4k栈内存，协程的栈会根据需要进行伸缩，不出现栈溢出，开发者不需要关心栈的大小。当协程结束的时候，它会静默退出，用来启动这个协程的函数不会得到任何的返回值。

```go
package main

import (
	"fmt"
	"time"
)

func GoRun(i int) int {
	fmt.Println("go ", i)
	return i
}

func main() {
	fmt.Println("Hello World")

	go func() {
		fmt.Println("go")
	}()

	go func(i int) {
		fmt.Println("go ", i)
	}(1)

	go GoRun(2)

	time.Sleep(1*time.Second)
}
```

输出 ：

```
Hello World
go  2
go
go  1
```

这个输出结果的顺序并不是固定的，因为 go 关键字启动的协程都是并发执行的。

Go 程序 main() 函数也可以看做是一个协程，尽管它并没有通过 go 来启动。如果 main() 函数退出了，其他协程也会随之退出，这就是为什么上面的代码要在最后加上 `time.Sleep(1*time.Second)`。

> 在一个协程中，如果需要进行非常密集的运算，可以在运算循环中周期的使用 runtime.Gosched()。这会让出处理器，允许运行其他协程；它并不会使当前协程挂起，所以它会自动恢复执行。使用 Gosched() 可以使计算均匀分布，使通信不至于迟迟得不到响应。

### 通道 channel

协程间可以使用共享内存来实现通信，Go 提供 sync 包来实现协程同步，不过 Go 中还提供一种更优雅的方式：使用 channels 来同步协程。

通道就像一个可以用于发送类型化数据的管道，Go 保障在任何给定时间内，通道内的一个数据只有一个协程可以对其访问，所以不会发生数据竞争。也就是说，Go 语言保障通道的发送和接受的原子性。

```go
package main

import "fmt"

func main() {
	var ch chan int
	fmt.Println(ch)	// <nil>

	ch = make(chan int, 1)
	fmt.Println(ch, len(ch), cap(ch)) // 0xc00008c000 0 1
}
```

通道是引用类型，未初始化的通道的值是nil，使用 make 分配内存 `ch := make(chan int)`。

通道只能传输一种类型的数据，比如 chan int 或者 chan string，所有的类型都可以用于通道，空接口 interface{} 也可以。通道在 Go 中同样是一等公民，可以存储在变量中，作为函数的参数传递，作为函数返回值，甚至可以通过通道发送它们自身。

通道使用 `<-` 符号来发送或是接受数据，信息按照箭头的方向流动。

`ch <- int1` 表示用通道 ch 发送变量 int1。

`int2 := <- ch` 表示变量 int2 从通道 ch接收数据。如果 int2 已经声明过，则应该写成 `int2 = <- ch ` 。

`<- ch` 表示获取通道的一个值，并且丢弃之，

```go
package main

import (
	"fmt"
	"time"
)

func sendData(ch chan int) {
	ch <- 1
	ch <- 2
	ch <- 3
	ch <- 4
}

func getData(ch chan int) {
	var input int
	for {
		input = <-ch
		fmt.Println(input)
	}
}

func main() {
	ch := make(chan int)

	go sendData(ch)
	go getData(ch)

	time.Sleep(1*time.Second)
}
```

输出：

```
1
2
3
4
```

通道是可以带缓冲的，`ch := make(chan int, 5)` 即通道里可以容纳 5 个 int 类型的值。`ch := make(chan int)` 默认是没有缓冲区的，即容量大小为1 。当通道数据满时，往通道中发送操作会阻塞，直到通道中有空闲的空间。当通知中没有数据时，从通道中接受数据的操作会被阻塞，直到通道缓冲区中有数据。

将上面的例子稍作修改：

```go
package main

import (
	"fmt"
	"time"
)

func sendData(ch chan int) {
	fmt.Println("sendData")
	ch <- 1
	fmt.Println("ch <- 1")
	ch <- 2
	fmt.Println("ch <- 2")
	ch <- 3
	fmt.Println("ch <- 3")
	ch <- 4
	fmt.Println("ch <- 4")
}

func main() {
	ch := make(chan int)

	go sendData(ch)

	time.Sleep(1 * time.Second)
}
```

输出：

```
sendData
```

因为没有接收通道 ch 数据，所以协程 sendData 一直阻塞在 `ch <- 1`，直到 main 函数 time.Sleep 结束后程序退出。

将通道设为有缓冲区的，设置容量为2: `ch := make(chan int, 2)`, 重新执行，输出如下：

```
sendData
ch <- 1
ch <- 2
```

下面验证一下接收数据阻塞的情况

```go
package main

import (
	"fmt"
	"time"
)

func getData(ch chan int) {
	var input int
	for {
		fmt.Println("getData")
		input = <-ch
		fmt.Println(input)
	}
}

func main() {
	ch := make(chan int, 2)

	go getData(ch)

	time.Sleep(1 * time.Second)
}
```

输出：

```
getData
```

程序启动了一个协程来接收通道 ch 中的数据，但是没有操作来往通道中发送数据，所以协程 getData 一直阻塞在 `input = <-ch`，直到程序退出。

通道创建的时候都是双向的，但是通道类型可以用注解来表示它只发送或者只接收，从而来限制协程对通道的操作。

```go
package main

import (
	"fmt"
	"time"
)

func sendData(ch chan<- int) {
	ch <- 1
	ch <- 2
	ch <- 3
	ch <- 4
}

func getData(ch <-chan int) {
	var input int
	for {
		input = <-ch
		fmt.Println(input)
	}
}

func main() {
	ch := make(chan int)

	go sendData(ch)
	go getData(ch)

	time.Sleep(1 * time.Second)
}

```

通道可以通过 close 显式关闭，如果通道类型被注解，只有发送类型的通道可以被关闭。对已经 close 过的通过再次 close 会导致运行时的 panic 。读取已经关闭的通道，会立即返回通道数据类型的零值。

```go
package main

import (
	"fmt"
	"time"
)

func sendData(ch chan<- int) {
	ch <- 1
	ch <- 2
	ch <- 3
	ch <- 4
	close(ch)
}

func getData(ch <-chan int) {
	var input int
	for {
		input = <-ch
		fmt.Println(input)
	}
}

func main() {
	ch := make(chan int)

	go sendData(ch)
	go getData(ch)

	time.Sleep(1 * time.Second)
}
```

输出：

```go
1
2
3
4
0
0
...
```

上面的输出，会继续一直打印 0 ，直到程序退出。

Go 提供方法来检测通道是否已经关闭：

```go
v, ok := <-ch 
```

当通道已经关闭的时候，ok 为 false；通道打开时，ok 为 true 。

还可以使用 for-range 来读取通道，这会自动检测通道是否关闭。

```go
package main

import (
	"fmt"
	"time"
)

func sendData(ch chan<- int) {
	ch <- 1
	ch <- 2
	ch <- 3
	ch <- 4
	close(ch)
}

func getData(ch <-chan int) {
	var input int

	for input = range ch {
		fmt.Println(input)
	}

	fmt.Println("getData exit")
}

func main() {
	ch := make(chan int)

	go sendData(ch)
	go getData(ch)

	time.Sleep(1 * time.Second)
}
```

输出：

```
1
2
3
4
getData exit
```

从上面的例子可以看出，当通道被关闭时， for-range 循环会自动跳出，结束循环。

现实的开发中，会运行很多的协程，可能需要从多个通道中接收或者发送数据，Go 可以使用 select 关键字来处理多个通道的问题。

select 监听进入通道的数据，如果所有的通道的都没有数据则会一直阻塞，直到有一个通道有数据；如果有多个可以处理，select 会随机选择一个处理；特别需要注意的是，如果所有的通道都没有数据，而且写了 default 语句，则会执行 default 。

```go
package main

import (
	"fmt"
	"time"
)

func sendData1(ch chan<- int) {
	ch <- 1
	ch <- 2
	ch <- 3
	ch <- 4
	// close(ch)
}

func sendData2(ch chan<- string) {
	ch <- "a"
	ch <- "b"
	ch <- "c"
	ch <- "d"
	// close(ch)
}

func getData(ch1 <-chan int, ch2 <-chan string) {
	for {
		select {
		case v := <-ch1:
			fmt.Println(v)
		case v := <-ch2:
			fmt.Println(v)
		// default:
		// 	fmt.Println("default")
		}
	}
}

func main() {
	ch1 := make(chan int)
	ch2 := make(chan string)

	go sendData1(ch1)
	go sendData2(ch2)
	go getData(ch1, ch2)

	time.Sleep(1 * time.Second)
}
```

输出：

```
1
2
a
b
3
c
4
d
```

如果将上面注释掉的 default 语句处的代码打开，则在正确接收所有通道的所有数据后会一直打印 default ，直到程序退出。

select 不会自动处理通道关闭的情况，如果将代码中关于 close 的代码注释打开，select 正确接收所有通道的所有数据后会只一直打印 0 和 "" (int 和 string 的零值)。`case v,ok := <-ch1:` 可以判断通道的开关情况。

