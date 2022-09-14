---
title: Cluster Auto Import Policy
weight: 15
---
The custom resource `ClusterImportPolicy` defines how a certain type of resource should be converted into a `PediaCluster`,
so that clusterpedia can automatically create, update and delete [PediaCluster](../pediacluster) based on a certain resource.

First we need to define the type of resource (that you want to convert) to watch to in the `ClusterImportPolicy` resource , we will call the resource being watched to the **Source** resource.

When a **Source** resource is created or deleted, the *ClusterImportPolicy Controller* creates the corresponding `PediaClusterLifecycle` resource.

The custom resource `PediaClusterLifecycle` creates, updates and deletes `PediaCluster` based on the specific **Source** resource.

When creating and updating a `PediaCluster`, the **Source** resource may store the cluster's authentication information (eg. CA, Token, etc.) in other resources, which are collectively referred to as **Reference** resources.

> The reconciliation of `ClusterImportPolicy` and `PediaClusterLifecycle` is done by the *ClusterImportPolicy Controller* and *PediaClusterLifecycle Controller* within the **Clusterpedia Controller Manager**
>
> `ClusterImportPolicy` and `PediaClusterLifecycle` resource structures may be updated more frequently, so to avoid unnecessary impact on the *cluster.clusterpedia.io* group, they are placed in the *policy.clusterpedia.io* group.
> It may be migrated to *cluster.clusterpedia.io* in the future 1.0 release.

## ClusterImportPolicy
An example of a complete `ClusterImportPolicy` resource is as follows:
```yaml
apiVersion: policy.clusterpedia.io/v1alpha1
kind: ClusterImportPolicy
metadata:
  name: mcp
spec:
  source:
    group: "cluster.example.io"
    resource: clusters
    selectorTemplate: ""
  references:
    - group: ""
      resource: secrets
      namespaceTemplate: "{{ .source.spec.authSecretRef.namespace }}"
      nameTemplate: "{{ .source.spec.authSecretRef.name }}"
      key: authSecret
  nameTemplate: "mcp-{{ .source.metadata.name }}"
  template: |
    spec:
      apiserver: "{{ .source.spec.apiEndpoint }}"
      caData: "{{ .references.authSecret.data.ca }}"
      tokenData: "{{ .references.authSecret.data.token }}"
      syncResources:
        - group: ""
          resources:
          - "pods"
        - group: "apps"
          resources:
          - "*"
  creationCondition: |
    {{ if ne .source.spec.apiEndpoint "" }}
      {{ range .source.status.conditions }}
        {{ if eq .type "Ready" }}
          {{ if eq .status "True" }} true {{ end }}
        {{ end }}
      {{ end }}
    {{ end }}
```
There are these sections:

* `spec.source` and `spec.references` define **Source** resource type and **Reference** resources
* `spec.nameTemplate` defines the name of the generated `PediaClusterLifecycle` and `PediaCluster`, which can be rendered according to the **Source** resource.
* `spec.template` and `spec.creationCondition` define the resource conversion policy.

**Templates within references, template, and creationCondition all support the 70+ template functions provided by [sprig](https://github.com/Masterminds/sprig)**, [Sprig Function Documentation](http://masterminds.github.io/sprig/)

## Source and References Resource
The first thing we need to define in the `ClusterImportPolicy` resource is the **Source** resource type and the **Reference** resources.
```yaml
apiVersion: policy.clusterpedia.io/v1alpha1
kind: ClusterImportPolicy
metadata:
  name: multi-cluster-polatform
spec:
  source:
    group: "example.io"
    resource: clusters
    versions: []
    selectorTemplate: |
      {{ if eq .source.metadata.namespace "default" }} true {{ end }}
  references:
    - group: ""
      resource: secrets
      versions: []
      namespaceTemplate: "{{ .source.spec.secretRef.namespace }}"
      nameTemplate: "{{ .source.spec.secretRef.name }}"
      key: secret
```
The **Source** resource specifies the resource group and resource name via `spec.source.group` and `spec.source.resource`. We can also use `spec.source.versions` to restrict **Source** resource versions, by default there is no restriction on **Source** resource versions
> **A Source resource can only have one `ClusterImportPolicy` resource responsible for converting that resource**

You can also filter the **Source** by the `spec.source.selectorTemplate` field.
```yaml
apiVersion: policy.clusterpedia.io/v1alpha1
kind: ClusterImportPolicy
metadata:
  name: kubevela
spec:
  source:
    group: ""
    resource: secrets
    selectorTemplate: |
      {{ if eq .source.metadata.namespace "vela-system" }}
        {{ if .source.metadata.labels }}
          {{ eq (index .source.metadata.labels "cluster.core.oam.dev/cluster-credential-type") "X509Certificate" }}
        {{ end }}
      {{ end }}
```

**source** resources can be used for other template fields via `{{ .source.<field> }}`

The resources involved in the conversion process are defined in `spec.references` and we need to specify the type of the resource and the specific **reference** resource via namespace and name templates. We can also restrict the version of the Reference resource in the same way as the **Source** resource.

In addition to this, we need to set a key for the **reference** resource, which will be used by subsequent template fields as `{{ .references.<key> }}`

*The later items in *`spec.references` can also refer to the earlier ones via `.references.<key>`
```yaml
spec:
  references:
  - group: a.example.io
    resource: aresource
    namespaceTemplate: "{{ .source.spec.aNamespace }}"
    nameTemplate: "{{ .source.spec.aName }}"
    key: refA
  - group: b.example.io
    resource: bresource
    namespaceTemplate: "{{ .references.refA.spec.bNamespace }}"
    nameTemplate: "{{ .references.refA.spec.bName }}"
    key: refB
```

## PediaClusterLifecycle
When a **Source** is created, the *ClusterImportPolicy Controller* creates the corresponding `PediaClusterLifecycle` resource based on the `ClusterImportPolicy` resource.

The name of the `PediaClusterLifecycle` resource is set via the `spec.nameTemplate` field of the `ClusterImportPolicy`.
```
apiVersion: policy.clusterpedia.io/v1alpha1
kind: ClusterImportPolicy
metadata:
  name: multi-cluster-platform
spec:
  nameTemplate: "mcp-{{ .source.metadata.namespace }}-{{ .source.metadata.name }}"
```
The nameTemplate can only render templates based on **Source** resources, and the field is usually set based on whether it is a Cluster Scoped resource.
* `nameTemplate: "<prefix>-{{ .source.metadata.namespace}}-{{ .source.metadata.name }}"`
* `nameTemplate: "<prefix>-{{ .source.metadata.name }}"`

**nameTemplate is usually prefixed with a multi-cloud platform or other meaningful name, but can of course be set without a prefix. **

The `PediaClusterLifecycle` sets the specific **source** resource and **references** resource and the conversion policy
```yaml
apiVersion: policy.clusterpedia.io/v1alpha1
kind: PediaClusterLifecycle
metadata:
  name: <prefix>-example
spec:
  source:
    group: example.io
    version: v1beta1
    resource: clusters
    namespace: ""
    name: example
  references:
    - group: ""
      resource: secrets
      version: v1
      namespaceTemplate: "{{ .source.spec.secretRef.namespace }}"
      nameTemplate: "{{ .source.spec.secretRef.name }}"
      key: secret
```
The `spec.source` of the `PediaClusterLifecycle` sets a specific **Source** resource, including the specific version, namespace and name of the resource.


`spec.references` contains the specific resource version compared to `ClusterImportPolicy`, the other fields are the same as the References definition within `ClusterImportPolicy`.

The namespace and name of the references resource will be resolved when converting the resource.

### PediaClusterLifecycle 和 PediaCluster
The name of `PediaClusterLifecycle` corresponds to `PediaCluster`, and `PediaClusterLifecycle` creates and updates `PediaCluster` with the same name according to the conversion policy.

## PediaCluster Convertion Policy
We focus on the following aspects when defining the conversion policy:
* the template used to create or update the `PediaCluster`
* When to trigger the creation of a `PediaCluster

In `ClusterImportPolicy`, we use `spec.template` and `spec.creationCondition` to define them
```yaml
apiVersion: policy.clusterpedia.io/v1alpha1
kind: ClusterImportPolicy
metadata:
  name: mcp
spec:
... other fields
  template: |
    spec:
      apiserver: "{{ .source.spec.apiEndpoint }}"
      caData: "{{ .references.authSecret.data.ca }}"
      tokenData: "{{ .references.tokenData.data.token }}"
      syncResources:
        - group: ""
          resources:
          - "pods"
        - group: "apps"
          resources:
          - "*"
  creationCondition: |
    {{ if ne .source.spec.apiEndpoint "" }}
      {{ range .source.status.conditions }}
        {{ if eq .type "Ready" }}
          {{ if eq .status "True" }} true {{ end }}
        {{ end }}
      {{ end }}
    {{ end }}
```
Both of these fields are template fields that need to be rendered based on **Source** resource and **References** resources.

When the **Source** resource is created, the **ClusterImportPolicy Controller** will create the corresponding `PediaClusterLifecycle` based on the `ClusterImportPolicy`, and the `PediaClusterLifecycle` will will also contain the conversion policy.
> Of course, if the policy in `ClusterImportPolicy` is modified, it will be synchronized to all `PediaClusterLifecycle` resources it belongs to.
```yaml
apiVersion: policy.clusterpedia.io/v1alpha1
kind: ClusterImportPolicy
metadata:
  name: mcp-example
spec:
... other fields
  template: |
    spec:
      apiserver: "{{ .source.spec.apiEndpoint }}"
      caData: "{{ .references.authSecret.data.ca }}"
      tokenData: "{{ .references.tokenData.data.token }}"
      syncResources:
        - group: ""
          resources:
          - "pods"
        - group: "apps"
          resources:
          - "*"
  creationCondition: |
    {{ if ne .source.spec.apiEndpoint "" }}
      {{ range .source.status.conditions }}
        {{ if eq .type "Ready" }}
          {{ if eq .status "True" }} true {{ end }}
        {{ end }}
      {{ end }}
    {{ end }}
```
PediaClusterLifecycle` is responsible for creating and updating specific `PediaClusters` based on `spec.creationCondition` and `spec.template`.

### Creation Condition
Sometimes we don't create a `PediaCluster` immediately after a **Source** resource is created, but rather we need to wait until some fields or some state of the **Source** resource is ready before creating the `PediaCluster`.

`spec.creationCondition` uses the template syntax to determine if the creation condition is met,
and when the template is rendered with a value of `True` (case insensitive) then `PediaCluster` is created according to `spec.

If the `PediaCluster` already exists, `spec.creationCondition` will not affect updates to the `PediaCluster`.

### PediaCluster Template
`spec.template` defines a `PediaCluster` resource template that renders specific resources based on **Source** resource and **References** resources when creating or updating a `PediaCluster`.

The `PediaCluster` template can be separated into three parts.
* Metadata: labels and annotations
* Cluster endpoint and authentication fields: `spec.apiserver`, `spec.caData`, `spec.tokenData`, `spec.certData`, `spec.keyData` and `spec.kubeconfig`
* Resource sync fields: `spec.syncResources`, `spec.syncAllCustomResources`, `spec.syncResourcesRefName`


The **metadata** and **resource sync fields** of the `PediaCluster` resource are only available when the PediaCluser is created, and only the **cluster endpoint and authentication fields** are updated when the `PediaCluster` resource is updated

### PediaCluster Deletion Condition
If a `PediaCluster` is created by `PediaClusterLifecycle`, then `PediaClusterLifecycle` will be set to the owner of that `PediaCluster` resource.
```yaml
apiVersion: cluster.clusterpedia.io/v1alpha2
kind: PediaCluster
metadata:
  name: mcp-example
  ownerReferences:
  - apiVersion: policy.clusterpedia.io/v1alpha1
    kind: PediaClusterLifecycle
    name: mcp-example
    uid: f932483a-b1c5-4894-a524-00f78ea34a9f
```
When a **Source** resource is deleted, the `PediaClusterLifecycle` is deleted at the same time and the `PediaCluster` is deleted automatically.

If `PediaCluster` already existed before `PediaClusterLifecycle`, then **Source** will not automatically delete `PediaCluster` when it is deleted.

DeletionCondition will be added in the future to allow users to force or preempt the deletion of `PediaCluster`.
