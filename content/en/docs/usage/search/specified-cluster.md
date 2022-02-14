---
title: "Search in a Specified Cluster"
weight: 11
---

In addition to searching in multiple clusters, Clusterpedia can also search for resources in a specified cluster.  

> Using [`Search Label`](../#search-by-metadata) or [`URL Query`](../#search-by-metadata) to specify a single cluster is not different from specifying a cluster in URL Path in terms of performance
>
> This topic focuses on specifying clusters in URL Path

Before using kubectl in the way of specifying a cluster, you need to [configure the cluster shortcut for kubectl](../../access-clusterpedia#configure-the-cluster-shortcut-for-kubectl)

{{< tabs >}}

{{% tab name="kubectl" %}}
```bash
kubectl --cluster cluster-1 get deployments -n kube-system
```
```
# Output:
NAMESPACE     CLUSTER     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   cluster-1   coredns                   2/2     2            2           68d
```
{{% /tab %}}

{{% tab name="URL" %}}
Specify a cluster by using the cluster name in the URL path
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/clusters/cluster-1/apis/apps/v1/deployments"
```

You can also specify a single cluster by `URL Query`
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?clusters=cluster-1"
```
{{< /tab >}}

{{< /tabs >}}

The function supported by searching in a specified cluster is basically the same as that of [multi-cluster search](../multi-cluster). 

It is more convenient for searching by Owner in a specified cluster. In addition, when **getting a single resource, you can only use the specified cluster in the URL Path**.

## Search by Parent or Ancestor Owner
To query by Owner, you shall specify a single cluster. You can use [Search Label](../#search-by-metadata) or [URL Query](../#search-by-metadata) to specify, or specify the cluster name in the URL Path.

**Resource search based on ancestors Owner can be completed through `Owenr UID` and `Owenr Senirority`.**
> Currently only supports query by `Owner UID`

For example, directly query the corresponding `Pods` by using `Deployment UID`, without querying which `ReplicaSet` belongs to the `Deployment`.  

Firstly use kubectl to get `Deployment` UID
```bash
kubectl --cluster cluster-1 get deploy fake-deploy -o jsonpath="{.metadata.uid}"
```
```
#Output:
151ae265-28fe-4734-850e-b641266cd5da
```
> Getting the uid under kubectl may be tricky, but it's usually already easier to check `metadata.uid` in UI scenarios

{{< tabs >}}

{{% tab name="kubectl" %}}
Use `owner-uid` to specify Owner UID and use `owner-seniority` to promote the Owner's seniority.

> *`owner-seniority` is 0 by default, which represents Owner is parent. If you set it to 1, Owenr can be promoted to grandfather*

```
kubectl --cluster cluster-1 get pods -l \
    "internalstorage.clusterpedia.io/owner-uid=151ae265-28fe-4734-850e-b641266cd5da,\
     internalstorage.clusterpedia.io/owner-seniority=1"
```

{{% /tab %}}

{{% tab name="URL" %}}
Search by Owner is an experimental feature and URL Query has not been provided yet
{{< /tab >}}

{{< /tabs >}}

> Search by Owner is an experimental feature, temporarily prefixed with *internalstorage.clusterpedia.io* as [Search Label](../#search-by-owner)
>
> After you confirm the availability and usefulness of relevant features, move it under *search.clusterpedia.io*.

The search feature of combining `Owner Namespace` and `Owenr Name` into `Owner Key` is still in discussion, welcome to join [issue: Support for searching resources by owner](https://github.com/clusterpedia-io/clusterpedia/issues/49) and discuss together.

## Get a single resource
When we want to use the resource name to get (Get) a resource, we must pass the cluster name in the URL Path, just like namespace.

{{< tabs >}}

{{% tab name="kubectl" %}}
If a resource name is passed in a multi-cluster mode, an error will be reported
```bash
kubectl --cluster cluster-1 get deploy fake-deploy 
```
```
# Output:
CLUSTER     NAME        READY   UP-TO-DATE   AVAILABLE   AGE
cluster-1   fake-deploy 1/1     1            1           35d
```
Certainly, you can use [Search Label](../#search-by-metadata) to specify a resource name in the case of kubectl.

However, if you use `-o yaml` or other methods to check the returned source data, it is different from using `kubectl --cluster <cluster name>`.
```bash
# The actual server returns the DeploymentList resource, which is replaced with a list by kubectl
kubectl --cluster clusterpedia get deploy -l 
    "search.clusterpedia.io/clusters=cluster-1,\
     search.clusterpedia.io/names=fake-deploy" -o yaml
```
```yaml
# Output:
apiVersion: v1
items:
- ...
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```
The actual returned resource is still a `KindList`, while `kubectl --cluster <clsuter name>` returns a specific `Kind`.
```bash
kubectl --cluster cluster-1 get deploy fake-deploy -o yaml
```
```yaml
# Output:
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
    shadow.clusterpedia.io/cluster-name: cluster-1
  creationTimestamp: "2021-12-16T02:26:29Z"
  generation: 2
  name: fake-deploy
  namespace: default
  resourceVersion: "38085769"
  uid: 151ae265-28fe-4734-850e-b641266cd5da
spec:
  ...
status:
  ...
```

{{% /tab %}}

{{% tab name="URL" %}}
The URL to get a specified resource can be divided into three parts:
* Prefix to search for resource: */apis/clusterpedia.io/v1beta1/resources*
* Specified cluster name: */clusters/< cluster name >*
* Resource name for Kubernetes API: Path */apis/apps/v1/deployments/namespaces/<namespace>/<resource name>*

```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/clusters/cluster-1/apis/apps/v1alpha1/deployments/namespaces/default/fake-deploy"
```
{{< /tab >}}

{{< /tabs >}}
