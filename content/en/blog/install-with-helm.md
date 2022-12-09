---
title: Quickly Deploy Clusterpedia with Helm
date: 2022-04-11
---
<!--当前 Clusterpedia 已经支持通过 Helm 来快速部署。-->
Currently Clusterpedia has supported a rapid deployment with Helm.

<!--首先需要保证当前环境已经安装 helm v3。-->
All of first, you need to check if helm v3 is installed in your current environment.

<!--
## 准备阶段
拉取 Clusterpedia 仓库代码。
> 当前暂时还未将 chart 上传至 charts 公共仓库。
-->
## Preparation

Pull the Clusterpedia repository.

> Currently, the chart has not been uploaded to the public charts repository.

```bash
git clone https://github.com/clusterpedia-io/clusterpedia.git
cd clusterpedia/charts
```

<!--
由于 clusterpedia 使用 `bitnami/postgresql` 和 `bitnami/mysql` 作为存储组件子 chart，
所以需要添加 bitnami 仓库，并更新 clusterpedia chart 的依赖。
-->
Since Clusterpedia uses `bitnami/postgresql` and `bitnami/mysql` as subcharts of storage components, it is necessary to add the bitnami repository and update the dependencies of the clusterpedia chart.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm dependency build
```

<!--
### 选择存储组件
Clusterpedia Chart 通过子 chart 的方式，提供了 `bitnami/postgresql` 和 `bitnami/mysql` 两款存储组件可供选择。
`postgresql` 为默认的存储组件，如果想要使用 MySQL，那么在后续安装命令中添加 `--set postgresql.enabled=false --set mysql.enabled=true`
更多关于存储组件的配置，可以参考 [bitnami/postgresql](https://github.com/bitnami/charts/tree/master/bitnami/postgresql) 和 [bitnami/mysql](https://github.com/bitnami/charts/tree/master/bitnami/mysql)。
**用户也可以选择不安装存储组件，而是使用外部组件，相关设置可以参考 charts/values.yaml**
-->
### Choose storage components

The Clusterpedia chart provides two storage components such as `bitnami/postgresql` and `bitnami/mysql` to choose from as sub-charts.

`postgresql` is the default storage component. IF you want to use MySQL, you can add `--set postgresql.enabled=false --set mysql.enabled=true` in the subsequent installation command.

For specific configuration about storage components, see [bitnami/postgresql](https://github.com/bitnami/charts/tree/master/bitnami/postgresql) and [bitnami/mysql](https://github.com/bitnami/charts/tree/master/bitnami/mysql).

**You can also choose not to install any storage component, but use external components. For related settings, see charts/values.yaml**

<!--
### 选择 CRD 的安装/管理方式
clusterpedia 要求环境中创建相应的 CRD 资源，可以选择手动部署 CRD YAML，也可以在 Helm 中管理。
#### 手动管理
-->
### Choose a installation or management mode for CRDs

Clusterpedia requires proper CRD resources to be created in the retrieval environment. You can choose to manually deploy CRDs by using YAML, or you can manage it with Helm.

#### Manage manually

```bash
kubectl apply -f ./_crds
```

<!--
#### 使用 Helm 管理
在后续安装命令中需要手动添加 `--set installCRDs=true` 即可。

### 决定是否需要创建 Local PV
Clusterpedia Chart 可以为用户创建存储组件使用 Local PV。
**用户在安装时需要通过 `--set persistenceMatchNode=<selected node name>` 来指定 Local PV 所在节点。**
如果用户不需要创建 Local PV，那么需要使用 `--set persistenceMatchNode=None` 显式声明。

## 安装 Clusterpedia
经过上述决策后，用户可以进行安装：
-->
#### Manage with Helm

Manually add `--set installCRDs=true` in the subsequent installation command.

### Check if you need to create a local PV

Through the Clusterpedia chart, you can create storage components to use a local PV.

**You need to specify the node where the local PV is located through `--set persistenceMatchNode=<selected node name>` during installation.**

If you need not to create the local PV, you can use `--set persistenceMatchNode=None` to declare it explicitly.

## Install Clusterpedia

After the above procedure is completed, you can run the following command to install Clusterpedia:

```bash
helm install clusterpedia . \
--namespace clusterpedia-system \
--create-namespace \
--set persistenceMatchNode={{ LOCAL_PV_NODE }} \
--set installCRDs=true
```

<!--
## 卸载 Clusterpedia
在卸载 Clusterpedia 前需要手动清理所有 `PediaCluster` 资源。
-->
## Uninstall Clusterpedia

Before uninstallation, you shall manually clear all `PediaCluster` resources.

```bash
kubectl get pediacluster
```

<!--`PediaCluster` 清理完成后就可以执行卸载命令。-->
You can run the command to uninstall it after the `PediaCluster` resources are cleared.

```bash
helm -n clusterpedia-system uninstall clusterpedia
```
<!--如果用户使用手动创建的 CRD 资源，那么同样也需要手动清理 CRD。-->
If you use any CRD resource that is manually created, you also need to manually clear the CRDs.

```bash
kubectl delete -f ./_crds
```

<!--
**注意 PVC 和 PV 并不会删除，用户需要手动删除。**
如果创建了 Local PV，那么还需要进入相应节点，清理 Local PV 的遗留数据。
-->
**Note that PVC and PV will not be deleted. You need to manually delete them.**

If you created a local PV, you need log in to the node and remove all remained data about the local PV.

```bash
# Log in to the node with Local PV
rm -rf /var/local/clusterpedia/internalstorage/<storage type>
```
