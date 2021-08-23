---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "API网关选型"
subtitle: ""
summary: ""
authors: []
tags: []
categories: []
date: 2021-04-27T22:30:46+08:00
lastmod: 2021-04-27T22:30:46+08:00
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
来新公司第一个月，客户端部门做后端，我应该是这个部门里面最专业的后端了吧，嗯，不能这么讲，应该是在后端领域工龄最长的人，这样描述相对客观。在这里我得知现在团队当前后端Golang项目都是客户端开发的人做的，坦白说，代码规范不太好，我随便看了几个项目，几个问题比较突出：
1. 代码未格式化，风格凌乱;
2. 有的项目存在很多无用甚至是错误的测试用例，导致我用VSCode打开项目会看到不少错误;
3. 有的项目代码仓库提交了一些不该提交的文件;
4. 数据库表结构跟代码未同步，这直接导致我需要连上数据库才能看到表的索引等内容;
5. 普遍没有做静态检查，包括`go.mod`文件也存在冗余的情况;
……

表面问题看着是不少，还未检查业务逻辑，不过看似混乱的项目竟然能够支撑近半年，没有出现大的问题，这不能不说是一个奇迹，我相信我的小伙伴们都尽力了，毕竟客户端兼服务端开发，这活可并不好干，业务能从0到1做起来已经相当不错。不过，我来了，就是要解决这些问题。

首先摆在我面前的问题是后端业务比较分散，几个人各写各的，既没有代码共用，服务也没有分层，因为是内部系统，也没有做权限验证，所谓分久必合，那我想到的是先将后端系统进行整合，统一入口，将鉴权补足，于是就有了API网关的想法，需求其实比较简单：
1. 路由
2. 鉴权
3. 支持自定义插件

其实nginx也能满足需求，不过由于技术栈的原因，就不考虑了，搜索google得到的几大流行网关如下：
1. [Kong](https://konghq.com/): 云原生微服务网关，大名鼎鼎，看了下文档，这家伙太重了，考虑到目前我还没有操作k8s的权限, pass
2. [APISIX](https://github.com/apache/apisix): 貌似是国内的团队开发，有一个web UI，看着不错的样子，不过强依赖etcd, pass
3. [traefik](https://traefik.io/)：新版本支持自定义插件开发，dashboard也很漂亮，不过文档看着有点吃力，尝试了一下，还是放弃了，pass
4. [KrakenD](https://www.krakend.io)，说它是一个API网关框架其实更合适，根据文档，我快速写出自己的网关服务，自定义插件只需要实现`HandlerFunc`即可，非常方便快捷，配置文件虽然有点繁琐，不支持通配符，但瑕不掩瑜，一天时间我就完成了团队网关服务的雏形。

当然肯定还有其它网关我没有调研到，所谓调研也比较粗浅，文档友好，能快速开发，优先满足当前需求，解决从无到有，后续业务支撑起来，业务自然会失去技术的发展。
