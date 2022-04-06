---
title: "指定集群检索"
weight: 11
---

Clusterpedia 除了支持多个集群的混合检索，还可以指定集群来检索资源。

> 在性能上使用 [Search Label](../#元信息检索) 或者 [URL Query](../#元信息检索) 来指定单个集群和在 URL Path 中指定集群并无区别
>
> 本文主要讲述在 URL Path 中指定集群

在以指定集群的方式使用 kubectl 前，需要先[配置 kubectl 的集群快捷方式](../../access-clusterpedia#为-kubectl-生成集群访问的快捷配置)

{{< tabs >}}

{{% tab name="kubectl" %}}
```bash
kubectl --cluster cluster-1 get deployments -n kube-system
```
```
# 输出：
NAMESPACE     CLUSTER     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   cluster-1   coredns                   2/2     2            2           68d
```
{{% /tab %}}

{{% tab name="URL" %}}
将 cluster name 放到 URL 路径中来指定集群
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/clusters/cluster-1/apis/apps/v1/deployments"
```

也可以通过 `URL Query` 来指定单个集群
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?clusters=cluster-1"
```
{{< /tab >}}

{{< /tabs >}}

指定集群支持的功能和[多集群检索](../multi-cluster)的作用基本相同，而且在 Owner 检索上更加方便，
另外在**获取单个资源时也只能使用在 URL Path 中指定集群的方式**。


## 根据父辈或者祖辈 Owner 进行检索
Owner 查询必须指定单个集群，可以使用 [Search Label](../#元信息检索) 或者 [URL Query](../#元信息检索) 来指定，也可以在 URL Path 中指定集群名称

**通过 `Owenr UID` 或者 `Owner Name`, 并且配合用于 Owner 辈分提升的 `Owenr Senirority` 可以完成基于祖辈 Owner 的资源检索。**
> 关于具体的查询参数，可以参考 [Owner 检索](../#owner-检索)

通过这种方式，可以直接查询到 Deployment 相应的 Pods，无需查询有哪些 `ReplicaSet` 属于该 `Deployment`。

### 指定 Owner  的 UID
**在指定 `Owenr UID` 后，`Owner Name` 和 `Owenr Group Resource` 会被忽略**

首先使用 kubectl 获取 `Deployment` 的 UID
```bash
kubectl --cluster cluster-1 get deploy fake-deploy -o jsonpath="{.metadata.uid}"
```
```
#输出：
151ae265-28fe-4734-850e-b641266cd5da
```
> 在 kubectl 下获取 uid 可能比较麻烦，但是在 UI 场景中通常已经更容易查看 `metadata.uid`

{{< tabs >}}

{{% tab name="kubectl" %}}
使用 `owner-uid` 来指定 Owner 的 UID, `owner-seniority` 对 Owner 进行辈分提升。

> *`owner-seniority` 默认为 0，表示 Owner 为父辈。设置为 1，提升 Owenr 辈分成为祖父 Owenr*

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

### 指定 Owner Name
如果事先并不知道 Owner 的 UID，那么指定 `Owner UID` 是一种比较麻烦的方式。

我们可以通过 Owner 的名字来指定 Owner，同时还可以指定 `Owner Group Resource` 来限制 Owner 的 Group Resource。

同样，我们以获取 Deployment 下相应的 Pods 来举例
{{< tabs >}}

{{% tab name="kubectl" %}}
```bash
kubectl --cluster cluster-1 get pods -l \
    "search.clusterpedia.io/owner-name=deploy-1,\
     search.clusterpedia.io/owner-seniority=1"
```

另外为了避免某些情况下，owner 资源存在多种类型，我们可以使用 Owner Group Resource 来限制 Owner 的类型
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

## 获取单个资源
当我们想要使用资源名称获取（Get）一个资源时，就必须要以 URL Path 的方式将集群名称传递进去，就像 namespace 一样。

{{< tabs >}}

{{% tab name="kubectl" %}}
如果使用多集群方式传递一个资源名称会报错
```bash
kubectl --cluster cluster-1 get deploy fake-deploy 
```
```
# 输出：
CLUSTER     NAME        READY   UP-TO-DATE   AVAILABLE   AGE
cluster-1   fake-deploy 1/1     1            1           35d
```
当然在 kubectl 中也可以通过 [Search Label](../#元信息检索) 来指定一个资源名称。

不过在 `-o yaml` 或者其他方式查看返回的源数据时，和使用 `kubectl --cluster <cluster name>` 是不一样的。
```bash
# 实际服务端返回 DeploymentList 的资源，由 kubectl 替换成 List 资源
kubectl --cluster clusterpedia get deploy -l 
    "search.clusterpedia.io/clusters=cluster-1,\
     search.clusterpedia.io/names=fake-deploy" -o yaml
```
```yaml
# 输出：
apiVersion: v1
items:
- ...
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```
实际返回的资源依然是一个 `KindList`，而 `kubectl --cluster <clsuter name>` 返回的便是具体的 `Kind`。
```bash
kubectl --cluster cluster-1 get deploy fake-deploy -o yaml
```
```yaml
# 输出：
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
获取指定资源的 URL 可以分为三部分
* 资源检索前缀： */apis/clusterpedia.io/v1beta1/resources*
* 指定集群名称 */clusters/< cluster name >*
* 原生 Kubernetes API 的资源 Path */apis/apps/v1/namespaces/< namespace >/deployments/< resource name >*

```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/clusters/cluster-1/apis/apps/v1/namespaces/default/deployments/fake-deploy"
```
{{< /tab >}}

{{< /tabs >}}
