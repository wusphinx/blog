---
# Documentation: https://wowchemy.com/docs/managing-content/

title: "Go应用编译错误之'invalid Pseudo Version'"
subtitle: ""
summary: ""
authors: []
tags: []
categories: []
date: 2021-08-02T22:43:30+08:00
lastmod: 2021-08-02T22:43:30+08:00
featured: false
draft: true

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
最近在公司发布平台部署go应用的时候，出现了一个很诡异的错误，整个发布流程发下
1. 从gitlab仓库拉取代码
2. 使用构建镜像执行`go build`
构建需要拉取go的依赖，我使用的版本是`golang:1.15.6`， 出现的错误是
```
2021-07-29T10:09:22.159081359Z [INFO] go: gitlab.com/my/example@v0.0.14-0.20210722061107-3a3a22e36de8: invalid pseudo-version: git fetch --unshallow -f origin in /home/docker/go/pkg/mod/cache/vcs/a62f5b800fd93278735bde0e11e509cd45e79d15398a54c789ea65bfd7ad6b50: exit status 128:
```
而且这是一个必现的问题（我之前也遇到类似的错误，不过重试几次就好了，所以一直以为是公司gitlab有问题，就没太在意）。
令人费解的地方在于`gitlab.com/my/example@v0.0.14-0.20210722061107-3a3a22e36de8`这个版本是真实存在的，而且我在本地测试也是没问题的，考古一番后发现有人遇到类似的情况，所有终点都跟git的版本有关，于是我顺手查了下编译镜像所使用的git版本：`1.8.3`
果然非常老旧，将git升级到`2.30.1`就正常了。于是跟运维反馈，升级了编译镜像的git版本。不过还没完，为什么会出现这个问题呢？
