---
title: "Go代码断行规则"
publish_time: "2024-01-15"
updates:
hidden: false
---

go 编译器为下列情形自动添加换行分号

```txt
在Go代码中，注释除外，如果一个代码行的最后一个语法词段（token）为下列所示之一，则一个分号将自动插入在此字段后（即行尾）：
一个标识符；
一个整数、浮点数、虚部、码点或者字符串字面量；
这几个跳转关键字之一：break、continue、fallthrough和return；
自增运算符++或者自减运算符--；
一个右括号：)、]或}。
```

因此

1. `.`或`,`号结尾不违法, `)`结尾非法
2. 如果 if 条件太长, && 结尾
3. 如果函数太长, 一行代码结尾要带,号

```go
anObject.
MethodA().
MethodB().
MethodC() // 合法

anObject
.MethodA()
.MethodB()
.MethodC() // 非法

if a > 1 && b > 2 && c > 3 && d > 4 && e > 5 && f > 6 && g > 7 && h > 8 &&
i > 9 && j > 10 {
    // do something
} // 合法

if a > 1 && b > 2 && c > 3 && d > 4 && e > 5 && f > 6 && g > 7 && h > 8
 &&  > 9 && j > 10
{
    // do something
} // 非法
var _, _ = f1(123, "Go",  // 最后一个逗号是必需的
)
```

[参考](https://gfw.go101.org/article/line-break-rules.html)
