---
title: rock k8s with k3d
date: 2019-10-23 19:15:12
categories: 
- k8s
tags:
- k8s
- 容器化
---

## k3d是什么?

> Little helper to run Rancher Lab's k3s in Docker

k3s是一个轻量级的Kubernetes发行版，小到什么程度，运行起来只占用512M的内存（在我的树莓4B上亲测过, 确实如此），要知道在服务器完整的部署k8s集群，单个节点的内存占用都需要近2G，实在是过于臃肿。在学习k8s的过程中，不是每个人都有条件使用多节点真实机器来部署的（当然官方也提供了minikube），这里选择k3d还是因为k3s足够轻巧，对于个人学习与实践也足够友好。k3d让k3s运行在容器中，如此就可以轻松拥有一个k8s的“集群”了，废话不多说，动手。


### 准备工作
下载安装kubectl
```
# Debian/Ubuntu
apt-get update && apt-get install -y apt-transport-https
curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
apt-get update
apt-get install -y kubectl
```

### 安装
脚本安装，足够简单
```
# curl -s https://raw.githubusercontent.com/rancher/k3d/master/install.sh | bash
```

### 创建集群
创建5个工作节点的k8s"集群"
```
# k3d c -w 5
```
创建完成后查看
```
# k3d list
+-------------+------------------------------+---------+---------+
|    NAME     |            IMAGE             | STATUS  | WORKERS |
+-------------+------------------------------+---------+---------+
| k3s-default | docker.io/rancher/k3s:v0.9.1 | running |   5/5   |
+-------------+------------------------------+---------+---------+
```
看一下节点角色
```
# kubectl get no
NAME                       STATUS   ROLES    AGE     VERSION
k3d-k3s-default-worker-2   Ready    worker   3h56m   v1.15.4-k3s.1
k3d-k3s-default-worker-1   Ready    worker   3h56m   v1.15.4-k3s.1
k3d-k3s-default-worker-4   Ready    worker   3h56m   v1.15.4-k3s.1
k3d-k3s-default-worker-3   Ready    worker   3h56m   v1.15.4-k3s.1
k3d-k3s-default-server     Ready    master   3h56m   v1.15.4-k3s.1
k3d-k3s-default-worker-0   Ready    worker   3h56m   v1.15.4-k3s.1
```
一个主节点，5个负载节点的配置，还未发现k3d能够支持多个主节点的创建方式，目前k3s也还未支持master的高可用，不过因为是实验用途，倒是没啥大问题。


### pause镜像问题
在国内，由于众所周知的原因，必然会遇到以下问题
```
...
failed to pull image "k8s.gcr.io/pause:3.1": failed to resolve image "k8s.gcr.io/pause:3.1"
...
```
在k8s中，Pod是最小的调度单元，一个Pod中可以有多个容器共存，类似于多个容器运行在一台虚拟主机上，这些容器共享Linux的命名空间，而这个功能由pause容器提供。所以，下载不了pause镜像，就玩不转k8s了。微软提供了k8s的镜像代理服务，对微软的印象大好了。
```
# docker pull gcr.azk8s.cn/google-containers/pause:3.1
# docker tag gcr.azk8s.cn/google-containers/pause:3.1 k8s.gcr.io/pause:3.1
```

然后将pause镜像导入各个“节点”，这样各个节点就可以直接使用“本地”pause镜像从而避免`docker pull k8s.gcr.io/pause:3.1`
```
# k3d i k8s.gcr.io/pause:3.1
INFO[0000] Saving images [k8s.gcr.io/pause:3.1] from local docker daemon...
INFO[0000] Saved images to shared docker volume
INFO[0000] Importing images [k8s.gcr.io/pause:3.1] in container [k3d-k3s-default-server]
INFO[0001] Importing images [k8s.gcr.io/pause:3.1] in container [k3d-k3s-default-worker-4]
INFO[0001] Importing images [k8s.gcr.io/pause:3.1] in container [k3d-k3s-default-worker-3]
INFO[0002] Importing images [k8s.gcr.io/pause:3.1] in container [k3d-k3s-default-worker-2]
INFO[0002] Importing images [k8s.gcr.io/pause:3.1] in container [k3d-k3s-default-worker-1]
INFO[0003] Importing images [k8s.gcr.io/pause:3.1] in container [k3d-k3s-default-worker-0]
INFO[0003] Successfully imported images [k8s.gcr.io/pause:3.1] in all nodes of cluster [k3s-default]
INFO[0003] Cleaning up tarball
INFO[0003] Deleted tarball
INFO[0003] ...Done
```

现在， rock k8s with k3d!

参考：

- https://github.com/rancher/k3d
- https://k3s.io
- https://minikube.sigs.k8s.io/
- https://github.com/Azure/container-service-for-azure-china/blob/master/aks/README.md