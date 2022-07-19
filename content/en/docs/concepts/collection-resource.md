---
title: Collection Resource
weight: 10
---

In order to query multiple types of resources at once, Clusterpedia provides a new resource: `Collection Resource`.

`Collection Resource` is composed of different types of resources, and these resources can be retrieved and paged in a uniform way through the `Collection Resource`.

**What Collection Resources are supported by the Clusterpedia depends on the `Storage Layer`.** For example, the `Default Storage Layer` temporarily supports the `any`, `workloads` and `kuberesources`.
```bash
kubectl get collectionresources
```
```
# Output:
NAME            RESOURCES
any             *
workloads       deployments.apps,daemonsets.apps,statefulsets.apps
kuberesources   .*,*.admission.k8s.io,*.admissionregistration.k8s.io,*.apiextensions.k8s.io,*.apps,*.authentication.k8s.io,*.authorization.k8s.io,*.autoscaling,*.batch,*.certificates.k8s.io,*.coordination.k8s.io,*.discovery.k8s.io,*.events.k8s.io,*.extensions,*.flowcontrol.apiserver.k8s.io,*.imagepolicy.k8s.io,*.internal.apiserver.k8s.io,*.networking.k8s.io,*.node.k8s.io,*.policy,*.rbac.authorization.k8s.io,*.scheduling.k8s.io,*.storage.k8s.io
```

`any` means any resources, the use need to pass the *groups* or *resources* he wants to combine when using it, for details see [Use Any CollectionResource](../../usage/search/collection-resource#any-collectionresource)

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
    name: any
  resourceTypes: []
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
Clusterpedia plans to provide two ways to let users combine the types of resources they want to query at will.
* **`Any CollectionResource`** —— use `any collectionresource`
* **`CustomCollectionResource`** —— custom collection resource

After 0.4, Clusterpedia provides `any collectionresource` to allow users to combine defferent types of resources by passing `groups` and `resources` parameters.

However, it should be noted that `any collectionresource` cannot be retrieved using kubectl, see [Using Any CollectionResource](../../usage/search/collection-resource#any-collectionresource)
```bash
$ kubectl get collectionresources any
Error from server (BadRequest): url query - `groups` or `resources` is required
```

**Custom Collection Resource** allows users to create or update a Collection Resource via `kubectl apply collectionresource <collectionresource name>`, and users can configure the resource type of the Collection Resource at will.
```yaml
apiVersion: clusterpedia.io/v1beta1
kind: CollectionResource
metadata:
  name: workloads
resourceTypes:
- group: apps
  resource: deployments
- group: apps
  resource: daemonsets
- group: apps
  resource: statefulsets
- group: batch
  resource: cronjobs
```
**Custom Collection Resource are not currently supported**
