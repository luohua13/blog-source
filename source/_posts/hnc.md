---
title: 译：Hierarchical Namespace Controller (HNC):对k8s多租户特性的惊鸿一瞥
date: 2020-10-26 10:56
tags: [k8s, multitenancy]
categories: kubernetes
---

Hierarchical Namespace Controller (HNC) 将会带来一种更好的k8s多租户模型。这篇文章将要探索这个项目的当前状态以及一些有用的落地场景。

<!--more-->

Hierarchical Namespace Controller (HNC) 是google公司为了改善k8s多租户体验所驱动的一个项目。到今天为止，k8s的资源由集群级别的资源(namespace)来组织。遗憾的是，安全地将不同用户托管到同一集群中需要高度的自动化知识。HNC通过传递“parent namespace” 的配置到所属的"child namespace"的思路，尝试弥补这一遗憾。

> Hierarchical namespaces 通过创建功能强大的namespace从而使你的集群管理更加容易。例如，你能在你所属团队的team namespace下创建另外的“child namespace”,即使你没有创建集群级别的namespace的权限。这将很轻松的应用策略例如RBAC/Network Policies到你的team namespace下的所有的child namespace 中。

摘自：[multi-tenancy sig repository](https://github.com/kubernetes-sigs/multi-tenancy/tree/hnc-v0.5.1/incubator/hnc)

我们测试了HNC最新的版本(0.5.1), 可以看到很多有价值的东西。不幸的是，开发工作最近才开始，尽管社区已经讨论了很长时间。

HNC 在今年应该还不能被正式使用。

## 准备开始
你必须拥有一个 Kubernetes 的集群，同时你的 Kubernetes 集群必须带有 kubectl 命令行工具。 如果你还没有集群，你可以通过 Minikube 构建一个你自己的集群。

## 开始

感谢HNC 幕后团队出色的工作，使得hnc已经可以在本地开发和测试，并且是如此的easy!


```bash
$ kubectl apply -f https://github.com/kubernetes-sigs/multi-tenancy/releases/download/hnc-v0.5.1/hnc-manager.yaml

$ curl -L https://github.com/kubernetes-sigs/multi-tenancy/releases/download/hnc-v0.5.1/kubectl-hns -o ./kubectl-hns

$ chmod +x ./kubectl-hns

$ export PATH=${PWD}:${PATH}

```

**注意**：国内用户拉取google的镜像可能有点麻烦，可以将
gcr.io/k8s-staging-multitenancy/hnc/controller:v0.5.1 替换成：luohua13/hnc-controller
gcr.io/kubebuilder/kube-rbac-proxy:v0.4.0 替换成：luohua13/kube-rbac-proxy

然后你可以开始测试：

```bash

$ kubectl create ns hnc-parent
namespace/hnc-parent created
$ kubectl create ns hnc-child-1
namespace/hnc-child-1 created
$ kubectl create ns hnc-child-2
namespace/hnc-child-2 created

$ kubectl hns set hnc-child-1 -p hnc-parent
Setting the parent of hnc-child-1 to hnc-parent
Succesfully updated 1 property of the hierarchical configuration of hnc-child-1
$ kubectl hns set hnc-child-2 -p hnc-parent
Setting the parent of hnc-child-2 to hnc-parent
Succesfully updated 1 property of the hierarchical configuration of hnc-child-2

$ kubectl hns tree -A
default
hnc-parent
├── hnc-child-1
└── hnc-child-2
hnc-system
kube-node-lease
kube-public
kube-system
local-path-storage

```

默认情况下， parent namespace下的所属的Role 和 RoleBinding 对象会传递给 child 命名空间对象。

```bash

$ kubectl create role pod-reader --verb=get --verb=list --verb=watch --resource=pods -n hnc-parent
role.rbac.authorization.k8s.io/pod-reader created
```
所以，让我们验证下 hnc-child-{1,2} 命名空间下是否能看到相同的 role：

``` bash

$ kubectl get role -n hnc-child-1 --show-labels
NAME         CREATED AT             LABELS
pod-reader   2020-08-24T08:13:00Z   hnc.x-k8s.io/inheritedFrom=hnc-parent
$ kubectl get role -n hnc-child-2 --show-labels
NAME         CREATED AT             LABELS
pod-reader   2020-08-24T08:13:00Z   hnc.x-k8s.io/inheritedFrom=hnc-parent

```

怎么样？够酷炫吧！它能满足我们很多年梦寐以求的使用场景。

## 使用案例

kubernetes 是 SIGHUP 业务的核心部分。我们在很多大规模的公司工作，因为k8s没有多租户特性使我们遇到了很多难以解决的挑战。

在这篇文章中将展示 HNC 两种不同的使用案例。

### Namespace 自给（self provisioning）

有时开发者为了开发的工作需要创建新的 namespace。因为 namespace是集群级别的对象，集群管理员需要赋予开发者相应的权限。在赋予创建 namespace的权限之后，你（作为集群管理员）可能还想附加一些默认的配置，例如：

- NetworkPolicies 
- Roles and RoleBindings
- ResourceQuotas
- LimitRanges

否则，你将很快面临一个混乱的集群。那么你该怎么办呢？

HNC 能满足这个使用场景。你可以定义一个 parent 命名空间，所有的对象在 其 child 命名空间下， 并且不要去手动传递对象。换句话说，就是开发者不需要有任何的集群级别权限。使用 HNC时，你需要被赋予在 parent命名空间下管理 child命名空间的权限。


![setup flow](/image/hnc-1.jpeg)

关于这个场景如果你想了解更多，[follow our guide.](https://github.com/sighupio/hnc-example-use-cases/blob/master/use-cases/self-provision/README.md)

### 应用模版

之前的使用案例集中在配置命名空间的管理。接下来这个案例将更进一步。假定你是一个多层应用的owner：

- Frontend
- Backend
- DB

如果你要开发一个新的 Frontend feature，你能受益于有一个完整的 Backend 模版，用来在一个稳定和隔离的环境来测试你的 Frontend change!

在这个案例中，你可以在 parent 命名空间下定义如下公有的应用组件：

- DB Deployments and Services
- Backend Deployments and Services
- Frontend Services

![setup flow](/image/hnc-2.jpeg)

**重要提示：** 不要把  frontend deployment 放在parent 命名空间中。你需要的修改不应该在 parent 命名空间

然后创建一个 child 命名空间，将你的 修改的frontend deployment 放在这个命名空间下。
关于这个场景如果你想了解更多，[follow our guide.](https://github.com/sighupio/hnc-example-use-cases/blob/master/use-cases/application-template/README.md)


## 结论

HNC 尝试去弥补k8s的多租户特性缺失的遗憾。这个underlying的想法是一个足够好的出发点。它需要更多来自社区的爱和众多开发者的反馈。

下面是一些我们正在积极提的的issues和features:

- 将 HNCConfiguration 对象从集群级别(cluster-level)的资源转换为（parent）命名空间范围（namespace scope）的配置。另一种做法是保留集群级别的HNCConfiguration同时开发一种命名空间范围的 HNCConfiguration。

- 如果 child 命名空间中的对象和 parent 命名空间中的对象重名，则使 child 命名空间中的同名对象无效。假定你在child 命名空间定义一个名为developer的 RBAC Role, 如果你在 parent 命名空间设置了一个同名的 Role, parent 命名空间的 Role 就应该覆盖掉 child 命名空间的 Role.

- Fix 掉 parent命名空间和 child 命名空间传递集群级别的数据和属性（不可重复）时候出现的一些问题。举个栗子：Service 对象就不能被传递，因为 HNC 会复制 Service 对象的每个属性到 child 命名空间，包括 spec.ClusterIP （集群中的唯一数据）

你可以通过加入 [google 多租户特性工作组](https://github.com/kubernetes/community/blob/master/wg-multitenancy/README.md), 来及时了解 HNC 的开发过程。

## 结尾

SIGHUP 把 HNC 捐献出去的兴趣是非常浓厚的，因为很有可能在未来成为标准。在讨论这项评估期间，有很多基于k8s实现多租户特性的替代品，但是，它仍将很快在未来的某个时间点成为一个标准。

不要忘了访问我们的网站，诸如:
[SIGHUP site](https://sighup.io/)
[GitHub organization](https://github.com/sighupio)
以及本文中所有材料的来源：
[the Git repository](https://github.com/sighupio/hnc-example-use-cases)

本文译自：https://blog.sighup.io/an-introduction-to-hierarchical-namespace-controller-hnc/?utm_sq=gi975454xd
