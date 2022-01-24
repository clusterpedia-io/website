---
title: 聚合资源(Collection Resource)
weight: 10
---

Clusterpedia 为了能够一次性获取多个类型的资源，在单个资源类型的基础上提供了一种新的资源 —— `聚合资源（Collection Resource）`。

`聚合资源`是由不同的资源类型组合而成，可以对这些资源类型进行统一的检索和分页。

**具体支持哪些聚合资源是由`存储层`来决定的**，例如 `内置存储层` 暂时只支持 `workloads` 这一种聚合资源，用来表示工作负载。
```bash
kubectl get collectionresources
```
```
# 输出:
NAME        RESOURCES
workloads   deployments.apps,daemonsets.apps,statefulsets.apps
```

以 yaml 形式查看支持的`聚合资源`
```bash
kubectl get collectionresources -o yaml
```
```yaml
# 输出：
apiVersion: v1
items:
- apiVersion: pedia.clusterpedia.io/v1alpha1
  kind: CollectionResource
  metadata:
    creationTimestamp: null
    name: workloads
  reosurceTypes:
  - group: apps
    resource: deployments
    version: v1
  - group: apps
    resource: daemonsets
    version: v1
  - group: apps
    resource: statefulsets
    version: v1
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```
可以看到 `workloads` 包含了 `deployments`，`daemonsets`， `statefulsets` 三种资源。

更多关于`聚合资源`的操作可以查看 [聚合资源检索](../../usage/search/collection-resource)

## 自定义聚合资源
`内置存储层`未来会支持`自定义聚合资源`，允许用户随意组合任意的资源类型。
