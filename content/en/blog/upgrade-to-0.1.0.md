---
title: Upgrade to Clusterpedia 0.1.0
date: 2022-02-15
---
With the release of Clusterpedia 0.1.0, we can now update the earlier 0.0.9-alpha or 0.0.8 to 0.1.0

## Clean Resources
Since the url path to search resources has been modified([#73](https://github.com/clusterpedia-io/clusterpedia/pull/73)), we need to use [cealn-clusterconfigs.sh](https://github.com/clusterpedia-io/clusterpedia/blob/v0.0.9-alpha/hack/clean-clusterconfigs.sh) in 0.0.9-alpha to clean up the cluster shortcut in the *.kube/config*
```bash
curl -sfL https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/v0.0.9-alpha/hack/clean-clusterconfigs.sh | sh -
```

Backup and delete the `PediaCluster` resources.
```bash
kubectl get pediacluster -o clusters.yaml.bak

kubectl delete pediacluster --all
```

After all `PediaCluster` resources have been deleted, remove the `PediaCluster CRD`
```bash
kubectl delete crd pediaclusters.clusters.clusterpedia.io
```

Remove the `APIServices` used to register the Aggregated API
```bash
kubectl delete apiservices v1alpha1.pedia.clusterpedia.io
```

## Upgrade Clusterpedia
Create `PediaCluster CRD`, and upgrade `Clusterpedia APIServer` and `Clustersynchro Manager`.
```bash
DEPLOY_YAML_PATH=https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/v0.1.0/deploy
CRD_YAML_PATH=$DEPLOY_YAML_PATH/crds

kubectl apply -f \
    $CRD_YAML_PATH/cluster.clusterpedia.io_pediaclusters.yaml,\
    $DEPLOY_YAML_PATH/clusterpedia_clustersynchro_manager_deployment.yaml,\
    $DEPLOY_YAML_PATH/clusterpedia_apiserver_deployment.yaml,\
    $DEPLOY_YAML_PATH/clusterpedia_apiserver_apiservice.yaml
```
We can also download the YAML locally, or pull the [clusterpedia](https://github.com/clusterpedia-io/clusterpedia) locally and go to *./deploy* directory and run `kubectl apply`

## Re-import the clusters
Since the APIVersion and schema of `PediaCluster` have been optimized for incompatibility,
it it necessary to recreate `PediaCluster` based on the backed up *clusters.yaml.bak*.

The current example of `PediaCluster`:
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
There are three main changes compared to 0.0.9-alpha:
* `apiVersion`: *clusters.clusterpedia.io/v1alpha1* -> *cluster.clusterpedia.io/v1alpha2*
* `spec.apiserverURL` -> `spec.apiserver`
* `spec.resources` -> `spec.syncResources`

> The specific changes can be viewed: [#70](https://github.com/clusterpedia-io/clusterpedia/pull/70)
> [#67](https://github.com/clusterpedia-io/clusterpedia/pull/67)
> [#76](https://github.com/clusterpedia-io/clusterpedia/pull/76)
> [#77](https://github.com/clusterpedia-io/clusterpedia/pull/77)

Create new pediaclusters based on the old pediaclusters in *clusters.yaml.bak*
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

View clusters status
```bash
kubectl get pediacluster
```

Configure the cluster shortcut for kubectl
```bash
curl -sfL https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/v0.1.0/hack/gen-clusterconfigs.sh | sh -
```
