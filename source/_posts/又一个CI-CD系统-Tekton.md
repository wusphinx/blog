---
title: 又一个CI/CD系统-Tekton
date: 2020-11-19 09:23:38
categories: 
- DevOps
tags:
- DevOps
---

# Tekton是什么？

>Tekton 是一个强大且灵活的 Kubernetes 原生开源框架，可用于创建持续集成和交付 (CI/CD) 系统。该框架可让您跨多个云服务商或本地系统进行构建、测试和部署，而无需操心基础实现详情。

从定义可知，Tekton是一种新的CI/CD系统，正如常见的Jenkins、Drone、GitLab CI做的事情一样，特殊之处在于它是生于Kubernetes，长于Kubernetes。

# 安装Tekton
因为Tekton运行环境依赖于k8s，所以我们首先必须得有一个k8s环境，笔者本地使用的是minikube，如下本地启动一个k8s
```
minikube start -p k8s -v10 --image-mirror-country cn --iso-url=https://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/iso/minikube-v1.6.0.iso  --image-repository registry.cn-hangzhou.aliyuncs.com/google_containers --vm-driver=hyperkit --registry-mirror=https://dq4otk2m.mirror.aliyuncs.com --kubernetes-version=1.16.0
```
集群启动以后就可以安装Tekton，按照官网的描述，可按如下安装
```
kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
```
不过，因于Tekton需要依赖众多gcr站点镜像，安装过程需要先预先准备好这些镜像，笔者使用github的action将这些镜像转移到了阿里云的镜像中心，并将Tekton的安装文件重新换成了阿里云的镜像地址，项目地址为：

https://github.com/wusphinx/tektoncd-cn

国内码云地址为：

https://gitee.com/wucentaur/tektoncd-cn

项目目录如下：
```
├── LICENSE
├── README.md
├── example
│   ├── pipelineresource.yaml
│   ├── task-test.yaml
│   └── taskrun.yaml
└── tekton
    ├── tekton-dashboard-release.yaml
    └── tekton.yaml
```
如此，可以通过`kubectl apply -f tekton/`完成安装，顺带也把Tekton的dashboard也安装了，检查安装状态
```
➜  ~ kubectl get pods -n tekton-pipelines
NAME                                          READY   STATUS    RESTARTS   AGE
tekton-dashboard-5dcb9688bd-pls8b             1/1     Running   1          18h
tekton-pipelines-controller-6564b64ff-7gwbc   1/1     Running   1          18h
tekton-pipelines-webhook-5d6b9646b9-mr7jl     1/1     Running   1          18h
```

# 使用
既然是CI/CD系统，那流水线一定是必不可少的，前面说到Tekton生于k8s，所以才有了CRD定义的流水线，我们先定义获取代码仓库的资源
```yaml
apiVersion: tekton.dev/v1alpha1
kind: PipelineResource
metadata:
  name: tekton-example
spec:
  type: git
  params:
    - name: url
      value: https://github.com/wusphinx/multistage
    - name: revision
      value: main
```
简洁明了，如果代码仓库是gitlab，用gitlab ci来做，这一步是隐式的，用Jenkins的话，没有与gitlab集成的情况下，也需要显示配置。
定义流水线任务（只是为了演示，当然你可以自定义其它任务）
```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: test
spec:
  resources:
    inputs:
    - name: repo
      type: git
  steps:
  - name: run-test
    image: golang:1.13-alpine
    workingDir: /workspace/repo
    env:
      - name: "GO111MODULE"
        value: "on"
      - name: "GOPROXY"
        value: "https://goproxy.cn,direct"
    command: ["go"]
    args: ["mod", "download"]
```
任务的核心是对*某个*golang项目执行`go mod download`操作，`PipelineResource`跟`Task`缺少一纽带联系起来，这个纽带就是`TaskRun`
```yaml
apiVersion: tekton.dev/v1beta1
kind: TaskRun
metadata:
  name: testrun
spec:
  taskRef:
    name: test
  resources:
    inputs:
    - name: repo
      resourceRef:
        name: tekton-example
```
示例都准备好了，可以直接执行
```
➜  ~ kubectl apply -f example/
pipelineresource.tekton.dev/tekton-example created
task.tekton.dev/test created
taskrun.tekton.dev/testrun created
```
可通过dashboard查看任务执行结果如下所示：

![tekton](/images/tekton.png)

回头来看资源的作用就很容易理解了
- PipelineResource： 流水线的资源，示例就是指代码仓库了
- Task： 流水线要执行的任务与步骤定义
- TaskRun：将资源与任务做关联并执行

这么看若要完成一个流水线，至少要定义三类资源才行，这样代码跟CI/CD是分离的，需要在Tekton平台上做这些配置。当然因为Tekton本身就在k8s集群中，可以用尽k8s的资源调度优势，不过gitlab runner也可以安装在k8s中，从使用体验上来讲Tekton的优势并不明显，不过，这也是因为gitlab runner与gitlab是一家天然集成的缘故，Tekton的方式就是在做解耦，并无不妥。

# 结论
如此看来，Tekton是另一个CI/CD系统，抽象粒度很细，从实际使用效果来看并未带来明显的好处。最大的优势在于原生支持k8s，不过其它CI/CD平台也在向k8s靠拢，这个优势也并不十分突出。一点浅见，欢迎拍砖。

参考：
- https://cloud.google.com/tekton
- https://www.infoq.cn/article/arayxto19bd6avbmxfqz
- https://www.qikqiak.com/post/create-ci-pipeline-with-tekton-1/