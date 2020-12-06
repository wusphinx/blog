---
title: prometheus如何实现配置文件动态管理
date: 2020-12-06 16:34:55
tags:
---
因为prometheus并不提供配置文件动态更新的API，所以使用**文件**作为prometheus服务发现的情况下，只能通过修改文件的方式达到更新配置的目的，
常规步骤如下：

1. 修改配置文件
2. 重启prometheus服务（调用`/-/reload`使得配置变更生效，前提是开启了`--web.enable-lifecycle`支持）

一般修改配置如告警规则等属于低频事件，修改保持了就需要重启使之生效，那其实期望这两步能够合二为一，即：
修改完配置立刻应用更新。

promethesu本身也提供这一功能，配置关键字为`file_sd_configs`，实现这一功能的引入的项目为[fsnotify](https://github.com/fsnotify/fsnotify)
>Go的文件系统通知

要实现以上两步合二为一的功能，很容易想到以下做法：
1. 监听文件变化
2. 处理文件变化

可以先来看看fsnotify的示例代码
```go
package main

import (
	"log"

	"github.com/fsnotify/fsnotify"
)

func main() {
	watcher, err := fsnotify.NewWatcher()
	if err != nil {
		log.Fatal(err)
	}
	defer watcher.Close()

	done := make(chan bool)
	go func() {
		for {
			select {
			case event, ok := <-watcher.Events:
				if !ok {
					return
				}
				log.Println("event:", event)
				if event.Op&fsnotify.Write == fsnotify.Write {
					log.Println("modified file:", event.Name)
				}
			case err, ok := <-watcher.Errors:
				if !ok {
					return
				}
				log.Println("error:", err)
			}
		}
	}()

	err = watcher.Add("/tmp/foo")
	if err != nil {
		log.Fatal(err)
	}
	<-done
}
```
可以看到监听文件是由`watcher.Add("/tmp/foo")`完成的，对文件变化事件的处理是由其中的起的`Goroutine`完成的。看到这里其实可以知道关键在于监听这一步是如何实现的，那就来debug一下吧，笔者环境为`go1.14.4 darwin/amd64`，使用VSCode进行代码调试，代码目录为下（为方便调试，建议将项目依赖使用`go mod vendor`打入vendor目录中）：
```
.
├── go.mod
├── go.sum
├── main.go
├── notes.md
└── vendor
    ├── github.com
    ├── golang.org
    └── modules.txt
```
调试配置如下
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "fsnotify",
            "type": "go",
            "request": "launch",
            "mode": "auto",
            "program": "${workspaceRoot}/main.go",
            "env": {},
            "args": []
        }
    ]
}
```
单步调试的过程其实并不是特别直观，这里使用使用一个工具叫做`go-callvis`来全局看一下文件监听的整个调用链，可以注意到有一个`register`的函数调用如下：
![fsnotify](/images/fsnotify.webp)

关键代码如下
```
// register events with the queue
func register(kq int, fds []int, flags int, fflags uint32) error {
	changes := make([]unix.Kevent_t, len(fds))

	for i, fd := range fds {
		// SetKevent converts int to the platform-specific types:
		unix.SetKevent(&changes[i], fd, unix.EVFILT_VNODE, flags)
		changes[i].Fflags = fflags
	}

	// register the events
	success, err := unix.Kevent(kq, changes, nil, nil)
	if success == -1 {
		return err
	}
	return nil
}
```
此处能够注意到`Kevent`函数，到此处可以打住了（后绪的内容内容涉及cgo、动态链接细节，那是一块内容，跟此文关系不大），这是unix下基于kqueue的内核事件的系统调用。
至此，我们有了大致的脉络：
1. 监听kevent事件（在`NewWatcher`起一个goroutine专门读取kevent， `100ms`一次）
2. 注册文件监控
3. `main`中的goroutine即为对监听事件做出响应

其实看到这里，关键点就在于kevent了，跟操作系统内核打交道，比较有意思的一点是kevent的注册跟监底层调用是同一个函数，即：
```go
func Kevent(kq int, changes, events []Kevent_t, timeout *Timespec) (n int, err error) {
	var change, event unsafe.Pointer
	if len(changes) > 0 {
		change = unsafe.Pointer(&changes[0])
	}
	if len(events) > 0 {
		event = unsafe.Pointer(&events[0])
	}
	return kevent(kq, change, len(changes), event, len(events), timeout)
}
```
只不过在注册监听的时候参数`events`与`timeout`都为`nil`。用vim打开`/tmp/foo`并增加一行，程序运行结果为下：
```
2020/12/06 16:25:45 event: "/tmp/foo": CHMOD
2020/12/06 16:25:45 event: "/tmp/foo": WRITE
2020/12/06 16:25:45 modified file: /tmp/foo
2020/12/06 16:25:45 event: "/tmp/foo": CHMOD
```

总结，两个goroutine，一个读，一个写，这是典型的顾客-消费者模型，关键在于对操作系统kevent的运用。

参考：
- https://prometheus.io/docs/prometheus/latest/configuration/configuration/
- https://zh.wikipedia.org/wiki/Kqueue
- https://github.com/fsnotify/fsnotify
- https://github.com/TrueFurby/go-callvis
- https://www.freebsd.org/cgi/man.cgi?query=kevent&sektion=2