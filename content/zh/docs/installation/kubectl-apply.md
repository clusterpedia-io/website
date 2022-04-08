---
title: "使用 kubectl apply"
weight: 10
---

## 安装
Clusterpedia 的安装分为两个部分：
* [安装存储组件](#安装存储组件)
* [安装 Clusterpedia](#安装-clusterpedia)
> 用户如果使用已有的存储组件（MySQL 或者 PostgreSQL），则直接跳过安装存储组件

拉取项目：
```bash
git clone https://github.com/clusterpedia-io/clusterpedia.git
cd clusterpedia
git checkout v0.2.0
```

### 安装存储组件
Clusterpedia 安装时提供了 **MySQL 8.0** 和 **PostgreSQL 12** 两种存储组件以供选择
> 用户如果使用已有的存储组件（MySQL 或者 PostgreSQL），则直接[跳过](#安装-clusterpedia)存储组件安装

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
kubectl create -f .

# 跳回 Clusterpedia 项目根目录
cd ../../
```

### 安装 Clusterpedia
存储组件部署完成后，便可安装 Clusterpedia。

**如果选择使用已存在的存储组件，则需要参考 [配置存储层](../configurate/configurate-internalstorage) 来将存储组件对接到`默认存储层`中**

> 在 clusterpedia 项目根目录下进行操作
```bash
# 部署 crds
kubectl apply -f ./charts/_crds

# 部署 Clusterpedia 组件
kubectl apply -f ./deploy
```

### 安装完成
检查组件 Pods 运行是否正常
```bash
kubectl -n clusterpedia-system get pods
```

## 卸载
### 清理 PediaCluster
在卸载 Clusterpedia 前，需要查看环境中是否还存在 PediaCluster 资源，如果存在那么需要删除这些资源
```bash
kubectl get pediacluster
```

### 卸载 Clusterpedia
PediaCluster 资源清理完成后，卸载 Clusterpedia 相关组件。

```bash
kubectl delete -f ./deploy/clusterpedia_apiserver_apiservice.yaml
kubectl delete -f ./deploy/clusterpedia_apiserver_deployment.yaml
kubectl delete -f ./deploy/clusterpedia_clustersynchro_manager_deployment.yaml
kubectl delete -f ./deploy/clusterpedia_apiserver_rbac.yaml
kubectl delete -f ./charts/_crds
```

### 卸载存储组件
根据选择的存储组件类型，来移除相关的资源
```bash
kubectl delete -f ./deploy/internalstorage/<storage type>
```

#### 清理 Local PV 以及数据
存储组件卸载后，PV 和相应的数据会依然遗留在节点中，我们需要手动清理

通过 Local PV 资源详情，来查看挂载的节点
```bash
kubectl get pv clusterpedia-internalstorage-<storage type>
```

得知数据保存的节点后，删除 Local PV
```bash
kubectl delete pv clusterpedia-internalstorage-<storage type>
```

登录数据所在节点，清理数据
```bash
# 遗留数据所在节点
rm /var/local/clusterpedia/internalstorage/<storage type>
```
