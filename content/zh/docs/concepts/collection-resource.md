---
title: 聚合资源(Collection Resource)
weight: 10
---

Clusterpedia 为了能够一次性获取多个类型的资源，在单个资源类型的基础上提供了一种新的资源 —— `聚合资源（Collection Resource）`。

`聚合资源`是由不同的资源类型组合而成，可以对这些资源类型进行统一的检索和分页。

**具体支持哪些聚合资源是由`存储层`来决定的**，例如 `默认存储层` 支持 `workloads` 和 `kuberesources` 两种聚合资源。
```bash
kubectl get collectionresources
```
```
# 输出:
NAME            RESOURCES
workloads       deployments.apps,daemonsets.apps,statefulsets.apps
kuberesources   *,*.admission.k8s.io,*.admissionregistration.k8s.io,*.apiextensions.k8s.io,*.apps,*.authentication.k8s.io,*.authorization.k8s.io,*.autoscaling,*.batch,*.certificates.k8s.io,*.coordination.k8s.io,*.discovery.k8s.io,*.events.k8s.io,*.extensions,*.flowcontrol.apiserver.k8s.io,*.imagepolicy.k8s.io,*.internal.apiserver.k8s.io,*.networking.k8s.io,*.node.k8s.io,*.policy,*.rbac.authorization.k8s.io,*.scheduling.k8s.io,*.storage.k8s.io
```
`kuberesources` 包含了所有 kube 的内置资源，我们可以通过 `kuberesources` 来对所有的内置资源进行统一的过滤和检索。

**以 yaml 形式查看支持的`聚合资源`**
```bash
kubectl get collectionresources -o yaml
```
```yaml
# 输出：
apiVersion: v1
items:
- apiVersion: clusterpedia.io/v1beta1
  kind: CollectionResource
  metadata:
    creationTimestamp: null
    name: workloads
  resourceTypes:
  - group: apps
    resource: deployments
    version: v1
  - group: apps
    resource: daemonsets
    version: v1
  - group: apps
    resource: statefulsets
    version: v1
- apiVersion: clusterpedia.io/v1beta1
  kind: CollectionResource
  metadata:
    creationTimestamp: null
    name: kuberesources
  resourceTypes:
  - group: ""
  - group: admission.k8s.io
  - group: admissionregistration.k8s.io
  - group: apiextensions.k8s.io
  - group: apps
  - group: authentication.k8s.io
  - group: authorization.k8s.io
  - group: autoscaling
  - group: batch
  - group: certificates.k8s.io
  - group: coordination.k8s.io
  - group: discovery.k8s.io
  - group: events.k8s.io
  - group: extensions
  - group: flowcontrol.apiserver.k8s.io
  - group: imagepolicy.k8s.io
  - group: internal.apiserver.k8s.io
  - group: networking.k8s.io
  - group: node.k8s.io
  - group: policy
  - group: rbac.authorization.k8s.io
  - group: scheduling.k8s.io
  - group: storage.k8s.io
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```
可以看到 `workloads` 包含了 `deployments`，`daemonsets`， `statefulsets` 三种资源。

而 `kuberesources` 则包含了 kube 内置的所有资源。

更多关于`聚合资源`的操作可以查看 [聚合资源检索](../../usage/search/collection-resource)

## 自定义聚合资源
`内置存储层`未来会支持`自定义聚合资源`，允许用户随意组合任意的资源类型。
