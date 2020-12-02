---
title: 后端开发视角下的CRD之一
date: 2020-11-23 13:18:34
categories: 
- k8s
tags:
- k8s
---
# CRD是什么？
Custom Resource Definitionsr的简称，它是对Kubernetes API 的扩展。打个并不严谨的比方，从所周知`Service`、`Pod`、`Deployment`等是k8s内置的资源**类型**，因为k8s使用golang开发，如果把`Pod`类比为golang中的`int`类型，那么CRD可以类比为golang中的`interface`: 抽象类型，这个比喻可能不那么恰当，只是为了方便个人理解。而关于CRD的产生过程，还有一段非常有坡的故事，可参考认文末张磊老师的的**Kubernetes API 与 Operator：不为人知的开发者战争**链接

# 如何编写自己的CRD？
对CRD有了初步了解之后，我们要如何写出自己想要的CRD呢？
就像我们写Http Server一样，有`Gin`等框架，写CRD同样也有对应的框架：kubebuilder。下面，我们一下一步来编辑自己的CRD

## 准备工作

### 准备工具
首先得自备一个k8s集群（笔者使用的是minikube）,根据自己的开发环境下载kubebuilder工具
```
os=$(go env GOOS)
arch=$(go env GOARCH)

# download kubebuilder and extract it to tmp
curl -L https://go.kubebuilder.io/dl/2.3.1/${os}/${arch} | tar -xz -C /tmp/

# move to a long-term location and put it on your path
# (you'll need to set the KUBEBUILDER_ASSETS env var if you put it somewhere else)
sudo mv /tmp/kubebuilder_2.3.1_${os}_${arch} /usr/local/kubebuilder
export PATH=$PATH:/usr/local/kubebuilder/bin
```

### 项目初始化
```
mkdir $GOPATH/src/example
cd $GOPATH/src/example
kubebuilder init --domain my.domain
```
项目结构如下所示：
```
.
├── Dockerfile
├── Makefile
├── PROJECT
├── bin
│   └── manager
├── config
│   ├── certmanager
│   ├── default
│   ├── manager
│   ├── prometheus
│   ├── rbac
│   └── webhook
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── main.go
```

### 创建API
```
kubebuilder create api --group webapp --version v1 --kind Guestbook
```
到这一步，语义理解还是非常清楚的，就个人的理解：定义了一个RESTful的URI，类型、资源名称、Group等，很像我们平时写HTTP API的感觉，事实上，我们正在做的也确实在写一个API，目录结构如下：
```
.
├── Dockerfile
├── Makefile
├── PROJECT
├── api
│   └── v1
├── bin
│   └── manager
├── config
│   ├── certmanager
│   ├── crd
│   ├── default
│   ├── manager
│   ├── prometheus
│   ├── rbac
│   ├── samples
│   └── webhook
├── controllers
│   ├── guestbook_controller.go
│   └── suite_test.go
├── go.mod
├── go.sum
├── hack
│   └── boilerplate.go.txt
└── main.go
```

### 自定义类型：Guestbook
以下即是我们新创建的CRD：Guestbook的类型结构
```
// Guestbook is the Schema for the guestbooks API
type Guestbook struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   GuestbookSpec   `json:"spec,omitempty"`
	Status GuestbookStatus `json:"status,omitempty"`
}
```
我们来跟k8s内置类型[Deployment](https://sourcegraph.com/github.com/kubernetes/kubernetes@2db51c85b948f8180b427964b43f84e517c6e35e/-/blob/staging/src/k8s.io/api/apps/v1/types.go#L254:6)对比一下
```
// Deployment enables declarative updates for Pods and ReplicaSets.
type Deployment struct {
	metav1.TypeMeta `json:",inline"`
	// Standard object metadata.
	// +optional
	metav1.ObjectMeta `json:"metadata,omitempty" protobuf:"bytes,1,opt,name=metadata"`

	// Specification of the desired behavior of the Deployment.
	// +optional
	Spec DeploymentSpec `json:"spec,omitempty" protobuf:"bytes,2,opt,name=spec"`

	// Most recently observed status of the Deployment.
	// +optional
	Status DeploymentStatus `json:"status,omitempty" protobuf:"bytes,3,opt,name=status"`
}
```
类型结构几乎是一模一样的，可以看到`metav1.TypeMeta`与`metav1.ObjectMeta`是内嵌结构体，这是类型的一些公共定义，篇幅有限，就不在此处不展开了，`Spec`是需要注意的，前面我们定义了一个资源及期接口，那`Spec`可以认为是接口的入参了，`Status`: 顾名思义，资源的状态


### 入参校验
```
type GuestbookSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Foo is an example field of Guestbook. Edit Guestbook_types.go to remove/update
	// 设定参数Foo的取值范围，也就是做入参校验

	// +kubebuilder:validation:MaxLength=10
	// +kubebuilder:validation:MinLength=1
	Foo string `json:"foo,omitempty"`
}
```
此处只有一个参数`Foo`，我们通过注解的方式设定Foo的取值范围为[1,10]


### 安装CRD到k8s集群中
```
make install 
```
将`Guestbook`这种资源类型注册到自己的k8s中以使外部系统能够访问到
部署一个`Guestbook`实例
```
kubectl apply -f config/samples/
```
这一步，以后端视角，就是调用`POST`接口新建一条"数据库"记录的，当然，k8s的数据库自然就是etcd了；如果我们将`foo`的值设为`asdfgasdfg1`，长度为11的字符串，再运行以上命令，会是什么结果呢？
```
The Guestbook "guestbook-sample" is invalid: spec.foo: Invalid value: "": spec.foo in body should be at most 10 chars long
```
可以看到入参校验生效了,

# 总结
至此，我们完成了一个CRD的最基本定义，以后端开发的视角而言，如果将CRD可以看作是一张表，这张表作为资源，这一节完成了对CRD这类资源的基本定义跟校验。


参考：
- https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/custom-resources/
- [Kubernetes API 与 Operator：不为人知的开发者战争（一）](https://mp.weixin.qq.com/s/jZLkS4SFVVXS1Mx2aG4PUA)
- [Kubernetes API 与 Operator：不为人知的开发者战争（二）](https://mp.weixin.qq.com/s/ODaOiwXMBhRFZjbdu1BFUA)
- [kubebuilder](https://book.kubebuilder.io/)
- https://github.com/wusphinx/example-crd