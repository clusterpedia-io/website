---
title: 多集群 kube-state-metrics
weight: 10
---

Clusterpedia 以极小的代价提供了多集群资源的 kube-state-metrics 功能，它提供的指标信息与 [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics) 一致，当然会额外增加了一个集群名称的 label.
```text
kube_deployment_created{cluster="test-14",namespace="clusterpedia-system",deployment="clusterpedia-apiserver"} 1.676557618e+09
```

由于该功能处于试验阶段，所以使用该功能需要先额外通过[标准方式安装 Clusterpedia](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia#clusterpedia).

Clusterpedia 安装完成后，我们需要更新 helm 来开启 `多集群 kube-state-metrics` 功能。
> 当前 main 分支中已经合并 kube-state-metrics 功能，未来会包含在 `v0.8.0` 中。
> ghcr.io/iceber/clusterpedia/clustersynchro-manager:v0.8.0-ksm.1 镜像中包含该功能呢

## 开启多集群 kube-state-metrics
### 确保 Clusterpedia Chart 版本 >= v1.8.0
```bash
$ helm repo update clusterpedia
$ helm search clusterpedia
 NAME                            CHART VERSION   APP VERSION     DESCRIPTION
 clusterpedia/clusterpedia       1.8.0           v0.7.0          A Helm chart for Kubernetes
```

### 获取当前 chart values
```bash
$ helm -n clusterpedia-system get values clusterpedia > values.yaml
```

### 创建 patch values
```yaml
echo "clustersynchroManager:
  image:
    repository: iceber/clusterpedia/clustersynchro-manager
    tag: v0.8.0-ksm.1
  kubeStateMetrics:
    enabled: true
" > patch.yaml
```

### 更新 Clusterpedia 开启多集群 kube-state-metrics
```bash
$ helm -n clusterpedia-system upgrade -f values.yaml -f patch.yaml clusterpedia clusterpedia/clusterpedia
```

查看 clusterpedia kube-state-metrics services
```bash
$ kubectl -n clusterpedia-system get svc
NAME                                          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
clusterpedia-apiserver                        ClusterIP   10.97.129.238   <none>        443/TCP    150d
clusterpedia-clustersynchro-manager-metrics   ClusterIP   10.108.129.32   <none>        8081/TCP   51m
clusterpedia-kube-state-metrics               ClusterIP   10.108.130.62   <none>        8080/TCP   43m
clusterpedia-mysql                            ClusterIP   10.102.38.225   <none>        3306/TCP   150d
clusterpedia-mysql-headless                   ClusterIP   None            <none>        3306/TCP   150d
```

For more information on importing clusters and using clusterpedia: [Import Clusters](../../usage/import-clusters)

## 未来
`多集群 kube-state-metrics` 是一个非常有趣的功能，它可以让你不再需要在每一个集群中安装单集群版本的 kube-state-metrics，并且它可以很好的处理资源版本不同的问题。

这里有很多关于该功能的讨论，欢迎评论

* [The resource state metrics provide different metrics paths depending on the cluster](https://github.com/clusterpedia-io/clusterpedia/issues/544)
* [Support remote write to send resource metrics data](https://github.com/clusterpedia-io/clusterpedia/issues/545)
* [Support for filtering exposed resource state metrics based on namespace](https://github.com/clusterpedia-io/clusterpedia/issues/546)
* [Support for filtering exposed resource state metrics by cluster labels/annotations](https://github.com/clusterpedia-io/clusterpedia/issues/547)

也欢迎创建[新的 issue](https://github.com/clusterpedia-io/clusterpedia/issues/new/choose)
