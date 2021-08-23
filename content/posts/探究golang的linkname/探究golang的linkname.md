---
title: 探究golang的linkname
date: 2018-10-27 21:58:13
tags:
---
在编写golang程序的过程中，会经常有一些sleep的需求，于是我们使用`time.Sleep`函数
跳转到函数定义处发现这个函数定义如下：
```
// Sleep pauses the current goroutine for at least the duration d.
// A negative or zero duration causes Sleep to return immediately.
func Sleep(d Duration)

```
没错，只有定义没有实现？显然不是，函数的实现在`runtime/time.go`
```
// timeSleep puts the current goroutine to sleep for at least ns nanoseconds.
//go:linkname timeSleep time.Sleep
func timeSleep(ns int64) {
	if ns <= 0 {
		return
	}

	gp := getg()
	t := gp.timer
	if t == nil {
		t = new(timer)
		gp.timer = t
	}
	*t = timer{}
	t.when = nanotime() + ns
	t.f = goroutineReady
	t.arg = gp
	tb := t.assignBucket()
	lock(&tb.lock)
	if !tb.addtimerLocked(t) {
		unlock(&tb.lock)
		badTimer()
	}
	goparkunlock(&tb.lock, waitReasonSleep, traceEvGoSleep, 2)
}
```
看看`go:linkname`的[官方说明](https://golang.org/cmd/compile/)
```
//go:linkname localname importpath.name

The //go:linkname directive instructs the compiler to use “importpath.name” as the object file symbol name for the variable or function declared as “localname” in the source code. Because this directive can subvert the type system and package modularity, it is only enabled in files that have imported "unsafe".
```
这个指令告诉编译器为当前源文件中私有函数或者变量在编译时链接到指定的方法或变量。因为这个指令破坏了类型系统和包的模块化，因此在使用时必须导入`unsafe`包，所以可以看到`runtime/time.go`文件是有导入`unsafe`包的。
我们看到`go:linkname`的格式，这里`localname`自然对应`timeSleep`, `importpath.name`就对应`time.Sleep`,但为什么要这么做呢？
我们知道`time.Sleep`在`time`包里，是可导出，而`timeSleep`在`runtime`包里面，是不可导出了，那么`go:linkname`的意义在于让`time`可以调用`runtime`中原本不可导出的函数，有点`hack`，举个栗子：

目录结构如下
```
➜  demo git:(master) ✗ tree
.
├── linkname
│   └── a.go
├── main.go
└── outer
    └── world.go
```

文件内容 a.go
```
package linkname

import _ "unsafe"

//go:linkname hello examples/demo/outer.World
func hello() {
	println("hello,world!")
}
```
world.go
```
package outer

import (
	_ "examples/demo/linkname"
)

func World()
```
main.go
```
package main

import (
	"examples/demo/outer"
)

func main() {
	outer.World()
}
```
运行如下：
```
# examples/demo/outer
outer/world.go:7:6: missing function body
```
难道理解错了，这是因为`go build`默认加会加上`-complete`参数，这个参数检查到`World()`没有方法体，在`outer`文件夹中增加一个空的`.s`文件即可绕过这个限制
```
➜  demo git:(master) ✗ tree
.
├── linkname
│   └── a.go
├── main.go
└── outer
    ├── i.s
    └── world.go
```
输出如下：
```
hello,world!
```

参考：
- https://forum.golangbridge.org/t/implementation-of-time-sleep/2637
- https://github.com/golang/go/issues/15006
- https://golang.org/cmd/compile/
- https://github.com/golang/go/blob/master/src/runtime/time.go#L47
