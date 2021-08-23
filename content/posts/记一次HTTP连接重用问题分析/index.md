---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "记一次HTTP连接重用问题分析"
subtitle: ""
summary: ""
authors: []
tags: []
categories: []
date: 2021-02-20T12:24:21+08:00
lastmod: 2021-02-20T12:24:21+08:00
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Focal points: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight.
image:
  caption: ""
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---
最近新发现了一个开源项目叫[pyroscope](https://pyroscope.io/):一个开源持续Profiling平台。
![image.png](pyroscope.png)

之所以关注到这个开源项目跟我以前的一个想法有一些契合，所以就先照着官方文档，写了个样例试用
```
package main

import (
	"github.com/gin-gonic/gin"
	"github.com/pyroscope-io/pyroscope/pkg/agent/profiler"
)

func main() {
	profiler.Start(profiler.Config{
		ApplicationName: "backend.purchases",
		ServerAddress:   "http://localhost:4040",
	})

	r := gin.Default()
	r.GET("/ping", func(c *gin.Context) {
		c.JSON(200, gin.H{
			"message": "pong",
		})
	})
	r.Run()
}
```
先把服务跑起来再说，结果却发现Agent上送Profiler经常会有EOF错误，这让我觉得有点尴尬，看到Issues上有人提了这个问题，在好奇心驱使下，准备看看怎么回事

## 抓包分析
用wireshark抓包看了一下
![image.png](wireshark.png)

发现竟然是服务端在先关闭连接，此时我还未看代码，直接上此类服务应该用长连接才对，翻看服务端代码也是常规写法
```go
s := &http.Server{
    Addr:           ctrl.cfg.Server.ApiBindAddr,
    Handler:        mux,
    ReadTimeout:    10 * time.Second,
    WriteTimeout:   10 * time.Second,
    MaxHeaderBytes: 1 << 20,
    ErrorLog:       golog.New(w, "", 0),
}
```
Agent端了也是默认长连接的
```go
&http.Client{
    Transport: &http.Transport{
        MaxConnsPerHost: cfg.UpstreamThreads,
    },
    Timeout: cfg.UpstreamRequestTimeout,
}
```
所以其实两端都是支持长连接的，但连接确实是首先由服务端关闭的，这不合理啊，回头再来看抓包信息，Agent发送了[FIN, ACK]以后，还发了一次POST请求，正常情况Server端应该回一个ACK，不过由于经过了[FIN]->[FIN, ACK]此时服务端已经处于FIN_WAIT_1状态了，正等对端回ACK和FIN，不过比较巧的是刚好Agent端此时发关了一个POST请求，此时服务端只能收数据，不能发送数据，所以服务端发回了一个RST
![image.png](conn.png)

## 原因是什么？
现象分析完了，那为什么会出现这种情况呢？网上看到一些此类问题解决办法是客户端处理POST请求直接Close关掉连接，这个就没法复用连接了，而且场景不同，根本不应该这么暴力操作，还是要具体问题具体分析的。关注一个小细节，在Agent发起[SYN]建立连接到Server发起[FIN]关注连接时间间隔正好是**10s**，这个时间与Server的读写超时时间相同，而Agent的上送Profiler的默认时间间隔也是**10s**，这之间会不会有什么关系？因为理想情况至少客户端是应该复用这个连接的，直觉上应该是Agent端关闭连接才对的。果不其然，在[`server.go`](https://github.com/golang/go/blob/release-branch.go1.14/src/net/http/server.go#L2560)中找到了线索：
```go
	// IdleTimeout is the maximum amount of time to wait for the
	// next request when keep-alives are enabled. If IdleTimeout
	// is zero, the value of ReadTimeout is used. If both are
	// zero, there is no timeout.
	IdleTimeout time.Duration
```
服务端本意是想复用连接的，但是并没有设置`IdleTimeout`，但是有设置`ReadTimeout`为10s秒，这正好是Agent端上送Profiler的间隔时间，所以很快就破案了，真是好巧不巧的，这个时间点卡的可真准，其实一般情况服务端这么设置也没问题，因为长连接通常适用于并发调用，以Agent端的调用频率并不高，而且也没有并发，所以将服务端`IdleTimeout`设置为魔数30s，就没有再出现EOF的错误，然后我提了PR，很快就合入了主干。

## 总结
一开始上网搜解决方案，发现完全不是那么回事，果然是人云亦云，每个人给出的上下文不一样，解决方案自然有差别。其实用wireshark抓一下包就能找到线索，大胆猜测，小心求证，数据总不会骗人的，源码也静待剖析。

参考：
- https://colobu.com/2016/07/01/the-complete-guide-to-golang-net-http-timeouts/
