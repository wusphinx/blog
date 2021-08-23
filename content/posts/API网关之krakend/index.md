---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "API网关之krakend"
subtitle: ""
summary: ""
authors: []
tags: []
categories: []
date: 2021-07-12T00:08:09+08:00
lastmod: 2021-07-12T00:08:09+08:00
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
这一篇已经拖了好久了，以至于提笔开始写的时候，krakend项目已经改名为[lura](https://luraproject.org/)了，而且这个网关项目还进入到了CNCF中，真的是士别三日，当刮目相看了。不知不觉，我使用的这个开源项目作为团队的业务网关都已经上线了。

![lura-gateway](./lura-gateway.png)

当初我为什么要选择这个网关呢？首先团队跟外部沟通的协议为HTTP，有一些鉴权需求，本身业务量并不大，需要支持自定义插件的，最好是用golang写的，便于维护。从功能上来讲，其实nginx非常符合我的需求，除了技术栈，我只用了几天时间调研试用，就迅速确定了krakend作为API网关的框架，使用简单，[性能还很强](https://www.krakend.io/docs/benchmarks/api-gateway-benchmark/)的样子。当然，从业务量上来讲，性能不是第一优先级，我需要解决的是从无到有的问题。

我所在的团队属于初创，后端这部分有好几个人在写，功能上重复，每个人都跟自己的业务前端对接，部门所做的几个功能都挺分散的，再加上权限控制的需求愈发强烈，网关的必要性就凸显出来了。短短不到百行代码，就能创建一个API网关，这是我选择的主要原因。

```golang
package main

    import (
        "flag"
        "log"
        "os"

        "github.com/luraproject/lura/config"
        "github.com/luraproject/lura/logging"
        "github.com/luraproject/lura/proxy"
        "github.com/luraproject/lura/router/gin"
    )

    func main() {
        port := flag.Int("p", 0, "Port of the service")
        logLevel := flag.String("l", "ERROR", "Logging level")
        debug := flag.Bool("d", false, "Enable the debug")
        configFile := flag.String("c", "/etc/lura/lura.json", "Path to the configuration filename")
        flag.Parse()

        parser := config.NewParser()
        serviceConfig, err := parser.Parse(*configFile)
        if err != nil {
            log.Fatal("ERROR:", err.Error())
        }
        serviceConfig.Debug = serviceConfig.Debug || *debug
        if *port != 0 {
            serviceConfig.Port = *port
        }

        logger, _ := logging.NewLogger(*logLevel, os.Stdout, "[LURA]")

        routerFactory := gin.DefaultFactory(proxy.DefaultFactory(logger), logger)

        routerFactory.New().Run(serviceConfig)
    }
```
不过，看似简洁的背后，其实是一把双刃剑，很快我就遇到了几个问题。

## krakend不支持websocket代理
这个其实也不能叫做问题，因为我使用的是开源社区版本，企业版是支持的，我在[issues](https://github.com/devopsfaith/krakend-ce/issues/84#issuecomment-666330448)中看到了关于这个问题的回答，老实说，有点心酸：
>We are developing and maintaining the project and giving support for free. You can extend and customize it all you want, for free. Do you want us to move to other (payed) projects and let KrakenD die? I'm sure you'll understand that engineers (good or bad) are people and we need to eat and pay bills too

## krakend配置文件不支持正则表达式
如果你想使用正则代理接口如`^/ping/.*/pong$`，抱歉，不支持，你只能按以下格式写
```json
{
  "endpoint": "/myserviceA/{level1}/{level2}/{level3}",
  "backend": [
    {
      "host": [ "http://serverA/" ],
      "url_pattern": "/{level1}/{level2}/{level3}"
    }
  ]
},
{
  "endpoint": "/myserviceB/{level1}/{level2}/{level3}",
  "backend": [
    {
      "host": [ "http://localhost:8060" ],
      "url_pattern": "/{level1}/{level2}/{level3}"
    }
  ]
}
```
这样写你才能匹配地到`/myserviceA/a/b/c`，虽然有点难看，但勉强也能接受，只要多码点字而已，具体讨论请查看此处[issues](https://github.com/devopsfaith/krakend-ce/issues/30#issuecomment-413629959)

## krakend不支持同时配置多个Method
这个意思就是如果`/myserviceA/{level1}/{level2}/{level3}`这个URI有`POST`、`PUT`、`GET`三个方法，你得分别配置三次
```json
{
  "endpoint": "/myserviceA/{level1}/{level2}/{level3}",
  "method": "POST",
  "backend": [
    {
      "host": [ "http://serverA/" ],
      "url_pattern": "/{level1}/{level2}/{level3}"
    }
  ]
},
{
  "endpoint": "/myserviceA/{level1}/{level2}/{level3}",
  "method": "PUT", 
  "backend": [
    {
      "host": [ "http://serverA/" ],
      "url_pattern": "/{level1}/{level2}/{level3}"
    }
  ]
},
{
  "endpoint": "/myserviceA/{level1}/{level2}/{level3}",
  "method": "GET", 
  "backend": [
    {
      "host": [ "http://serverA/" ],
      "url_pattern": "/{level1}/{level2}/{level3}"
    }
  ]
},
```
是不是感觉心态要爆炸，这得写的多繁琐啊。


## krakend配置文件模板难用
如以上示例所见，krakend的配置管理是个大难题，通常我们需要一个模板来简化配置，无奈官方提供的模板功能太弱且难用，我在公司用[apollo](https://github.com/ctripcorp/apollo)作为配置中心，
key为`config`，value就是krakend的json配置文件，因为配置较多，超过了value的长度限制，不得已，只能拆，根据host拆成不同的文件，然后用patch的方式再将json文件合并成一个，也不是不能用，至少比官方模板要好一点点吧，不过核心问题还是配置文件不能简化，这真的是很伤脑筋啊。

## HTTP响应配置
一开始想当然地觉着使用krakend代理的http请求响应是透传的，但偏偏不是这样的，默认情况是只要是返回的http code大于400，krakend返回的就是500，要想做到透传，需要增加两次`no-op`配置，如下所示：
```
{
    "endpoint": "/auth/login",
    "output_encoding": "no-op",
    "backend": [
        {
            "encoding": "no-op",
            "host": [ "localhost:8080" ],
            "url_pattern": "/__debug/login"
        }
    ]
}
```
[点此查看文档](https://www.krakend.io/docs/endpoints/no-op/)
正如你所看到的那样，配置文件在不知不觉中越变越长……

---
说了那么多槽点，总得说点优点吧，毕竟我也是给这个项目做出过贡献的人，优点我总结了两点
1. 简单，这个在开篇也说了
2. 无状态，不需要数据库，貌似还不支持动态更新配置

客观来说，因为简单，所以接入门槛自然很低，熟悉gin框架的情况下，自定义插件也非常容易，适合轻量型网关场景，确实解决了团队网关从无到有这个问题，根据使用情况来看，目前最大的问题在于配置文件过于冗长，这个是先天劣势，不好弥补；再就是不支持websocket。但是瑕不掩瑜，krakend自有其适用的场景和其存在的价值。哦，不过呢，随着团队业务的发展，后期大概率是要换掉krakend了。
