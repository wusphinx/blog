---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "go mod依赖包处理"
subtitle: ""
summary: ""
authors: []
tags: []
categories: []
date: 2021-03-24T16:00:12+08:00
lastmod: 2021-03-24T16:00:12+08:00
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
自从golang原生支持`go mod`以后，将golang应用依赖处理统一和标准化了，但是在使用中还是会遇到一些问题，比如以下`go.mod`
```
module github.com/wowchemy/starter-blog

go 1.14

require (
	github.com/wowchemy/wowchemy-hugo-modules/wowchemy v0.0.0-20210209220000-aa4fe0c75726 // indirect
	github.com/wowchemy/wowchemy-hugo-modules/wowchemy-cms v0.0.0-20210209220000-aa4fe0c75726 // indirect
)
```
这是我的博客使用的模板，需要依赖`github.com/wowchemy/wowchemy-hugo-modules/wowchemy`，可以看到版本号`v0.0.0-20210209220000-aa4fe0c75726`由tag+commit时间+commit哈希组成，因为`github.com/wowchemy/starter-blog`新版本存在bug，已经在`github.com/wowchemy/wowchemy-hugo-modules/wowchemy`中得到修复，所以我需要改一下依赖的版本，当然我可以去查看bug修复的日期以及commit哈希，然后拼接下go mod所支持的格式，但是总感觉不太方便，于是摸索出一种方法，通过commit哈希获取依赖的版本号，具体操作如下：
```
➜  curl  https://goproxy.cn/github.com/wowchemy/wowchemy-hugo-modules/wowchemy/@v/3cf9f6c.info
{"Version":"v0.0.0-20210215224117-3cf9f6cdeef0","Time":"2021-02-15T22:41:17Z"}
```
可以看到`v0.0.0-20210215224117-3cf9f6cdeef0`即是我需要的版本号，开式也很简单

```
                   |-----------------------依赖项---------------------|   |-哈希-|
https://goproxy.cn/github.com/wowchemy/wowchemy-hugo-modules/wowchemy/@v/3cf9f6c.info 
```
当然，如果能用信赖项的tag尽量用tag, 这也是go官方建议的做法。
