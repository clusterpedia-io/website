---
title: Collection Resource
weight: 10
---

In order to query multiple types of resources at once, Clusterpedia provides a new resource: `Collection Resource`.

`Collection Resource` is composed of different types of resources, and these resources can be retrieved and paged in a uniform way through the `Collection Resource`.

**What Collection Resources are supported by the Clusterpedia depends on the `Storage Layer`.** For example, the `Default Storage Layer` temporarily supports the `workloads` and `kuberesources`.
```bash
kubectl get collectionresources
```
```
# Output:
NAME            RESOURCES
workloads       deployments.apps,daemonsets.apps,statefulsets.apps
kuberesources   *,*.admission.k8s.io,*.admissionregistration.k8s.io,*.apiextensions.k8s.io,*.apps,*.authentication.k8s.io,*.authorization.k8s.io,*.autoscaling,*.batch,*.certificates.k8s.io,*.coordination.k8s.io,*.discovery.k8s.io,*.events.k8s.io,*.extensions,*.flowcontrol.apiserver.k8s.io,*.imagepolicy.k8s.io,*.internal.apiserver.k8s.io,*.networking.k8s.io,*.node.k8s.io,*.policy,*.rbac.authorization.k8s.io,*.scheduling.k8s.io,*.storage.k8s.io
```
`kuberesources` contains all of kube's built-in resources, and we can use `kuberesources` to filter and search all of theme in a uniform api.

View the supported `Collection Resource` in a yaml file
```bash
kubectl get collectionresources -o yaml
```
```yaml
# Output:
apiVersion: v1
items:
- apiVersion: clusterpedia.io/v1beta1
  kind: CollectionResource
  metadata:
    creationTimestamp: null
    name: workloads
  resourceTypes:
  - group: apps
    resource: deployments
    version: v1
  - group: apps
    resource: daemonsets
    version: v1
  - group: apps
    resource: statefulsets
    version: v1
- apiVersion: clusterpedia.io/v1beta1
  kind: CollectionResource
  metadata:
    creationTimestamp: null
    name: kuberesources
  resourceTypes:
  - group: ""
  - group: admission.k8s.io
  - group: admissionregistration.k8s.io
  - group: apiextensions.k8s.io
  - group: apps
  - group: authentication.k8s.io
  - group: authorization.k8s.io
  - group: autoscaling
  - group: batch
  - group: certificates.k8s.io
  - group: coordination.k8s.io
  - group: discovery.k8s.io
  - group: events.k8s.io
  - group: extensions
  - group: flowcontrol.apiserver.k8s.io
  - group: imagepolicy.k8s.io
  - group: internal.apiserver.k8s.io
  - group: networking.k8s.io
  - group: node.k8s.io
  - group: policy
  - group: rbac.authorization.k8s.io
  - group: scheduling.k8s.io
  - group: storage.k8s.io
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```
It is found that `workloads` includes three resources: `deployments`, `daemonsets`, and `statefulsets`.

And `kuberesources` contains all of kube's built-in resources.

For details about `Collection Resource`, see [Search for Collection Resource](../../usage/search/collection-resource)

## Custom Collection Resource
`Default Storage Layer` will support `Custom Collection Resource` in future to enable users to combine any resource types freely.
