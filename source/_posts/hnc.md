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

### 开始

感谢HNC 幕后团队出色的工作，使得hnc已经可以在本地用 Kind 命令开发和测试，并且是如此的easy!


```bash
$ kind create cluster
$ kubectl apply -f https://github.com/kubernetes-sigs/multi-tenancy/releases/download/hnc-v0.5.1/hnc-manager.yaml

$ curl -L https://github.com/kubernetes-sigs/multi-tenancy/releases/download/hnc-v0.5.1/kubectl-hns -o ./kubectl-hns

$ chmod +x ./kubectl-hns

$ export PATH=${PWD}:${PATH}

```
