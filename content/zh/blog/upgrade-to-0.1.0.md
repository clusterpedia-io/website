---
title: 升级到 Clusterpedia 0.1.0
date: 2022-02-15
---
随着 Clusterpedia 0.1.0 版本的发布，我们可以将早期的 0.0.9-alpha 或者 0.0.8 更新到 0.1.0 了。

## 旧版本资源清理
由于资源检索的路径发生了修改([#73](https://github.com/clusterpedia-io/clusterpedia/pull/73))，所以需要使用 0.0.9-alpha 的 [clean-clusterconfigs.sh](https://github.com/clusterpedia-io/clusterpedia/blob/v0.0.9-alpha/hack/clean-clusterconfigs.sh) 来清理 *.kube/config* 中的 Clusterpedia 集群访问配置
```bash
curl -sfL https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/v0.0.9-alpha/hack/clean-clusterconfigs.sh | sh -
```

备份并删除 `PediaCluster` 资源
```bash
kubectl get pediacluster -o clusters.yaml.bak

kubectl delete pediacluster --all
```

所有的 `PediaCluster` 资源都删除后，删除 `PediaCluster CRD`
```bash
kubectl delete crd pediaclusters.clusters.clusterpedia.io
```

删除用于注册聚合式 API 的 `APIServices`
```bash
kubectl delete apiservices v1alpha1.pedia.clusterpedia.io
```

## 更新 Clusterpedia
新建 `PediaCluster CRD`, 并且更新 `Clusterpedia APIServer` 和 `Clustersynchro Manager`
```bash
DEPLOY_YAML_PATH=https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/v0.1.0/deploy
CRD_YAML_PATH=$DEPLOY_YAML_PATH/crds

kubectl apply -f \
    $CRD_YAML_PATH/cluster.clusterpedia.io_pediaclusters.yaml,\
    $DEPLOY_YAML_PATH/clusterpedia_clustersynchro_manager_deployment.yaml,\
    $DEPLOY_YAML_PATH/clusterpedia_apiserver_deployment.yaml,\
    $DEPLOY_YAML_PATH/clusterpedia_apiserver_apiservice.yaml
```
可以将 YAML 下载到本地，或者拉取项目到本地，进入 ./deploy 目录下执行 kubectl apply。

## 重新接入集群
由于 `PediaCluster` 的 APIVersion 和结构都进行了一些不兼容的优化，所以需要重新根据备份的 *clusters.yaml.bak* 来重新创建 `PediaCluster`。

当前 `PediaCluster` 的示例为：
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
  syncResources:
  - group: apps
    resources:
     - deployments
  - group: ""
    resources:
     - pods
```
相比 0.0.9-alpha 主要有三个修改：
* `apiVersion` 由 *clusters.clusterpedia.io/v1alpha1* -> *cluster.clusterpedia.io/v1alpha2*
* `spec.apiserverURL` -> `spec.apiserver`
* `spec.resources` -> `spec.syncResources`

> 具体的修改可以查看: [#70](https://github.com/clusterpedia-io/clusterpedia/pull/70)
> [#67](https://github.com/clusterpedia-io/clusterpedia/pull/67)
> [#76](https://github.com/clusterpedia-io/clusterpedia/pull/76)
> [#77](https://github.com/clusterpedia-io/clusterpedia/pull/77)

根据 *clusters.yaml.bak* 内旧的 PediaCluster 来创建新的 `PediaCluster`。
```yaml
apiVersion: cluster.clusterpedia.io/v1alpha2
kind: PediaCluster
metadata:
  name: cluster-1
spec: {}
---
apiVersion: cluster.clusterpedia.io/v1alpha2
kind: PediaCluster
metadata:
  name: cluster-2
spec: {}
```

查看集群接入状态
```bash
kubectl get pediacluster
```

为多集群检索生成快捷访问配置
```bash
curl -sfL https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/v0.1.0/hack/gen-clusterconfigs.sh | sh -
```
