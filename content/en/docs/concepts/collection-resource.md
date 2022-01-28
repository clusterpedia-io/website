---
title: Collection Resource
weight: 10
---

In order to query multiple types of resources at once, Clusterpedia provides a new resource: `Collection Resource`.

`Collection Resource` is composed of different types of resources, and these resources can be retrieved and paged in a uniform way through the `Collection Resource`.

**What Collection Resources are supported by the Clusterpedia depends on the `Storage Layer`.** For example, the `Default Storage Layer` temporarily only supports the `workloads`, which is used to represent workloads.
```bash
kubectl get collectionresources
```
```
# Output:
NAME        RESOURCES
workloads   deployments.apps,daemonsets.apps,statefulsets.apps
```

View the supported `Collection Resource` in a yaml file
```bash
kubectl get collectionresources -o yaml
```
```yaml
# Output:
apiVersion: v1
items:
- apiVersion: pedia.clusterpedia.io/v1alpha1
  kind: CollectionResource
  metadata:
    creationTimestamp: null
    name: workloads
  reosurceTypes:
  - group: apps
    resource: deployments
    version: v1
  - group: apps
    resource: daemonsets
    version: v1
  - group: apps
    resource: statefulsets
    version: v1
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```
It is found that `workloads` includes three resources: `deployments`, `daemonsets`, and `statefulsets`.

For details about `Collection Resource`, see [Search for Collection Resource](../../usage/search/collection-resource)

## Custom Collection Resource
`Default Storage Layer` will support `Custom Collection Resource` in future to enable users to combine any resource types freely.
