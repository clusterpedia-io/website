--- 
title: PediaCluster
weight: 1
--- 

Clusterpedia 中使用 `PediaCluster` 资源来代表一个需要同步资源的集群。
> Clusterpedia 需要非常友好地对接到其他多云平台中，而多云平台可能会使用 `Cluster` 资源来代表纳管的集群。为了避免冲突，Clusterpedia 使用 PediaCluster。

```bash
$ kubectl get pediacluster
NAME        APISERVER                        VERSION            STATUS
demo1       https://10.6.101.100:6443        v1.22.3-aliyun.1   Healthy
demo2       https://10.6.101.100:6443        v1.21.0            Healthy
```

```yaml
apiVersion: cluster.clusterpedia.io/v1alpha2
kind: PediaCluster
metadata:
  name: demo1
spec:
  apiserver: https://10.6.101.100:6443
  kubeconfig:
  caData:
  tokenData:
  certData:
  keyData:
  syncResources: []
  syncAllCustomResources: false
  syncResourcesRefName: ""
```

`PediaCluster` 有两个作用：
* 配置集群的认证信息
* 配置同步的资源

配置集群认证信息可以参考[集群接入](../../usage/import-clusters)。

`PediaCluster` 有三个字段可以配置同步的资源：
* `spec.syncResources` 配置该集群需要同步的资源
* `spec.syncAllCustomResources` 同步所有的自定义资源
* `spec.syncResourcesRefName` 引用[公共的集群资源同步配置](../cluster-sync-resources)

具体如何配置同步资源，可以参考[同步集群资源](../../usage/sync-resources)。
