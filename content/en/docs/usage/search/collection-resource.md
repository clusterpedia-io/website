---
title: "Search for collection resources"
weight: 12
---

For collection resource, refer to [what is Collection Resource](../../../concepts/collection-resource)  

Due to the limitation of kubectl, we cannot pass search conditions through `Label Selector` or other methods, so **it is recommended to search for `collection resource`** by using a URL.


{{< tabs >}}

{{% tab name="URL" %}}
> **When requesting `collection resources`, you shall use [paging function](../#paging) because the number of resources may be very large.**
```bash
kubectl get --raw="/apis/pedia.clusterpedia.io/v1alpha1/collectionresources/workloads?limit=1" | jq
```
```json
# Output
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

The complex search of `collection resources` is basically the same as the function of [multi-cluster resource search](../multi-cluster), only some operations are not supported:
* Search by Owner is not supported. If you need to specify a specific resource type to search by Owner, you can refer to [`Multi-cluster search`](../multi-cluster#query by parent/grandparent owner) and [`Specified cluster search`](. ./specified-cluster#Search by parent or ancestor owner)
* Getting a specific single resource in Collection Resource is not supported, because you shall specify cluster and type for a specific resource. In this case, you can use [Get a single resource]().
{{% /tab %}}

{{% tab name="kubectl" %}}
It is not easy to search for `collection resources` by using kubectl. However, you can have a try.
> kubectl cannot pass pages and other search conditions and may get all `collection resources` at one time. It is not recommended to use kubectl to view `collection resources` if a large number of clusters are imported or a cluster has many `deployments`, `daemonsets` and `statefulsets` resources.

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

**Search for `collection resources` by using URL**
{{< /tab >}}

{{< /tabs >}}
