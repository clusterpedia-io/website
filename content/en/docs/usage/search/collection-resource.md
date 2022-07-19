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

## Only Metadata
When we retrieve a CollectionResource, the default resource is the full resource content, but sometimes we just need to retrieve the metadata of the resource.

We can use url query -- `onlyMetadata` to retrieve only the resource metadata when retrieving.
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
> `any collectionresource` is one of the ways that users can freely combine resource types with [custom collection resources](../../../concepts/collection-resource#custom-collection-resource) to see more.

clusterpedia supports a special CollectionResource —— `any`.
```bash
$ kubectl get collectionresources
NAME            RESOURCES
any             *
```

When retrieving `any collectionresource`, we must specify a set of resource types by url query, so we can only retrieve `any collectionresource` via [clusterpedia-io/client-go](https://github.com/clusterpedia-io/client-go/pull/43) or URL.
```bash
$ kubectl get collectionresources any
Error from server (BadRequest): url query - `groups` or `resources` is required
```

`any collectionresource` supports two url queries —— `groups` and `resources`

**`groups` and `resources` can be specified together, currently they are taken together and are not de-duplicated, the caller is responsible for de-duplication**,
there are some future optimizations for this behavior.

```bash
$ kubectl get --raw "/apis/clusterpedia.io/v1beta1/collectionresources/any?onlyMetadata=true&groups=apps&resources=batch/jobs,batch/cronjobs" | jq
```

#### groups
`groups` can specify a group and version of a set of resources, with multiple group versions separated by `,`,

the group version format is **< group >/< version >**, or no version can be specified **< group >**, *for resources under /api, you can just use the empty string*.

**Example**: **groups=apps/v1,,batch** specifies three groups apps/v1, core, batch.

#### resources
`resources` can specify a specific resource type, multiple resource types are separated by `,`,

the resource type format is **< group >/< version >/< resource>**, by also not specifying the version **< group >/< resource >**.

**Example**: **resources=apps/v1/deployments,apps/daemonsets,/pods** specifies three resources deloyments, daemonsets and pods.
