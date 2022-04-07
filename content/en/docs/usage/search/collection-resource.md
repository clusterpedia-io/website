---
title: "Collection Resource"
weight: 12
---

For collection resource, refer to [What is Collection Resource](../../../concepts/collection-resource)  

Due to the limitation of kubectl, we cannot pass search conditions through `Label Selector` or other methods, so **it is recommended to search for `Collection Resource`** by using a URL.


{{< tabs >}}

{{% tab name="URL" %}}
> **When requesting `Collection Resource`, you shall use [paging](../#paging) because the number of resources may be very large.**
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/collectionresources/workloads?limit=1" | jq
```
```json
# Output
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

The complex search of `Collection Resource` is basically the same as the function of [multi-cluster resource search](../multi-cluster), only some operations are not supported:
* Search by Owner is not supported. If you need to specify a specific resource type to search by Owner, you can refer to [`multi-cluster resource search`](../multi-cluster#query-by-parent-or-ancestor-owner) and [`specified cluster search`](../specified-cluster#search-by-parent-or-ancestor-owner)
* Getting a specific single resource in `Collection Resource` is not supported, because you shall specify cluster and type for a specific resource. In this case, you can use [Get a single resource](../specified-cluster#get-a-single-resource).
{{% /tab %}}

{{% tab name="kubectl" %}}
It is not easy to search for `Collection Resource` by using kubectl. However, you can have a try.
> kubectl cannot pass pages and other search conditions and may get all `Collection Resources` at one time. It is not recommended to use kubectl to view `Collection Resource` if a large number of clusters are imported or a cluster has many `deployments`, `daemonsets` and `statefulsets` resources.

```bash
kubectl get collectionresources workloads
```
```
# Output
CLUSTER     GROUP   VERSION   KIND         NAMESPACE                     NAME                                          AGE
cluster-1   apps    v1        DaemonSet    kube-system                   vsphere-cloud-controller-manager              63d
cluster-2   apps    v1        Deployment   kube-system                   calico-kube-controllers                       109d
cluster-2   apps    v1        Deployment   kube-system                   coredns-coredns                               109d
...
```

**Search for `Collection Resource` by using URL**
{{< /tab >}}

{{< /tabs >}}
