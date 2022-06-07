---
title: Clusterpedia v0.2.0 发布
date: 2022-04-12
---
## 使用 kube config 来接入集群
v0.1.0 时，用户需要分别填写被接入集群的 apiserver 地址，以及访问集群时的认证信息
```yaml
apiVersion: cluster.clusterpedia.io/v1alpha2
kind: PediaCluster
metadata:
  name: cluster-example
spec:
  apiserver: "https://10.30.43.43:6443"
  caData:
  tokenData:
  certData:
  keyData:
  syncResources: []
```

在 v0.2.0 中 `PediaCluster` 增加了 `spec.kubeconfig` 字段，用户可以直接使用 kube config 来接入集群。

首先 base64 集群的 *kube config*:
```bash
$ base64 ./kubeconfig.yaml
```
然后填充到 PediaCluster 的 `spec.kubeconfig` 字段中
```yaml
apiVersion: cluster.clusterpedia.io/v1alpha2
kind: PediaCluster
metadata:
  name: cluster-example
spec:
  kubeconfig: **base64 kubeconfig**
  syncResources: []
```
在使用 *kube config* 时，不需要填写 `spec.apiserver` 以及其他认证字段。

需要注意，使用 `kubectl get pediacluster` 查看接入的集群列表时，**APISERVER** 不会显示集群地址
```bash
$ kubectl get pediacluster
NAME              APISERVER   VERSION   STATUS
cluster-example               v1.22.2   Healthy
```
如果需要显示，那么需要额外手动设置 `spec.kubeconfig`，未来会添加 Mutating Admission Webhook 来解析 kubeconfig 并自动填充 `spec.apiserver` 字段。

## 新增的检索功能
### 通过资源的创建时间来过滤资源
| 作用 | Search Label Key | URL Query |
| ---- | ---------------- | --------- |
| Since | search.clusterpedia.io/since | since |
| Before | search.clusterpedia.io/before | before |

创建时间的区间采用左闭右开的规则，**since <= creation time < before**。

时间格式支持 4 种：
1. `Unix 时间戳格式` 为了方便使用会根据时间戳的长度来区分单位为 s 还是 ms。 10 位时间戳单位为秒，13 位时间戳单位为毫秒。
2. `RFC3339` 2006-01-02T15:04:05Z or 2006-01-02T15:04:05+08:00
3. `UTC Date` 2006-01-02
4. `UTC Datetime` 2006-01-02 15:04:05

> 由于 Kube Label Selector 的限制，Search Label 只支持使用 Unix 时间戳和 UTC Data 的格式
> URL Query 可以使用四种格式

首先查看一下当前都有哪些资源
```bash
$ kubectl --cluster clusterpedia get pods
CLUSTER           NAME                                                 READY   STATUS      RESTARTS       AGE
cluster-example   quickstart-ingress-nginx-admission-create--1-kxlnn   0/1     Completed   0              171d
cluster-example   fake-pod-698dfbbd5b-wvtvw                            1/1     Running     0              8d
cluster-example   fake-pod-698dfbbd5b-74cjx                            1/1     Running     0              21d
cluster-example   fake-pod-698dfbbd5b-tmcw7                            1/1     Running     0              8d
```

我们使用创建时间来过滤资源
```bash
$ kubectl --cluster clusterpedia get pods -l "search.clusterpedia.io/since=2022-03-20"
CLUSTER           NAME                        READY   STATUS    RESTARTS   AGE
cluster-example   fake-pod-698dfbbd5b-wvtvw   1/1     Running   0          8d
cluster-example   fake-pod-698dfbbd5b-tmcw7   1/1     Running   0          8d


$ kubectl --cluster clusterpedia get pods -l "search.clusterpedia.io/before=2022-03-20"
CLUSTER           NAME                                                 READY   STATUS      RESTARTS       AGE
cluster-example   quickstart-ingress-nginx-admission-create--1-kxlnn   0/1     Completed   0              171d
cluster-example   fake-pod-698dfbbd5b-74cjx                            1/1     Running     0              21d
```

### 使用 Owner Name 检索
在 v0.1.0 时，我们可以指定祖辈或者父辈 `Owner UID` 来查询资源，不过 `Owner UID` 使用起来并不方便，毕竟还需要提前得知 Owner 资源的 UID。

在 v0.2.0 版本中，支持直接使用 `Owner Name` 来查询，并且 Owner 查询由实验性功能进入到正式功能，Search Label 的前缀也由 *internalstorage.clusterpedia.io* 升级为 *search.clusterpedia.io*，并且提供了 URL Query。

| 作用 | Search Label Key | URL Query |
| ---- | ---------------- | --------- |
|指定 Owner UID|search.clusterpedia.io/owner-uid|ownerUID|
|指定 Owner Name|search.clusterpedia.io/owner-name|ownerName|
|指定 Owner Group Resource|search.clusterpedia.io/owner-gr|ownerGR|
|指定 Owner 辈分|internalstorage.clusterpedia.io/owner-seniority|ownerSeniority|

> 如果用户同时指定了 `Owner UID` 和 `Owner Name`，那么 `Owner Name` 会被忽略。

```bash
$ kubectl --cluster cluster-example get pods -l \
    "search.clusterpedia.io/owner-name=fake-pod, \
     search.clusterpedia.io/owner-seniority=1"
CLUSTER           NAME                        READY   STATUS    RESTARTS   AGE
cluster-example   fake-pod-698dfbbd5b-wvtvw   1/1     Running   0          8d
cluster-example   fake-pod-698dfbbd5b-74cjx   1/1     Running   0          21d
cluster-example   fake-pod-698dfbbd5b-tmcw7   1/1     Running   0          8d
```

**另外为了避免某些情况下，owner 资源存在多种类型，我们可以使用 `Owner Group Resource` 来限制 Owner 的类型。**
```bash
$ kubectl --cluster cluster-example get pods -l \
    "search.clusterpedia.io/owner-name=fake-pod,\
     search.clusterpedia.io/owner-gr=deployments.apps,\
     search.clusterpedia.io/owner-seniority=1"
... some output
```

### 根据资源名称的模糊搜索
模糊搜索是一个非常常用的功能，当前暂时只提供了资源名称上的模糊搜索，由于还需要更多功能上的讨论，暂时作为试验性功能
| 作用 | Search Label Key | URL Query |
| ---- | ---------------- | --------- |
| 根据资源名称进行模糊搜索| internalstorage.clusterpedia.io/fuzzy-name | - |

```bash
$ kubectl --cluster clusterpedia get deployments -l "internalstorage.clusterpedia.io/fuzzy-name=fake"
CLUSTER           NAME       READY   UP-TO-DATE   AVAILABLE   AGE
cluster-example   fake-pod   3/3     3            3           113d
```
可以使用 `in` 操作符来指定多个参数，这样可以过滤出名字包含所有模糊字符串的资源。

## 其他功能
在 v0.1.0 中，查询资源列表时，允许返回的剩余的资源数量，这样用户可以通过计算就能得知当前检索添加下的资源总量。

在 v0.2.0 中对该功能进行了强化， 当分页查询的 `Offset` 参数过大时，`ReaminingItemCount` 可以为负数，
这样可以保证通过 **`offset + len(list.items) + list.metadata.remainingItemCount`** 总是可以计算出正确的资源总量。


# [发布日志](https://github.com/clusterpedia-io/clusterpedia/releases/tag/v0.2.0)
## What's New
* 支持使用 Helm 部署 ([#53](https://github.com/clusterpedia-io/clusterpedia/pull/53), [#125](https://github.com/clusterpedia-io/clusterpedia/pull/125), @calvin0327, @wzshiming)
* PediaCluster 支持使用 kube config 来接入集群 ([#115](https://github.com/clusterpedia-io/clusterpedia/pull/115), @wzshiming)

### APIServer
* 支持通过创建时间的区间来过滤资源 ([#113](https://github.com/clusterpedia-io/clusterpedia/pull/113), @cleverhu)
* 支持根据 Owner 的名字来检索资源，并且 Owner 查询成为 clusterpedia 的正式功能，同时支持 Search Label 和 URL Query ([#91](https://github.com/clusterpedia-io/clusterpedia/pull/91), @Iceber)

### Default Storage Layer
* 支持根据资源名称的模糊搜索 ([#117](https://github.com/clusterpedia-io/clusterpedia/pull/117), @cleverhu)
* RemainingItemCount 可以为负数，在 Offset 过大时依然可以使用 `offset + len(items) + remainingItemCount` 来计算资源总量。([#123](https://github.com/clusterpedia-io/clusterpedia/pull/123), @cleverhu)

#### Bug Fixes
* 修复由于不必要的反序列化导致的 cpu 损耗，提升了查询时的性能 ([#89](https://github.com/clusterpedia-io/clusterpedia/pull/89), [#92](https://github.com/clusterpedia-io/clusterpedia/pull/92), @Iceber)

#### Deprecation
* Owner 查询已移动到正式功能，用于 Owner 查询的试验性 Search Label —— `internalstorage.clusterpedia.io/owner-name` 和 `internalstorage.clusterpedia.io/owner-seniority` 会在下一个版本被移除 ([#91](https://github.com/clusterpedia-io/clusterpedia/pull/91), @Iceber)
