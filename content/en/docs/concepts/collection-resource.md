---
title: Collection Resource
weight: 10
---

In order to find different resource types at one time, Clusterpedia provides a new resource: `Collection Resource`.

`Collection Resource` is composed of different resources that can be retrieved by types and paginated uniformly.

**What collection resources are supported by the Clusterpedia depends on the `storage layer`.** For example, the `built-in storage layer` temporarily only supports the `workloads` collection resource, which is used to represent workloads.
```bash
kubectl get collectionresources
```
```
# Output:
NAME        RESOURCES
workloads   deployments.apps,daemonsets.apps,statefulsets.apps
```

View the supported `collection resource` in a yaml file
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

For details about `collection resource`, see [Search for collection resource](../../usage/search/collection-resource)

## Custom collection resource
`Built-in storage layer` will support `custom collection resource` in future to enable users to combine any resource types freely.
