---
title: Hierarchical Namespace Controller (HNC):对k8s多租户特性的惊鸿一瞥
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

HNC 闪亮登场！ 
