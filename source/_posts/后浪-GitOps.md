---
title: 后浪-GitOps
date: 2020-11-05 10:32:40
categories: 
- DevOps
tags:
- DevOps
---


说到GitOps就一定要提DevOps了，如果说DevOps是前浪，那GitOps就是后浪了。为了表达对前辈的尊重，我们再次回顾一下DevOps：

>DevOps（Development和Operations的组合词）是一种重视“软件开发人员（Dev）”和“IT运维技术人员（Ops）”之间沟通合作的文化、运动或惯例。透过自动化“软件交付”和“架构变更”的流程，来使得构建、测试、发布软件能够更加地快捷、频繁和可靠。

诚然，DevOps是一种讲究合作的文化，一定程序上打破了开发、运维、测试等软件开发的职能界限，让整个软件开发的流程更快，以此为标尺，我们可以度量一个公司是存在DevOps文化，是否在推动、践行DevOps文化。

回归正题，说到浪，没有前浪，哪来后浪，没有DevOps， GitOps也只是空谈，那么GitOps到底是翻的是哪种浪花？

GitOps的思想源自于一家叫做Weaveworks的公司，这家公司开源了一个名为Flux的GitOps的Operator，它运行在Kubernetes或OpenShift集群内。作用是提供自动监控，保证Kubernetes集群的状态与Git中提供的配置相匹配。

但GitOps不仅仅是Flux，也不仅仅是ArgoCD等同类工具。得益于从DevOps以及可靠性工程的思想经验，GitOps已经成为云原生开发的最新热点趋势。

如果它不是简单的工具，那么GitOps到底是什么？采用它所倡导的最佳实践会给DevOps团队及其构建的应用带来哪些优势？

# GitOps到底是什么？

Weaveworks将GitOps总结为Kubernetes和其他用于云原生开发的技术的一种运营模式，

* 提供一套最佳实践，为容器化集群和应用统一部署、管理和监控。
* 提供了一条端到端的CI/CD管道以及Git工作流。

![image.png](https://upload-images.jianshu.io/upload_images/44480-20616a21c53e7fcf.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


对GitOps的扩展定义是，它是一个为云原生应用实现持续部署的标准，它注重以开发者为中心的基础设施运营体验。通过使用开发者已经熟悉的工具--Git以及持续部署工具（可选工具很多，比如我们正在用的Jenkins，但我推荐GitLab）--来实现Kubernetes集群管理及应用交付。

Git担负起声明式基础设施和应用的唯一定义工具。任何Git和K8s集群中实际运行的内容之间的差异都会被监测到。然后Kubernetes控制器可以回滚集群以与Git相匹配，或自动更新它。

将Git置于交付管道的中心，允许开发人员使用他们熟悉的工具（不用重新学习，没有认知负担）进行拉取请求。一个自动化的流程将仓库中的描述状态与生产环境进行匹配。 GitOps被描述为 "在生产环境中管理应用程序的控制器"。

# GitOps原则分解

GitOps管理Kubernetes集群的工作流程基于以下4个原则。

#1 声明式描述整个系统

GitOps与包括Kubernetes在内的云原生工具经常代表的声明式系统合作。声明式系统可以被视为代码，因为配置是由事实而不是指令来保证的。当一个应用程序用声明的方式在Git中进行版本化时，那它就一定是明确的-这就是GitOps的核心所在。

由于声明或者说是定义真实明确的记录在Git中，应用程序可以很容易地从Kubernetes部署或者回滚。在最坏的情况下，集群的基础架构可以从Git仓库中准确而快速地恢复。

#2 Git决定了系统状态

如果生产版本的声明（K8s中的资源部署）与Git版本（声明式定义）有出入，就应该发出警报。如果需要回滚，简单的Git还原就能恢复正确的系统声明。

Git强大的安全性意味着代码的作者和出处始终是可追踪的，这意味着系统的每一次变更都有迹可寻。

#3 自动化部署

以Git作为生产集群必须准确反映声明的状态，对该状态的更改可以自动更新部署。由于状态定义存在于生产环境之外的Git中，所以系统变更时不需要集群凭证。所以，GitOps可以实现做什么和怎么做的分离。

#4 软件代理如果出现分歧就会报警

如果生产集群的实际情况与你所期望的系统软件不一致，可以发出警报。这样就能整个系统的自我修复，包括在发生人为错误的情况下。


# GitOps工作流

![image.png](https://upload-images.jianshu.io/upload_images/44480-b3660d76c74e64f8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


# 为什么要在云原生开发中采用 GitOps？

我们已经介绍了GitOps的特点，但具体的商业案例对系统的好处是什么？你很难找到一种持续部署技术或方法论不承诺更快、更频繁的部署（真实需求就是快速频繁的迭代和部署）。GitOps也不例外，但有很多东西可以支持这个承诺。

自动交付管道会将Git上的变更推送到你的基础架构中。GitOps更进一步。它使用像 Weaveworks 的 Flux 这样的工具来自动比较集群的状态和 Git 配置。集群中的操作者会触发Kubernetes内部的部署，从而使任何其他CD工具的需求变得多余。

这意味着CI不需要访问集群。每一次更新都是原子的、事务性的，并且在Git提交日志中。要么成功，要么失败。由于完全基于代码，不需要增加新的基础设施。

![image.png](https://upload-images.jianshu.io/upload_images/44480-1dfd2a5f1fbebc9e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

Git作为基础架构和应用代码的 "真相之源"，可以加快部署和可靠性，因为任何错误都会被迅速发现。这样一来，无论是回滚生产状态还是从Git上更新，都非常容易

# XOps
无论是DevOps还是GitOps亦或是AIOpx，最终的目的都是为了Ops，将软件产品提供给受众，在X这一端也可以说是部署之前有很多种方法以及工具，我们姑且将X理解为CI，那Jenkins作为这一领域的前浪，依旧发挥着能量，不过后浪GitLab的发展也是迅猛

![image.png](https://upload-images.jianshu.io/upload_images/44480-c5ac73f0e8f76b69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

公司目前使用的CI工具正是Jenkins，当我试图使用jenkins去实践DevOps甚至是GitOps,  我需要做以下几件事

1. 登录Jenkins平台上，新建一个PipeLine流水线；
2. 配置好代码仓库demo以及代码访问权限；
3. 编写Jenkinsfile；
4. 因为测试环境Jenkins暂时没有与我们的代码仓库GitLab做集成，所以我不得不找到自己的项目，新建一个webhook，指向步骤1）的任务
5. 因为需要知道流水线最终的运行结果，还需要配置任务通知

对于我而言，有以下几个问题：

1. 我给项目demo配置了Jenkins流水线，这个项目也可能有其它贡献者，如果我不写文档，他们可能并不知道这件事，结果就需要问我，如同我给其它项目贡献需要问其它人一样。
2. 其它人就算知道我配置了Jenkins流水线，总还得知道Jenkins服务的地址以及开通对应的权限

所以测试环境常规发布过程归纳起来就是：

1. 开发完成，push代码；
2. 登录Jenkins，选择分支，并完成发布
3. 关注发布结果

当然，这样做依然可以达到最终完成部署的目的。

可以设想一下，当一个新人开发者拿到代码仓库时，他没法快速知道整个CI/CD流程，但是实际上这些CI/CD流程是相对固定，是有规律可循可以用语言描述的的，是可以固化在代码仓库里面的，这就是为什么我更倾向于使用GitLab CI/CD的原因。

1. 天然与GitLab集成(与GitLab同出一家)
2. 符合Iac(基础设施即代码)的哲学

![image.png](https://upload-images.jianshu.io/upload_images/44480-dcef583873b9af79.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


CI/CD各阶段需要做的事情在.gitlab-ci.yml文件中

```yaml
.....
stages:
  - test
  - build
  - deploy-dev
  - deploy-prod
lint:
  stage: test
  image: harbor.yourprivate.com/base/golangci-lint:v1.21-alpine
  script:
    - golangci-lint run -v
  allow_failure: false
build:
  stage: build
  script:
    - docker build . -t harbor.yourprivate.com/alert-service/gitops:$CI_COMMIT_BRANCH-$CI_COMMIT_SHORT_SHA
    - docker login --username=$DOCKER_PRIVATE_USEE harbor.yourprivate.com --password $DOCKER_PRIVATE_KEY
    - docker push harbor.yourprivate.com/alert-service/gitops:$CI_COMMIT_BRANCH-$CI_COMMIT_SHORT_SHA
deploy-dev:
  stage: deploy-dev
  image: cnych/kustomize:v1.0
  before_script:
    - git remote set-url origin https://${CI_USERNAME}:${CI_TOKEN}@git.myprivate.com/wuql/gitops.git
    - git config --global user.email "gitlab@git.k8s.local"
    - git config --global user.name "GitLab CI/CD"
  script:
    - git checkout -B $CI_COMMIT_BRANCH
    - git pull origin $CI_COMMIT_BRANCH
    - cd deployment/dev
    - kustomize edit set image harbor.yourprivate.com/alert-service/gitops:$CI_COMMIT_BRANCH-$CI_COMMIT_SHORT_SHA
    - cat kustomization.yaml
    - git commit -am '[skip ci] DEV image update'
    - git push origin $CI_COMMIT_BRANCH
  only:
    - dev

deploy-prod:
  stage: deploy-prod
  image: cnych/kustomize:v1.0
  before_script:
    - git remote set-url origin https://${CI_USERNAME}:${CI_TOKEN}@git.myprivate.com/wuql/gitops.git
    - git config --global user.email "gitlab@git.k8s.local"
    - git config --global user.name "GitLab CI/CD"
  script:
    - git checkout -B $CI_COMMIT_BRANCH
    - git pull origin $CI_COMMIT_BRANCH
    - cd deployment/prod
    - kustomize edit set image harbor.yourprivate.com/alert-service/gitops:$CI_COMMIT_BRANCH-$CI_COMMIT_SHORT_SHA
    - cat kustomization.yaml
    - git commit -am '[skip ci] PROD image update'
    - git push origin $CI_COMMIT_BRANCH
  only:
    - master
  when: manual
```
忽略细节部分，这段描述可以相对容易地理解为做了以下几件事

1. Test：此处是静态检查）
2. Build：构建镜像
3. Deploy：根据分支名部署到对应环境(master分支的部署需要手动执行)

![image.png](https://upload-images.jianshu.io/upload_images/44480-7267dd120779e465.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)



虽然Weaveworks最早提出了GitOps这一理念，也有许多GitOps工具，比如flux、argocd、jenkins-X， 我们选用其中的argocd。

目前公司的CI/CD平台使用的是jenkins,  虽然也可以用，但是需要在jenkins配置任务以及仓库的代码访问权限，相对繁琐，其实公司的代码托管平台gitlab不仅仅可以用来做代码仓库，也可以做CI/CD, 因此，我自己搭建了gitlab runner将集成到公司的gitlab，项目代码目录如下

![image.png](https://upload-images.jianshu.io/upload_images/44480-7086a6e1ebf1fc17.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

其中main.go为http server代码

Dockerfile将仓库代码通过Multistage打包成开箱即用的窗口镜像

deployment为k8s部署的资源描述，dev为开发环境，prod为生产环境，对应的，我们需要在argocd的管理平台上新建两个应用

![image.png](https://upload-images.jianshu.io/upload_images/44480-ec56cebc0052ca72.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

背后就是在k8s中创建了applications.argoproj.io这个crd的实例

应用的描述如下所示

```yaml
...
spec:
  destination:
    namespace: dev
    server: https://kubernetes.default.svc
  project: default
  source:
    path: deployment/dev
    repoURL: https://git.myprivate.com/wuql/gitops.git
    targetRevision: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
...
```
当有人误将生产应用删除的时候，

```shell
kubectl delete deploy gitops -n dev
```
argocd会监测到这一情况
![图片](https://upload-images.jianshu.io/upload_images/44480-14cf26e14e2975e8.png!thumbnail?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

并重新将应用资源建立起来

![图片](https://upload-images.jianshu.io/upload_images/44480-8515b1fbe1950a5f.png!thumbnail?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

至此，算是实践了一下gitops的思想。

# 解决了什么问题？

仔细想想，如果我们不用gitops这一套思路，gitlab runner本身就在k8sk, 我可以在gitlab的ci完成后直接apply部署，这样最终也能达到部署这一目的，gitops思路很大的一点不同在于【部署】这件事由opertaor来保证，并且持续检查部署状态是否与git仓库定义的是否一致，如果说在gitlab runner中可以一气呵成完成CI/CD, 那么gitops就是将CI/CD解耦了，这样带来的好处是否大于收益，个人认为需要看基础设施是否完善，如果本身CI/CD本身都做的很好，可以尝试，但是根据笔者的观察，IaC的思路并不是那么容易被接受的，思路可以借鉴，自动化确实很美好，不过也是道阻且长，最重要还是能解决团队当时当下的具体问题。

参考

* [https://zh.wikipedia.org/wiki/DevOps](https://zh.wikipedia.org/wiki/DevOps)
* [https://www.weave.works/technologies/gitops/](https://www.weave.works/technologies/gitops/)
* [https://docs.docker.com/develop/develop-images/multistage-build/](https://docs.docker.com/develop/develop-images/multistage-build/)
* [https://argoproj.github.io/argo-cd/getting_started/](https://argoproj.github.io/argo-cd/getting_started/)
* [https://kruschecompany.com/gitops/](https://kruschecompany.com/gitops/)
* [https://www.qikqiak.com/post/gitlab-ci-argo-cd-gitops/](https://www.qikqiak.com/post/gitlab-ci-argo-cd-gitops/)
* [https://about.gitlab.com/devops-tools/jenkins-vs-gitlab/](https://about.gitlab.com/devops-tools/jenkins-vs-gitlab/)
* [https://about.gitlab.com/stages-devops-lifecycle/continuous-integration/](https://about.gitlab.com/stages-devops-lifecycle/continuous-integration/)
