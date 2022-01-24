---
title: "聚合资源检索"
weight: 12
---

关于聚合资源的概念可以查看 [什么是聚合资源 （Collection Resource）](../../../concepts/collection-resource)

由于 kubectl 的限制，我们无法通过 `Label Selector` 或者其他方式来传递检索参数，
所以**建议以 URL 的方式来检索`聚合资源`**。

{{< tabs >}}

{{% tab name="URL" %}}
> **请求`聚合资源`时，由于资源数量可能会非常大，一定要搭配[分页功能](../#分页)使用。**
```bash
kubectl get --raw="/apis/pedia.clusterpedia.io/v1alpha1/collectionresources/workloads?limit=1" | jq
```
```json
# 输出
{
  "kind": "CollectionResource",
  "apiVersion": "pedia.clusterpedia.io/v1alpha1",
  "metadata": {
    "name": "workloads",
    "creationTimestamp": null
  },
  "reosurceTypes": [
    {
      "group": "apps",
      "version": "v1",
      "kind": "Deployment",
      "resource": "deployments"
    },
    {
      "group": "apps",
      "version": "v1",
      "resource": "daemonsets"
    },
    {
      "group": "apps",
      "version": "v1",
      "resource": "statefulsets"
    }
  ],
  "items": [
    {
      "apiVersion": "apps/v1",
      "kind": "Deployment",
      ...
    }
  ]
}
```

`聚合资源`的复杂检索和[多集群资源检索](../multi-cluster) 功能基本相同，只有部分操作不支持：
* 不支持根据 Owner 检索，如果需要进行根据 Owner 检索需要指定具体的资源类型，可以参考 [多集群检索](../multi-cluster#根据父辈以及祖辈-owner-查询) 和 [指定集群检索](../specified-cluster#根据父辈或者祖辈-owner-进行检索)
* 不支持在 Collection Resource 中获取具体的单个资源，因为具体资源需要指定集群和资源类型，可以使用 [获取单个资源]()
{{% /tab %}}

{{% tab name="kubectl" %}}
尽管 kubectl 无法很好的检索`聚合资源`，但是可以简单的体验一下。
> 由于无法传递分页以及其他检索条件，kubectl 会一次获取到所有的`聚合资源`，在接入了大量集群或者存在大量 `deployments`，`daemonsets`，`statefulsets` 资源时，不要使用 kubectl 来查看`聚合资源`

```bash
kubectl get collectionresources workloads
```
```
# 输出
CLUSTER     GROUP   VERSION   KIND         NAMESPACE                     NAME                                          AGE
cluster-1   apps    v1        DaemonSet    kube-system                   vsphere-cloud-controller-manager              63d
cluster-2   apps    v1        Deployment   kube-system                   calico-kube-controllers                       109d
cluster-2   apps    v1        Deployment   kube-system                   coredns-coredns                               109d
...
```

**请使用 URL 来检索`聚合资源`**
{{< /tab >}}

{{< /tabs >}}
