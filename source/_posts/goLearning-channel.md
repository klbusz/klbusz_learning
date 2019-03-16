---
title: channel的使用
date: 2019-03-02 16:41:25
catagories: 
- go学习笔记
tags:
- channel
- go
---

<!-- toc -->

# 简介
channel是Go语言的重要特性，和goroutine配合使用，使得Go语言的并发编程变得方便和有趣。

Go语言之父Rob Pike说过：不要通过共享内存来通信，要通过通信来共享内存。如果通过共享内存来交换数据，为了避免数据竞争(data race),我们就需要对数据区域加锁。因此Go语言中就设计实现了channel这种内置类型来支持并发编程中的数据交换。Go中的channel其实就是一个FIFO的队列，可以在一个gorountinue中写入数据，从另一个gotountine中读取数据。


# 使用

## 画重点
* channel 和slice、map一样，是引用类型
* 无缓冲区channel阻塞性读写
* 关闭channel写入panic
* 关闭已关闭channel会panic
* 关闭值为nil的channel会panic
* for-range读取channel
* select-case读取channel


操作|nil channel|关闭的channel|未关闭可使用channel
--|----|---|----
close|panic|panic|成功关闭
写 ch<- | 阻塞|panic|阻塞或者写入成功(有无可用缓冲区或者读取)
读 <- ch |阻塞|读取零值|阻塞或者读取成功(有无数据可读)


## 声明channel
```GO
var chanName chan ElementType
```

```Go
//双向channel
var ch chan int //可读可写
//只读channel
var chR <-chan int 
//只写channel
var chW chan<- int

```

```go
package main

func main(){
	ch := make(chan int,1)
	readOnly(ch)
}

func readOnly(chR <-chan int){
	chR <- 1
}
//编译报错：./test.go:10:6: invalid operation: chR <- 1 (send to receive-only type <-chan int)

```

## 创建channel

```Go
//不带缓冲区channel
ch := make(chan int,0) 

// 带缓冲区channel

ch := make(chan int,10)
```

## channel 关闭

```Go
close(ch)
```

关闭已关闭的channel会panic

一般在生产方关闭channel

关于关闭channel的更多细节，可以看[如何优雅关闭channel](https://studygolang.com/articles/9478)

## channel 使用

**cap(ch)**

channel容量 make(chan int,x)中x的值

**len(ch)**  

channel 中实际元素数目

```Go
package main

import("fmt")

func main(){
    ch := make(chan int,2)
    c := cap(ch)
    ch <- 1
    l := len(ch)
    fmt.Println(c,l)
}
//输出 2，1

```

**读写**

写入

`ch <- i`

读取

`v := <- ch`

`v , ok := <- ch`

当缓冲区满时，channel写入是阻塞的；当缓冲区为空时，channel的读取也是阻塞的；不带缓冲区的channel对于写入就是缓冲区满，对于读取就是缓冲区空。程序会一直阻塞直到满足读写条件(有数据可读，有缓冲区可写)。

当channel关闭后，再写入将会panic；

从关闭的channel中读取，ok值为false，表示channel关闭，v值为channel的元素的零值。因此当循环读取channel值时，可以根据ok的值来判断是否结束循环。

```Go
package main

func main(){
    ch := make(chan int)
    ch <- 1
}
//输出：fatal error: all goroutines are asleep - deadlock!
```

```go
package main

import(
    "fmt"
    "time"
)

func main(){
    ch := make(chan int,1)
    fmt.Println(time.Now().Unix())
    go func(ch chan int){
        time.Sleep(1 * time.Second)
        ch <- 1
    }(ch)
    <- ch
    fmt.Println(time.Now().Unix())
}
/*
输出
1551523332
1551523333
*/
```

```go
package main
import(
    "fmt"
    "time"
)

func main(){
    ch := make(chan int,1)
    go func(ch chan int){
        ch <- 1
        close(ch)
    }(ch)
    time.Sleep(time.Second)
    i := <-ch
    fmt.Println(i) 
    i = <-ch
    fmt.Println(i)
    ch <- 1

    time.Sleep(time.Second)
}
/*
输出
1
0
panic: send on closed channel
*/
```

```go
package main
import(
    "fmt"
    "time"
)

func main(){
    ch := make(chan int,1)
    go func(ch chan int){
        ch <- 1
        close(ch)
    }(ch)
    time.Sleep(time.Second)
    
    //循环读取channel值
    for {
        v,ok := <- ch
        if ok{
            fmt.Println(v)
        }else{
            break
        }	
    }
}
```

**for-range**

使用for-range也可以从channel中读取数据，当channel关闭后，会自动结束循环，不再需要我们自己判断

```Go
package main

import(
	"fmt"
)

func main(){
	ch := make(chan int,3)
	go func(ch chan int){
		ch <- 1
		ch <- 2
		ch <- 3
		close(ch)
	}(ch)
	for v := range ch{
		fmt.Println(v)
	}
}
```

**select-case**

select-case用于处理存在多个case的情况。case 后的语句只能是对channel的读写操作，所有case对应的语句会依次自上而下、自左向右执行，判断是阻塞和操作成功。若所有的case都阻塞，且没有default语句，则select阻塞，若有default则会执行default代码块。当存在多个可执行case，则会从随机选取一个执行；只要擦存在可执行的非阻塞case，则不会执行default。

```Go
package main

import(
	"fmt"
	"time"
)

func main(){
	ch1 := make(chan int,1)
	ch2 := make(chan int,1)
	go func(ch1,ch2 chan int){
		ch1 <- 1
	}(ch1,ch2)
	time.Sleep(time.Second)
	select{
	case v1 := <- ch1:
		fmt.Println(v1)
	case v2 := <- ch2:
		fmt.Println(v2)
	default:
		fmt.Println("default")
	}
}


```



# 参考
* http://go101.org/article/channel.html
* https://studygolang.com/articles/9532
* https://www.jianshu.com/p/2a1146dc42c3
* https://studygolang.com/articles/9478
