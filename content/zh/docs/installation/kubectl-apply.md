---
title: "使用 kubectl apply"
weight: 10
---

## 安装
Clusterpedia 的安装分为两个部分：

* [安装存储组件](#安装存储组件)
* [安装 Clusterpedia](#安装-clusterpedia)

> 用户如果使用已有的存储组件（MySQL 或者 PostgreSQL），则直接跳过安装存储组件。

拉取项目：

```bash
git clone https://github.com/clusterpedia-io/clusterpedia.git
cd clusterpedia
git checkout v0.5.0
```

### 安装存储组件
Clusterpedia 安装时提供了 **MySQL 8.0** 和 **PostgreSQL 12** 两种存储组件以供选择。
> 用户如果使用已有的存储组件（MySQL 或者 PostgreSQL），则直接[跳过](#安装-clusterpedia)存储组件安装。

{{< tabs >}}

{{% tab name="PostgreSQL" %}}
**进入所选存储组件的安装目录**

```bash
cd ./deploy/internalstorage/postgres
```

{{% /tab %}}

{{% tab name="MySQL" %}}
**进入所选存储组件的安装目录**

```bash
cd ./deploy/internalstorage/mysql
```

{{< /tab >}}

{{< /tabs >}}

**存储组件使用 Local PV 的方式存储数据，部署时需要指定 Local PV 所在节点**

> 用户可以选择自己提供 PV

```bash
export STORAGE_NODE_NAME=<节点名称>
sed "s|__NODE_NAME__|$STORAGE_NODE_NAME|g" `grep __NODE_NAME__ -rl ./templates` > clusterpedia_internalstorage_pv.yaml
```

**部署存储组件**

```bash
kubectl apply -f .

# 跳回 Clusterpedia 项目根目录
cd ../../../
```

### 安装 Clusterpedia

存储组件部署完成后，便可安装 Clusterpedia。

**如果选择使用已存在的存储组件，则需要参考 [配置存储层](../configurate/configurate-internalstorage) 来将存储组件对接到`默认存储层`中。**

> 在 clusterpedia 项目根目录下进行操作。

```bash
# 部署 Clusterpedia CRD 与组件
kubectl apply -f ./deploy
```

### 安装完成

检查组件 Pods 运行是否正常

```bash
kubectl -n clusterpedia-system get pods
```

### 部署集群自动接入策略 —— ClusterImportPolicy
0.4.0 后，Clusterpedia 提供了更加友好的接入多云平台的方式。

用户通过创建 `ClusterImportPolicy` 来自动发现多云平台中纳管的集群，并将这些集群自动同步为 `PediaCluster`，用户不需要根据纳管的集群来手动去维护 `PediaCluster` 了。

我们在 [Clusterpedia 仓库](https://github.com/clusterpedia-io/clusterpedia/tree/main/deploy/clusterimportpolicy) 中维护了各个多云平台的 `ClusterImportPolicy`。
**大家也提交用于对接其他多云平台的 `ClusterImportPolicy`。**

用户在安装 Clusterpedia 后，创建合适的 `ClusterImportPolicy` 即可，用户也可以根据自己的需求来[创建新的 `ClusterImportPolicy`](../../usage/interfacing-to-multi-cloud-platforms#新建-clusterimportpolicy)

具体可以参考 [接入多云平台](../../usage/interfacing-to-multi-cloud-platforms)
```bash
kubectl get clusterimportpolicy
```

## 卸载

### 删除 ClusterImportPolicy
如果用户部署了 ClusterImportPolicy 那么需要先清理 ClusterImportPolicy 资源

```bash
kubectl get clusterimportpolicy
```

### 清理 PediaCluster

在卸载 Clusterpedia 前，需要查看环境中是否还存在 PediaCluster 资源，如果存在那么需要删除这些资源。

```bash
kubectl get pediacluster
```

### 卸载 Clusterpedia

PediaCluster 资源清理完成后，卸载 Clusterpedia 相关组件。

```bash
# delete compontents
kubectl delete -f ./deploy/clusterpedia_apiserver_apiservice.yaml
kubectl delete -f ./deploy/clusterpedia_apiserver_deployment.yaml
kubectl delete -f ./deploy/clusterpedia_clustersynchro_manager_deployment.yaml
kubectl delete -f ./deploy/clusterpedia_controller_manager_deployment.yaml
kubectl delete -f ./deploy/clusterpedia_apiserver_rbac.yaml

# delete crds
kubectl delete -f ./deploy/cluster.clusterpedia.io_clustersyncresources.yaml
kubectl delete -f ./deploy/cluster.clusterpedia.io_pediaclusers.yaml
kubectl delete -f ./deploy/policy.clusterpedia.io_clusterimportpolicies.yaml
kubectl delete -f ./deploy/policy.clusterpedia.io_pediaclusterlifecycles.yaml
```

### 卸载存储组件

根据选择的存储组件类型，来移除相关的资源。

```bash
kubectl delete -f ./deploy/internalstorage/<storage type>
```

#### 清理 Local PV 以及数据

存储组件卸载后，PV 和相应的数据会依然遗留在节点中，我们需要手动清理。

通过 Local PV 资源详情，来查看挂载的节点。

```bash
kubectl get pv clusterpedia-internalstorage-<storage type>
```

得知数据保存的节点后，删除 Local PV。

```bash
kubectl delete pv clusterpedia-internalstorage-<storage type>
```

登录数据所在节点，清理数据。

```bash
# 遗留数据所在节点
rm /var/local/clusterpedia/internalstorage/<storage type>
```
