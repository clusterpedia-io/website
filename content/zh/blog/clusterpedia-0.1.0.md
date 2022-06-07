---
title: 首发｜Clusterpedia 0.1.0 四大重要功能
date: 2022-02-16
---

**Clusterpedia 第一个版本 -- Clusterpedia 0.1.0 正式发布**，即日起进入版本迭代阶段。相比于初期的 0.0.8 和 0.0.9-alpha，0.1.0 添加了很多功能，并且做了一些不兼容的更新。

如果由 0.0.9-alpha 升级的话，可以参考 [Upgrade to Clusterpedia 0.1.0](/blog/2022/02/15/upgrade-to-clusterpedia-0.1.0/)

## 重要功能
我们先介绍一下在 0.1.0 中增加的四大重要的功能：
* **对 Not Ready 的集群进行资源检索时，增加了 Warning 提醒**
* **增强了原生 Field Selector 的能力**
* **根据父辈或者祖辈的 Owner 来进行查询**
* **响应数据携带 remaining item count**

[v0.1.0 Release Notes](https://github.com/clusterpedia-io/clusterpedia/releases/tag/v0.1.0)

## 资源检索时的 Warning 提醒
集群由于某些原因处于非 Ready 的状态时，资源通常也无法正常同步，在获取到该集群内的资源时，会通过 Warnning 提醒来告知用户集群异常，并且获取到的资源可能并不是实时准确的。

```bash
$ kubectl get pediacluster
NAME        APISERVER                   VERSION   STATUS
cluster-1   https://10.6.100.10:6443    v1.22.2   ClusterSynchroStop

$ kubectl --cluster cluster-1 get pods
Warning: cluster-1 is not ready and the resources obtained may be inaccurate, reason: ClusterSynchroStop
CLUSTER     NAME                                                 READY   STATUS      RESTARTS   AGE
cluster-1   fake-pod-698dfbbd5b-64fsx                            1/1     Running     0          68d
cluster-1   fake-pod-698dfbbd5b-9ftzh                            1/1     Running     0          39d
cluster-1   fake-pod-698dfbbd5b-rk74p                            1/1     Running     0          39d
cluster-1   quickstart-ingress-nginx-admission-create--1-kxlnn   0/1     Completed   0          126d
```

## 强化 Field Selector
原生 kubernetes 对于 Field Selector 的支持非常有限，默认只支持 metadata.namespace 和 metadata.name 字段的过滤，尽管一些特定的资源会支持一些特殊的字段，但是使用起来还是比较局限，操作符只能支持 =, ==, !=。

Clusterpedia 在兼容原生 Field Selector 的基础上，不仅仅支持了更加灵活的字段过滤，还支持和 Label Selector 相同的操作符：!, =, !=, ==, in, notin。

例如我们可以像 label selector 一样，通过 annotations 过滤资源。
```bash
kubectl get deploy --field-selector="metadata.annotations['test.io'] in (value1, value2)"
```
[Lean More](/docs/usage/search/multi-cluster/#field-selector)


## 根据父辈或者祖辈 Owner 进行查询
Kubernetes 资源之间通常会存在一种 Owner 关系, 例如：
```yaml
apiVersion: v1
kind: Pod
metadata:
  ownerReferences:
  - apiVersion: apps/v1
    blockOwnerDeletion: true
    controller: true
    kind: ReplicaSet
    name: fake-pod-698dfbbd5b
    uid: d5bf2bdd-47d2-4932-84fb-98bde486d244
```
Clusterpedia 不仅支持根据 Owner 查询，还支持对 Owner 进行辈分提升来根据祖辈或者更高辈分的 Owner 来检索资源。

例如可以通过 Deployment 获取相应的所有 pods。
> 当前暂时只支持通过Owner UID 来查询资源, 使用 Owner Name 来进行查询的功能尚在讨论中，可以在 issue: Support for searching resources by owner 参与讨论。
> v0.2.0 中已经支持通过 Owner name 进行查询

```bash
$ DEPLOY_UID=$(kubectl --cluster cluster-1 get deploy fake-deploy -o jsonpath="{.metadata.uid}")
$ kubectl --cluster cluster-1 get pods -l \
    "internalstorage.clusterpedia.io/owner-uid=$DEPLOY_UID,\
     internalstorage.clusterpedia.io/owner-seniority=1"
```
[Lean More](/docs/usage/search/multi-cluster/#search-by-parent-or-ancestor-owner)

## 响应数据内携带剩余资源数量
在一些 UI 场景下，往往需要获取当前检索条件下的资源总量。

Kubernetes 响应的 ListMeta 中 RemainingItemCount 字段表示剩余的资源数量。
```go
type ListMeta struct {
    ...

        // remainingItemCount is the number of subsequent items in the list which are not included in this
        // list response. If the list request contained label or field selectors, then the number of
        // remaining items is unknown and the field will be left unset and omitted during serialization.
        // If the list is complete (either because it is not chunking or because this is the last chunk),
        // then there are no more remaining items and this field will be left unset and omitted during
        // serialization.
        // Servers older than v1.15 do not set this field.
        // The intended use of the remainingItemCount is *estimating* the size of a collection. Clients
        // should not rely on the remainingItemCount to be set or to be exact.
        // +optional
    RemainingItemCount *int64 `json:"remainingItemCount,omitempty" protobuf:"bytes,4,opt,name=remainingItemCount"`
}
```
复用 ListMeta.RemainingItemCount，通过简单计算便可以获取当前检索条件下的资源总量: **total = offset + len(list.items) + list.metadata.remainingItemCount**
> 该功能需要搭配分页功能一起使用

```bash
$ kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?withRemainingCount&limit=1" | jq
{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "remainingItemCount": 24
  },
  "items": [
    ...
  ]
}
```
[Lean More](/docs/usage/search/multi-cluster/#response-with-remaining-count)
