---
title: "Specified a Cluster"
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

**Searching for resources based on ancestor owners can be done with `Owner UID` or `Owner Name`, and with `Owner Seniority` for Owner seniority advancement.**
> For the specific query parameters, you can refer to [Search by Owner](../#search-by-owner)

In this way, you can directly search for the `Pods` corresponding to a Deployment without having to query which `ReplicaSet` belong to that `Deployment`.

### Use the Owner UID
**`Owner Name` and `Owner Group Resource` will be ignored after `Owner UID` is specified.**

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
    "search.clusterpedia.io/owner-uid=151ae265-28fe-4734-850e-b641266cd5da,\
     search.clusterpedia.io/owner-seniority=1"
```

{{% /tab %}}

{{% tab name="URL" %}}
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/clusters/cluster-1/api/v1/namespaces/default/pods?ownerUID=151ae265-28fe-4734-850e-b641266cd5da&ownerSeniority=1"
```
{{< /tab >}}

{{< /tabs >}}

## Use the Owner Name
If the Owner UID is not known in advance, then using `Owner UID` is a more troublesome way.

We can specify the Owner by it's name, and we can also specify `Owner Group Resource` to restrict the Owner's Group Resource.

Again, let's take the example of getting the corresponding Pods under Deployment.
{{< tabs >}}

{{% tab name="kubectl" %}}
```bash
kubectl --cluster cluster-1 get pods -l \
    "search.clusterpedia.io/owner-name=deploy-1,\
     search.clusterpedia.io/owner-seniority=1"
```

In addition, to avoid multiple types of owner resources in some cases, we can use the `Owner Group Resource` to restrict the type of owner.
```bash
kubectl --cluster cluster-1 get pods -l \
    "search.clusterpedia.io/owner-name=deploy-1,\
     search.clusterpedia.io/owner-gr=deployments.apps,\
     search.clusterpedia.io/owner-seniority=1"
```

{{% /tab %}}

{{% tab name="URL" %}}
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/clusters/cluster-1/api/v1/namespaces/default/pods?ownerName=deploy-1&ownerSeniority=1"
```
{{< /tab >}}

{{< /tabs >}}

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
* Resource name for Kubernetes API: Path */apis/apps/v1/namespaces/< namespace >/deployments/< resource name >*

```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/clusters/cluster-1/apis/apps/v1/namespaces/default/deployments/fake-deploy"
```
{{< /tab >}}

{{< /tabs >}}
