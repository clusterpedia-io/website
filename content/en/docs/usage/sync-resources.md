---
title: "Synchronize Cluster Resources"
weight: 2
---

The main function of Clusterpedia is to provide complex search for resources in multiple clusters.  

Clusterpedia uses the `PediaCluster` resource to specify which resources in the cluster need to support complex search,
and synchronizes these resources onto the `Storage Component` via `Storage Layer` in real time.
```yaml
# example
apiVersion: clusters.clusterpedia.io/v1alpha1
kind: PediaCluster
metadata:
  name: cluster-example
spec:
  apiserverURL: "https://10.30.43.43:6443"
  resources:
  - group: apps
    resources:
     - deployments
  - group: ""
    resources:
     - pods
     - configmsp
  - group: cert-manager.io
    versions:
      - v1
    resources:
      - certificates
```

## Synchronize built-in resources
In order to manage and view these synchronized resources through `PediaCluster`, you need to configure resources in groups
```yaml
resources:
 - group: apps
   versions: []
   resources:
     - deployments
     - daemonsets
```
**For built-in resources, `versions` is not required.**

Clusterpedia will automatically select the appropriate version to synchronize based on the resource version supported in the cluster.

Also, you do not need to worry about version conversion because Clusterpedia will open all version interfaces for built-in resources.
```bash
kubectl get --raw="/apis/pedia.clusterpedia.io/v1alpha1/resources/apis/apps" | jq
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
Clusterpedia supports three versions of `Deployment`: `v1`, `v1beta2`, and `v1beta1`.

## Synchronize custom resources
Compared with built-in resources, custom resources have slightly different configuration on resource versions.
```yaml:
resources:
 - group: cert-manager.io
   versions: []
   resources:
    - certificates
```
**You can also ignore the versions field and then Clusterpedia will synchronize the previous three cluster versions in the Group.**

Take cert-manager.io as an example to get the Group supported by cert-manager.io in an imported cluster
```bash
# Run the command in an imported cluster
kubectl --cluster clusterpedia get --raw="/apis/cert-manager.io" | jq
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
The imported cluster supports four versions for *cert-manager.io*: `v1`, `v1beta1`, `v1alpha3`, `v1alpha2`.  

When `resources.[group].versions` is left blank, Clusterpedia will synchronize three versions `v1`, `v1beta1`, `v1alpah3` in the order of the `APIGroup.versions` list except for `v1alpha2`.

### Specify a sync version for custom resources
If you specified `versions`, the specific resource would be synchronized by `versions`.
```yaml
resources:
 - group: cert-manager.io
   versions:
    - v1beta1
   resources:
    - certificates
```
The above snippet only synchronizes `v1beta1`.

### Usage notes
The custom resource synchronization does not support version conversion currently. The versions are fixed after synchronization.  

If cluster-1 only synchronizes `v1beta1` resources when you are searching for multi-cluster resources, the request to search for version v1 will ignore the version `v1beta1`.  

You are required to learn and handle the different versions in multiple clusters for custom resources.

## View synchronized resources
You can view resources, sync versions, and storage versions by using `Status` of the `PediaCluster` resource.  

For `Status`, a resource may have **Sync Version** and **Storage Version**:
* **Sync Version** refers to the resource version from a synchronized cluster by Clusterpedia
* **Storage Version** refers to the version stored at the storage layer by Clusterpedia

```yaml
status:
  resources:
  - group: apps
    resources:
    - kind: Deployment
      namespaced: true
      resource: deployments
      syncConditions:
      - lastTransitionTime: "2022-01-13T04:34:08Z"
        status: Syncing
        storageVersion: v1
        version: v1
```
In general, **Sync Version** is same as **Storage Version** for a cluster resource.  

However, if an imported cluster only provides the `Deployment` resource of the `v1beta1` version, the **Sync Version** is `v1beta1` and the **Storage Version** is `v1`.  

For example, when synchronizing a Deployment of Kubernetes 1.10, the synchronization status is as follows:
```yaml
status:
  resources:
  - group: apps
    resources:
    - kind: Deployment
      namespaced: true
      resource: deployments
      syncConditions:
      - lastTransitionTime: "2022-01-13T04:34:04Z"
        status: Syncing
        storageVersion: v1
        version: v1beta1
```

For a custom resource, **Synchronized Version** is same as **Storage Version**

## Next
After resource synchronization, you can [Access the Clusterpedia](../access-clusterpedia) to [Search for Resources](../search)
