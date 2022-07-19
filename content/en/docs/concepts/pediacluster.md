---
title: PediaCluster
weight: 1
---
Clusterpedia uses the `PediaCluster` resource to represent a cluster that needs to synchronize and retrieve resources.
> Clusterpedia needs to be very friendly to other multi-cloud platforms that may use `Cluster` resource to represent managed clusters,
> to avoid conflicts Clusterpedia uses `PediaCluster`.
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

PediaCluster has two uses:
* Configure authentication information for the cluster
* Configure resources for synchronization

Configuring cluster authentication information can be found in [Import Clusters](../../usage/import-clusters)

There are three fields to configure the resources to be synchronized.
* `spec.syncResources` configures the resources that need to be synchronized for this cluster
* `spec.syncAllCustomResources` synchronizes all custom resources
* `spec.syncResourcesRefName` references [Public Configuration of Cluster Sync Resources](../cluster-sync-resources)

For details on configuring synchronization resources, see [Synchronize Cluster Resources](../../usage/sync-resources)
