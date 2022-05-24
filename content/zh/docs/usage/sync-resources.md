---
title: "同步集群资源"
weight: 2
---

Clusterpedia 的主要功能，便是提供对多集群内的资源进行复杂检索。

通过 `PediaCluster` 资源来指定该集群中哪些资源需要支持复杂检索，Clusterpedia 会将这些资源实时的通过`存储层`同步到`存储组件`中
```yaml
# example
apiVersion: cluster.clusterpedia.io/v1alpha2
kind: PediaCluster
metadata:
  name: cluster-example
spec:
  apiserver: "https://10.30.43.43:6443"
  syncResources:
  - group: apps
    resources:
     - deployments
  - group: ""
    resources:
     - pods
     - configmaps
  - group: cert-manager.io
    versions:
      - v1
    resources:
      - certificates
```

## 内置资源同步
`PediaCluster` 为了方便管理和查看这些同步的资源，用户需要以 Group 为单位来配置资源
```yaml
syncResources:
 - group: apps
   versions: []
   resources:
     - deployments
     - daemonsets
```
**对于内置资源，不需要填写 `versions` 字段。**

Clusterpedia 会根据该集群内所支持的资源版本自动选择合适的版本来收集，
并且用户无需担心版本转换的问题， Clusterpedia 会开放出该内置资源的所有版本接口。
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps" | jq
```
```json
{
  "kind": "APIGroup",
  "apiVersion": "v1",
  "name": "apps",
  "versions": [
    {
      "groupVersion": "apps/v1",
      "version": "v1"
    },
    {
      "groupVersion": "apps/v1beta2",
      "version": "v1beta2"
    },
    {
      "groupVersion": "apps/v1beta1",
      "version": "v1beta1"
    }
  ],
  "preferredVersion": {
    "groupVersion": "apps/v1",
    "version": "v1"
  }
}
```
可以看到 Clusterpedia 支持 `v1`，`v1beta2`，`v1beta1` 三个版本的 `Deployment`

## 自定义资源同步
相比内置资源，自定义资源在资源版本的配置上会稍有不同。
```yaml:
syncResources:
 - group: cert-manager.io
   versions: []
   resources:
    - certificates
```
**用户同样可以忽略 versions 字段，这时 Clusterpedia 就会同步该 Group 在该集群的前三个版本。**

以 cert-manager.io 为例，获取被接入集群中 cert-manager.io 支持的 Group
```bash
# 在被接入集群内执行
kubectl get --raw="/apis/cert-manager.io" | jq
```
```json
{
  "kind": "APIGroup",
  "apiVersion": "v1",
  "name": "cert-manager.io",
  "versions": [
    {
      "groupVersion": "cert-manager.io/v1",
      "version": "v1"
    },
    {
      "groupVersion": "cert-manager.io/v1beta1",
      "version": "v1beta1"
    },
    {
      "groupVersion": "cert-manager.io/v1alpha3",
      "version": "v1alpha3"
    },
    {
      "groupVersion": "cert-manager.io/v1alpha2",
      "version": "v1alpha2"
    }
  ],
  "preferredVersion": {
    "groupVersion": "cert-manager.io/v1",
    "version": "v1"
  }
}
```
可以看到，被接入集群支持 *cert-manager.io*  的 `v1`，`v1beta1`，`v1alpha3`，`v1alpha2` 四个版本。

当 `syncResources.[group].versions` 为空时，Clusterpedia 就会以 `APIGroup.versions` 列表的顺序，收集 `v1`， `v1beta1`，`v1alpah3` 三个版本，而 `v1alpha2` 不会被收集

### 指定自定义资源的同步版本
如果用户指定了 `versions`，那么就会按照 `versions` 的配置来收集指定的版本资源。
```yaml
syncResources:
 - group: cert-manager.io
   versions:
    - v1beta1
   resources:
    - certificates
```
这时，只会收集 `v1beta1` 版本。

### 使用注意
自定义资源收集暂时不支持版本转换，收集了哪些版本，那么就只支持哪些资源版本的收集。

这时在检索多集群资源时，如果 cluster-1 只收集了 `v1beta1` 版本的资源，而检索请求 v1 版本的资源便会忽略 cluster-1 所收集的 `v1beta1` 版本

需要用户去协调处理自定义资源在多个集群内的版本情况

## 查看资源同步状态
我们可以通过 `PediaCluster` 资源的 `Status` 来查看资源的信息，同步的资源版本和状态以及存储版本

在 `Status` 中，资源会有**同步版本**和**存储版本**：
* **同步版本**是 Clusterpedia 从被同步集群中获取到的资源的版本
* **存储版本**是 Clusterpedia 存储到存储层中的版本

```yaml
status:
  syncResources:
  - group: apps
    resources:
    - name: deployments
      kind: Deployment
      namespaced: true
      syncConditions:
      - lastTransitionTime: "2022-01-13T04:34:08Z"
        status: Syncing
        storageVersion: v1
        version: v1
```
通常集群资源的**同步版本**和**存储版本**是相同的。

但是当接入一个比较老的集群时，集群只提供了 `v1beta1` 版本的 `Deployment` 资源，而这时资源的**同步版本**为 `v1beta1`，**存储版本**为 `v1`

例如，同步 1.10 版本 Kubernetes 的 Deployment 时，同步状态为：
```yaml
status:
  syncResources:
  - group: apps
    resources:
    - name: deployments
      kind: Deployment
      namespaced: true
      syncConditions:
      - lastTransitionTime: "2022-01-13T04:34:04Z"
        status: Syncing
        storageVersion: v1
        version: v1beta1
```

对于自定义资源来说，**同步版本**和**存储版本**是一致的

## 接下来
资源同步完成后，便可以[访问 Clusterpedia](../access-clusterpedia)来[检索资源](../search)
