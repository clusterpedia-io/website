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
kubectl get --raw="/apis/clusterpedia.io/v1beta1/collectionresources/workloads?limit=1" | jq
```
```json
# 输出
{
  "kind": "CollectionResource",
  "apiVersion": "clusterpedia.io/v1beta1",
  "metadata": {
    "name": "workloads",
    "creationTimestamp": null
  },
  "resourceTypes": [
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

## Only Metadata
我们在检索 CollectionResource 时，获取到的资源默认是全量的资源信息，但是我们通常只需要获取资源的 metadata 即可。

在检索时我们可以使用 url query —— `onlyMetadata` 来只获取资源 Metadata。
```bash
$ kubectl get --raw "/apis/clusterpedia.io/v1beta1/collectionresources/workloads?onlyMetadata=true&limit=1" | jq
{
  "kind": "CollectionResource",
  "apiVersion": "clusterpedia.io/v1beta1",
  "metadata": {
    "name": "workloads",
    "creationTimestamp": null
  },
  "resourceTypes": [
    {
      "group": "apps",
      "version": "v1",
      "kind": "Deployment",
      "resource": "deployments"
    }
  ],
  "items": [
    {
      "apiVersion": "apps/v1",
      "kind": "Deployment",
      "metadata": {
        "annotations": {
          "deployment.kubernetes.io/revision": "1",
          "shadow.clusterpedia.io/cluster-name": "cluster-example"
        },
        "creationTimestamp": "2021-09-24T10:19:19Z",
        "generation": 1,
        "labels": {
          "k8s-app": "tigera-operator"
        },
        "name": "tigera-operator",
        "namespace": "tigera-operator",
        "resourceVersion": "125073610",
        "uid": "992f9d53-37cb-4184-a004-15b278b11f79"
      }
    }
  ]
}
```

## Any CollectionResource
> `any collectionresource` 是用户自由组合资源类型的方式之一，可以通过 [自定义聚合资源](../../../concepts/collection-resource#自定义聚合资源) 来查看更多。

clusterpedia 支持一种特殊的 CollectionResource —— `any`
```bash
$ kubectl get collectionresources
NAME            RESOURCES
any             *
```

在检索 any collectionresource 时，我们必须通过 url query 来指定一组资源类型，因此我们只能通过 [clusterpedia-io/client-go](https://github.com/clusterpedia-io/client-go/pull/43) 或者 url 来访问 any collectionresource
```bash
$ kubectl get collectionresources any
Error from server (BadRequest): url query - `groups` or `resources` is required
```

`any collectionresource` 支持两个 url query —— `groups` 和 `resources`

**`groups` 和 `resources` 可以同时指定，当前是取两者并集，并且不会进行去重处理，由调用者负责去重**，未来对该行为有一些优化。

```bash
$ kubectl get --raw "/apis/clusterpedia.io/v1beta1/collectionresources/any?onlyMetadata=true&groups=apps&resources=batch/jobs,batch/cronjobs" | jq
```

#### groups
`groups` 可以指定一组资源的组和版本，多个组版本使用 `,` 分隔，组版本格式为 **< group >/< version >**，也可以不指定版本 **< group >**，*对于 /api 下的资源，可以直接使用空字符串*

示例：**groups=apps/v1,,batch** 指定了三个组 apps/v1, core, batch。

#### resources
`resources` 可以指定具体的资源类型，多个资源类型使用 `,` 分隔，资源类型格式为 **< group >/< version >/< resource>**，通过也可以不指定版本 **< group >/< resource >**。

示例：**resources=apps/v1/deployments,apps/daemonsets,/pods** 指定了三种资源 deloyments，daemonsets 和 pods。
