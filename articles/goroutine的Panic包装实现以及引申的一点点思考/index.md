---
title: "Golang的Panic包装实现以及引申的一点点思考"
publish_time: "2026-07-01"
hidden: false
---

<p style="color: rgba(127, 127, 127, 0.9);">最近在清理之前收藏的文章, 其中有一段话是, "刚使用 Golang 的人很容易踩到的一个 goroutine 坑点是，一个 goroutine 如果 panic 了，在它的父 goroutine 是无法 recover 的——严格来讲，并没有父子 goroutine 的概念，一旦启动，就是一个独立的 goroutine 了"</p>

这就很有意思了, 虽然现在有很多的开源实现, 但是我想自己写一个, 因为很明显的, 这里需要用到泛型\. 而我之前没有实践过\. 还是从最简单的部分开始\.

```Go
func **Go**(fn func()) {
    go func() {
        defer func() {
            if v := **recover**(); v != nil {
                log.**Printf**("goroutine panic: %v\n", v)
            }
        }()
        **fn**()
    }()
}

func **main**() {
    **Go**(func() {
        **panic**("something went wrong")
    })

    **Go**(func() {
        fmt.**Println**("hello from goroutine")
    })

    time.**Sleep**(time.Second)
}
```

但是这里有一个问题是, goroutine没有参数啊\. 假设这里我需要传递一个in, 一个out 两个channel, 为了达到这个目的, 我想到了三种办法

```Go
package main

import (
    "fmt"
    "log"
    "time"
)

func **Go**(fn func()) {
    go func() {
        defer func() {
            if v := **recover**(); v != nil {
                log.**Printf**("goroutine panic: %v\n", v)
            }
        }()
        **fn**()
    }()
}

// --- 方法 1：工厂函数，先绑 in/out，再返回 func() ---

func **pipeWorker**(in <-chan int, out chan<- int) func() {
    return func() {
        for v := range in {
            out <- v * 2
        }
        **close**(out)
    }
}

func **example1**() {
    in := **make**(chan int, 3)
    out := **make**(chan int, 3)
    in <- 1
    in <- 2
    in <- 3
    **close**(in)

    **Go**(**pipeWorker**(in, out))

    for v := range out {
        fmt.**Println**("method 1:", v)
    }
}

// --- 方法 2：结构体打包，参数与业务逻辑一并放入结构体 ---

type PipeWorker struct {
    In  <-chan string
    Out chan<- string
    Fn  func(string) string
}

func (w PipeWorker) **Run**() {
    for v := range w.In {
        w.Out <- w.**Fn**(v)
    }
    **close**(w.Out)
}

func **example2**() {
    in := **make**(chan string, 3)
    out := **make**(chan string, 3)
    in <- "go"
    in <- "rust"
    in <- "zig"
    **close**(in)

    w := PipeWorker{
        In:  in,
        Out: out,
        Fn: func(s string) string {
            return "[" + s + "]"
        },
    }
    **Go**(w.**Run**)

    for v := range out {
        fmt.**Println**("method 2:", v)
    }
}

// --- 方法 3：直接闭包，捕获外层的 in/out ---

func **example3**() {
    in := **make**(chan int, 3)
    out := **make**(chan int, 3)
    in <- 7
    in <- 8
    in <- 9
    **close**(in)

    **Go**(func() {
        for v := range in {
            out <- v + 1
        }
        **close**(out)
    })

    for v := range out {
        fmt.**Println**("method 3:", v)
    }
}

func **main**() {
    **example1**()
    **example2**()
    **example3**()

    time.**Sleep**(100 * time.Millisecond)
}

```

写到这里, 忽然发现并没有泛型什么事情, 等下一次好机会\.\.\.

<完>
