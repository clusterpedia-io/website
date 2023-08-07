---
title: Multi-Cluster kube-state-metrics
weight: 10
---

Clusterpedia provides kube-state-metrics features for multi-cluster resources at a fraction of the cost, providing the same metrics information as [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics), but with the addition of a cluster name label.
```text
kube_deployment_created{cluster="test-14",namespace="clusterpedia-system",deployment="clusterpedia-apiserver"} 1.676557618e+09
```

Since this feature is experimental, you will install Clusterpedia [the standard way first](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia#clusterpedia).

Once Clusterpedia is installed, we need to update the helm to enable the `multi-cluster kube-state-metrics` feature.
> The kube-state-metrics feature has been merged into [the main branch](https://github.com/clusterpedia-io/clusterpedia) and will be included in `v0.8.0` in the future.
> The feature is included in the ghcr.io/iceber/clusterpedia/clustersynchro-manager:v0.8.0-ksm.1

## Enable Multi-Cluster kube-state-metrics
### Ensure Clusterpedia Chart Version >= v1.8.0
```bash
$ helm repo update clusterpedia
$ helm search clusterpedia
 NAME                            CHART VERSION   APP VERSION     DESCRIPTION
 clusterpedia/clusterpedia       1.8.0           v0.7.0          A Helm chart for Kubernetes
```

### Get the current chart values
```bash
$ helm -n clusterpedia-system get values clusterpedia > values.yaml
```

### Create patch values
```yaml
$ echo "clustersynchroManager:
  image:
    repository: iceber/clusterpedia/clustersynchro-manager
    tag: v0.8.0-ksm.1
  kubeStateMetrics:
    enabled: true
" > patch.yaml
```

### Update Clusterpedia to enable multi-cluster kube-state-metrics.
```bash
$ helm -n clusterpedia-system upgrade -f values.yaml -f patch.yaml clusterpedia clusterpedia/clusterpedia
```

Get clusterpedia kube-state-metrics services
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

## Future
Multi-cluster kube-state-metrics is a very interesting feature that removes the need to install a single-cluster version of kube-state-metrics in each cluster, and it handles **the issue of differing resource versions very well.**

There is a lot of discussion about this feature here, feel free to comment!
* [The resource state metrics provide different metrics paths depending on the cluster](https://github.com/clusterpedia-io/clusterpedia/issues/544)
* [Support remote write to send resource metrics data](https://github.com/clusterpedia-io/clusterpedia/issues/545)
* [Support for filtering exposed resource state metrics based on namespace](https://github.com/clusterpedia-io/clusterpedia/issues/546)
* [Support for filtering exposed resource state metrics by cluster labels/annotations](https://github.com/clusterpedia-io/clusterpedia/issues/547)

Also welcome to create [a new issue](https://github.com/clusterpedia-io/clusterpedia/issues/new/choose)
