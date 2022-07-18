---
title: 聚合资源(Collection Resource)
weight: 10
---

Clusterpedia 为了能够一次性获取多个类型的资源，在单个资源类型的基础上提供了一种新的资源 —— `聚合资源（Collection Resource）`。

`聚合资源`是由不同的资源类型组合而成，可以对这些资源类型进行统一的检索和分页。

**具体支持哪些聚合资源是由`存储层`来决定的**，例如 `默认存储层` 支持 `any`，`workloads` 和 `kuberesources` 两种聚合资源。
```bash
kubectl get collectionresources
```
```
# 输出:
NAME            RESOURCES
any             * 
workloads       deployments.apps,daemonsets.apps,statefulsets.apps
kuberesources   .*,*.admission.k8s.io,*.admissionregistration.k8s.io,*.apiextensions.k8s.io,*.apps,*.authentication.k8s.io,*.authorization.k8s.io,*.autoscaling,*.batch,*.certificates.k8s.io,*.coordination.k8s.io,*.discovery.k8s.io,*.events.k8s.io,*.extensions,*.flowcontrol.apiserver.k8s.io,*.imagepolicy.k8s.io,*.internal.apiserver.k8s.io,*.networking.k8s.io,*.node.k8s.io,*.policy,*.rbac.authorization.k8s.io,*.scheduling.k8s.io,*.storage.k8s.io
```
`any` 表示任意的资源，用户在使用时需要传递想要组合的 groups 或者 resources，具体使用可以参考 [使用 Any CollectionResource](../../usage/search/collection-resource#any-collectionresource)

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
    name: any
  resourceTypes: []
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
Clusterpedia 计划提供两种方式来让用户随意组合想要查询的资源类型
* **`Any CollectionResource`** —— 使用聚合资源 `any`
* **`CustomCollectionResource`** —— 自定义聚合资源

0.4 中，Clusterpedia 提供了 any collectionresource 来让用户通过传递 `groups` 和 `resources` 参数来组合不同类型的资源。

不过需要注意 any collectionresource 不能使用 kubectl 来获取，具体使用可以参考 [使用 Any CollectionResource](../../usage/search/collection-resource/use-any-collection-resource)
```bash
$ kubectl get collectionresources any
Error from server (BadRequest): url query - `groups` or `resources` is required
```

**自定义聚合资源**允许用户通过 `kubectl apply collectionresource <collectionresource name>` 来创建或者更新一个 Collection Resource，用户随意的配置 Collection Resource 的资源类型
```yaml
apiVersion: clusterpedia.io/v1beta1
kind: CollectionResource
metadata:
  name: workloads
resourceTypes:
- group: apps
  resource: deployments
- group: apps
  resource: daemonsets
- group: apps
  resource: statefulsets
- group: batch
  resource: cronjobs
```
**当前还未支持自定义聚合资源**
