---
title: Multiple Clusters
weight: 10
---
Multi-cluster resource search allows us to **filter resources in multiple clusters at once based on query criteria, and provides the ability to paginate and sort these resources**.

When using `kubectl`, we can see what resources are currently available for search
```bash
kubectl --cluster clusterpedia api-resources
```
```
# Output：
NAME          SHORTNAMES   APIVERSION           NAMESPACED   KIND
configmaps    cm           v1                   true         ConfigMap
namespaces    ns           v1                   false        Namespace
nodes         no           v1                   false        Node
pods          po           v1                   true         Pod
secrets                    v1                   true         Secret
daemonsets    ds           apps/v1              true         DaemonSet
deployments   deploy       apps/v1              true         Deployment
replicasets   rs           apps/v1              true         ReplicaSet
issuers                    cert-manager.io/v1   true         Issuer
```
Clusterpedia provides multi-cluster resource search based on all cluster-synchronized resources,
and we can view [Sync Cluster Resources](../../sync-resources) to update the resources that need to be synchronized.

## Basic Features
### Specify Clusters
When searching multiple clusters, all clusters will be retrieved by default, we can also specify a single cluster or a group of clusters

{{< tabs >}}

{{% tab name="kubectl" %}}
Use [Search Label](../#search-by-metadata) `search.clusterpedia.io/clusters` to specify a group of clusters.
```bash
kubectl --cluster clusterpedia get deployments -l "search.clusterpedia.io/clusters in (cluster-1,cluster-2)"
```
```
# Output：
NAMESPACE     CLUSTER     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   cluster-1   coredns                   2/2     2            2           68d
kube-system   cluster-2   coredns                   2/2     2            2           64d
```

For specifying a single cluster search, we can also use Search Label to set it up, or see [Search in Specified Cluster](../specified-cluster) to specify a cluster using **URL Path**.
```bash
# specifying a single cluster
kubectl --cluster clusterpedia get deployments -l "search.clusterpedia.io/clusters=cluster-1"

# specifying a cluster can also be done with --cluster <cluster name>
kubectl --cluster cluster-1 get deployments"
```
{{% /tab %}}

{{% tab name="URL" %}}
When using URL, use `clusters` as URL Query to pass.
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?clusters=cluster-1"
```

If we specify a single cluster, we can also put the cluster name in the URL Path.
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/clusters/cluster-1/apis/apps/v1/deployments"
```
Lean More [Specify Cluster Search](../specified-cluster)
{{< /tab >}}

{{< /tabs >}}

### Specify Namespaces
We can **specify a single namespace or all namespaces** as if we were viewing a native Kubernetes resource.

{{< tabs >}}

{{% tab name="kubectl" %}}
Use `-n <namespace>` to specify the namespace, the default is in the *default* namespace
```bash
kubectl --cluster clusterpedia get deployments -n kube-system
```
```
# Output：
CLUSTER     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
cluster-1   coredns                   2/2     2            2           68d
cluster-2   calico-kube-controllers   1/1     1            1           64d
cluster-2   coredns                   2/2     2            2           64d
```

Use `-A` or `--all-namespaces` to see the resources under all namespaces for all clusters
```bash
kubectl --cluster clusterpedia get deployments -A
```
```
# Output：
NAMESPACE     CLUSTER     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   cluster-1   coredns                   2/2     2            2           68d
kube-system   cluster-2   calico-kube-controllers   1/1     1            1           64d
kube-system   cluster-2   coredns                   2/2     2            2           64d
default       cluster-2   dd-airflow-scheduler      0/1     1            0           54d
default       cluster-2   dd-airflow-web            0/1     1            0           54d
```
{{% /tab %}}

{{% tab name="URL" %}}
The **URL Path** to get the resources is the same as the native Kubernetes */apis/apps/v1/deployments*.

We just need to prefix the path to Clusterpedia Resources with **/apis/clusterpedia.io/v1beta1/resources** to indicate that it is currently a Clusterpedia request.
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"

# Specify namespace
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/namespaces/kube-system/deployments"
```
{{< /tab >}}

{{< /tabs >}}

---

In addition to specifying a single namespace, we can also **specify to search the resources under a group of namespaces**.
{{< tabs >}}

{{% tab name="kubectl" %}}
Use [Search Label](../#search-by-metadata) `search.clusterpedia.io/namespaces` to specify a group of namespaces.
> Be sure to specify the `-A` flag to avoid kubectl setting default namespace in the path.
```bash
kubectl --cluster clusterpedia get deployments -A -l "search.clusterpedia.io/namespaces in (kube-system, default)"
```
```
# Output：
NAMESPACE     CLUSTER     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   cluster-1   coredns                   2/2     2            2           68d
kube-system   cluster-2   calico-kube-controllers   1/1     1            1           64d
kube-system   cluster-2   coredns                   2/2     2            2           64d
default       cluster-2   dd-airflow-scheduler      0/1     1            0           54d
default       cluster-2   dd-airflow-web            0/1     1            0           54d
```
{{% /tab %}}

{{% tab name="URL" %}}
When using URL, we don't need to use Label Selector to pass parameters, just use URL Query - `namespaces`
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?namespaces=kube-system,default"
```
{{< /tab >}}

{{< /tabs >}}

### Specify Resource Names
Users can filter resources by a group of resource names

{{< tabs >}}

{{% tab name="kubectl" %}}
Use Search Label `search.clusterpedia.io/names` to specify a group of resource names.
> **Note: To search for resources under all namespaces, specify the `-A` flag, or use `-n` to specify the namespace.**
```bash
kubectl --cluster clusterpedia get deployments -A -l "search.clusterpedia.io/names=coredns"
```
```
# Output:
NAMESPACE     CLUSTER     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   cluster-1   coredns                   2/2     2            2           68d
kube-system   cluster-2   coredns                   2/2     2            2           64d
```
{{% /tab %}}

{{% tab name="URL" %}}
When using URL, use `names` to pass as **URL Query**, and if you need to specify namespaces, then add namespace to the path.
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?names=kube-coredns,dd-airflow-web"

# search resources with specified names under default namespace
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/namespaces/default/deployments?names=kube-coredns,dd-airflow-web"
```

When searching from multiple clusters, the data returned is actually encapsulated in a structure similar to `DeploymentList`.

If we want to get a single `Deployment` then we need to specify the cluster name in the URL path, refer to [Get Single Resource]()
{{< /tab >}}

{{< /tabs >}}

### Creation Time Interval
The creation time interval used for the search is left closed and right open, **since <= creation time < before**.

For more details on the time interval parameters, see [Search by Creation Time Interval](../#search-by-creation-time-interval)

{{< tabs >}}
{{% tab name="kubectl" %}}
Use Search Label - `search.clusterpedia.io/since` and `search.clusterpedia.io/before` to specify the time interval respectively.
```bash
kubectl --cluster clusterpedia get deployments -A -l "search.clusterpedia.io/since=2022-03-24, \
    search.clusterpedia.io/before=2022-04-10"
```
{{% /tab %}}
{{% tab name="URL" %}}
When using URLs, you can use Query - `since` and `before` to specify the time interval respectively.

```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?since=2022-03-24&before=2022-04-10"
```
{{< /tab >}}
{{< /tabs >}}

## Fuzzy Search
Currently supports fuzzy search based on resource names. 

Since fuzzy search needs to be discussed further, it is temporarily provided as an experimental feature.

Only the Search Label method is supported, URL Query isn't supported.
```bash
kubectl --cluster clusterpedia get deployments -A -l "internalstorage.clusterpedia.io/fuzzy-name=test"
```
Filters out deployments whose names contain the *test* string.

You can use the `in` operator to pass multiple fuzzy arguments, so that you can filter out resources that have all strings in their names.

## Field Selector
Native Kubernetes currently only supports field filtering on `metadata.name` and `metadata.namespace`, and the operators only support `=, `!=`, `==`, which is very limited.

**Clusterpedia provides more powerful features based on the compatibility with existing Field Selector features, and supports the same operators as `Label Selector`.**

Field Selector's key currently supports three formats.
* **Use `.` to sperate fields**
```bash
kubectl --cluster clusterpedia get pods --field-selector="status.phase=Running"

# we can also add the first character `.`
kubectl --cluster clusterpedia get pods --field-selector=".status.phase notin (Running,Succeeded)"
```

* **Field names wrapped in `''` or `""`** can be used for fields with illegal characters like `.`
```bash
kubectl --cluster clusterpedia get deploy \
    --field-selector="metadata.annotations['test.io'] in (value1,value2),spec.replica=3"
```

* **Use `[]` to separate fields**, the string inside `[]` must be wrapped with `''` or `""`
```bash
kubectl --cluster clusterpedia get pods --field-selector="status['phase']!=Running"
```

### Support List Fields
The actual design of field filtering takes into account the filtering of fields within `list elements`, but more discussion is needed as to whether the usage scenario actually makes sense:
[issue: support list field filtering](https://github.com/clusterpedia-io/clusterpedia/issues/48)

Examples：
```bash
kubectl get po --field-selector="spec.containers[].name!=container1"

kubectl get po --field-selector="spec.containers[].name == container1"

kubectl get po --field-selector="spec.containers[1].name in (container1,container2)"
```

## Search by Parent or Ancestor Owner
Searching by Owner is a very useful search function,
and Clusterpedia also supports **the seniority advancement of Owner to search for grandparents and even higher seniority**.

By searching by Owner, we can query all `Pods` under `Deployment` at once, without having to query `ReplicaSet` in between.

**When using the Owner query, we must specify a single cluster, either as a [Serach Label](../#search-by-owner) or [URL Query](../#search-by-owner), or you can specify the cluster name in the URL Path.**

For details on how to **search by Owner**, you can refer to [Search by Parent or Ancestor Owenr within a specified cluster](../specified-cluster#search-by-parent-or-ancestor-owner)

## Paging and Sorting
Paging and sorting are essential features for resource retrieval.

### Sorting by multiple fields
Multiple fields can be specified for sorting, and the support for sorting fields is determined by the Storage Layer.

The current `Default Storage Layer` supports sorting `cluster`，`namespace`，`name`，`created_at`，`resource_version` in both asc and desc,
and the fields are also supported in any combination
{{< tabs >}}

{{% tab name="kubectl" %}}
Sorting using  multiple fields
```bash
kubectl --cluster clusterpedia get pods -l \
    "search.clusterpedia.io/orderby in (cluster, name)"
```

Because of Label Selector's validation of value, order by desc requires **_desc** at the end of the field.
```bash
kubectl --cluster clusterpedia get pods -l \
    "search.clusterpedia.io/orderby in (namespace_desc, cluster, name)"
```
{{% /tab %}}

{{% tab name="URL" %}}
Use [URL Query](../#orderby) to specify sorting fields
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?orderby=namespace,cluster"
```

When specifying a field in order by desc, add **desc** to the end of the field, separated by spaces
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?orderby=namespace desc,cluster"
```
{{< /tab >}}

{{< /tabs >}}

### Paging
Native Kubernetes actually supports paging, and fields for paging queries already exist in [ListOptions](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#ListOptions).

Clusterpedia reuses the `ListOptions.Limit` and `ListOptions.Continue` fields as the `size` and `offset` for paging.

{{< tabs >}}

{{% tab name="kubectl" %}}
`kubectl --chunk-size` is actually used for paging pulls by setting `ListOptions.Limit`.

The native Kubernetes APIServer carries the `continue` for the next list in the returned response,
and performs the next list based on `--chunk-size` and `conintue` until the `conintue` is empty in the response data.

Clusterpedia does not return the `continue` field in the response by default in order to ensure paged search in `kubectl`,
which prevents `kubectl` from pulling all data using chunks.
```bash
kubectl --cluster cluster-1 get pods --chunk-size 10
```

Note that kubectl sets the `limit` to the default value of 500 without setting `--chunk-size`,
which means that `search.clusterpedia.io/size` does not actually take effect and is only used to correspond to `search.clusterpedia.io/offset`.
> `URL Query` has a higher priority than `Search Label`

There is no flag to set for `continue` in kubectl. So you have to use [Search Label](../#paging) to pass it.
```bash
kubectl --cluster clusterpedia get pods --chunk-size 10 -l \
    "search.clusterpedia.io/offset=10"
```
{{% /tab %}}

{{% tab name="URL" %}}
To paginate resources, just set the `limit` and `continue` in the URL.
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?limit=10&continue=5"
```
{{< /tab >}}

{{< /tabs >}}

### Response With Continue
[ListMeta.Continue](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#ListMeta) can be used in [ListOptions.Continue](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#ListOptions) as the offset for the next request.

As mentioned in the paging feature, Clusterepdia does not have `continue` in the response to prevent kubectl from pulling the full amount of data in pieces.

However, if the user requires it, he can request that the response include `continue`.
{{< tabs >}}

{{% tab name="URL" %}}
When accessing Clusterepdia using a URL, the response' `continue` can be used as the offset for the next request.
> Use with paging
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?withContinue=true&limit=1" | jq
```
```json
{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "continue": "1"
  },
  "items": [
    ...
  ]
}
```
{{% /tab %}}

{{% tab name="kubectl" %}}
**Setting `search.clusterpedia.io/with-continue` in kubectl will result in pulling the full amount of resources as a paged pull.**
```bash
kubectl --cluster clusterpedia get deploy -l \
    "search.clusterpedia.io/with-continue=true"
```
{{< /tab >}}

{{< /tabs >}}

### Response With Remaining Count
**In some UI cases, it is often necessary to get the total number of resources in the current search condition.**

The `RemainingItemCount` field exists in the [ListMeta](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#ListMeta) of the Kubernetes List response.

By reusing this field, the total number of resources can be returned in a Kubernetes OpenAPI-compatible manner:

**`offset + len(list.items) + list.metadata.remainingItemCount`**
> **When offset is too large, `remainingItemCount` may be negative, ensuring that the total number of resources can always be calculated.**

{{< tabs >}}

{{% tab name="URL" %}}
Set `withRemainingCount` in the [URL Query](../#paging) to request that the response include the number of remaining resources.
> Use with paging
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?withRemainingCount&limit=1" | jq
```
```json
{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "remainingItemCount": 23
  },
  "items": [
    ...
  ]
}
```
{{% /tab %}}

{{% tab name="kubectl" %}}
Need to use this feature as a URL
{{< /tab >}}

{{< /tabs >}}
