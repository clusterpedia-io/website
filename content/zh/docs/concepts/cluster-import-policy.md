---
title: 集群自动接入策略(ClusterImportPolicy)
weight: 15
---
自定义资源 `ClusterImportPolicy` 定义了某种类型的资源应该如何转换成 `PediaCluster`，这样 clusterpedia 便可以根据某种资源来自动的创建、更新和删除 [PediaCluster](../pediacluster)

首先需要在 `ClusterImportPolicy` 资源中定义监听（想要转换的）的资源类型，我们将被监听的资源称为 **Source** 资源。

当一个 **Source** 资源被创建或者删除时，*ClusterImportPolicy Controller* 会创建相应 `PediaClusterLifecycle` 资源。
而 `PediaClusterLifecycle` 会根据具体的 **Source** 资源来创建，更新以及删除 `PediaCluster`。

在创建和更新 `PediaCluster` 时，**Source** 资源可能会将集群的认证信息（例如 CA，Token 等）保存在其他资源中，我们将这些涉及到的其他资源统称为 **References** 资源。

> 对于 `ClusterImportPolicy` 和 `PediaClusterLifecycle` 的调协由 **Clusterpedia Controller Manager** 内的 *ClusterImportPolicy Controller* 和 *PediaClusterLifecycle Controller* 负责
>
> `ClusterImportPolicy` 和 `PediaCusterLifecycle` 资源结构的更新可能会比较频繁，为了避免对 *cluster.clusterpedia.io* group 带来不必要的影响，所以放在 *policy.clusterpedia.io* group 中,
> 未来 1.0 发布时，可能会迁移到 *cluster.clusterpedia.io* 中。

## ClusterImportPolicy
一个完整 `ClusterImportPolicy` 示例如下：
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
主要分为三部分
* `spec.source` 和 `spec.references` 分别定义 **Source** 资源和 **References** 资源
* `spec.nameTemplate` 定义生成的 `PediaClusterLifecycle` 和 `PediaCluster` 的名称，可以根据 **Source** 资源进行渲染
* `spec.template` 和 `spec.creationCondition` 定义资源转换策略。

**references，template，creationCondition 内的模板均支持由 [sprig](https://github.com/Masterminds/sprig) 提供的 70 多种模版函数**，[Sprig Function Documentation](http://masterminds.github.io/sprig/)

## Source 和 References 资源
在 `ClusterImportPolicy` 资源中我们首先需要定义的就是 **Source** 资源类型以及 **References** 资源
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
**Source** 资源通过 `spec.source.group` 和 `spec.source.resource` 指定了资源组和资源名称。我们也可以使用 `spec.source.versions` 来限制 **Source** 的资源版本，默认是不会对 **Source** 资源版本进行限制
> **一种 Source 资源只能有一个负责转换该资源的 `ClusterImportPolicy` 资源**

用户也可以通过 `spec.source.selectorTemplate` 字段来过滤 **Source**
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

**source** 资源可以通过 `{{ .source.<field> }}` 来用于其他的模版字段

`spec.references` 中定义了在转换过程中涉及到的资源，我们需要指定资源的类型，以及通过 namespace 和 name 模版来获取具体的 **reference** 资源。我们也可以和 **Source** 资源一样对 Reference 资源的版本进行限制。

除此之外还要为 **reference** 资源设置一个 key，这个 key 会以 `{{ .references.<key> }}` 的方式来让后续模版字段使用

*`spec.references` 中后面的项也可以通过 `.references.<key>` 来引用前面的项*
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
当一个 **Source** 被创建后，*ClusterImportPolicy Controller* 会根据 `ClusterImportPolicy` 创建相应的 `PediaClusterLifecycle` 资源。

`PediaClusterLifecycle` 资源的名字是通过 `ClusterImportPolicy` 的 `spec.nameTemplate` 字段来设置的。
```
apiVersion: policy.clusterpedia.io/v1alpha1
kind: ClusterImportPolicy
metadata:
  name: multi-cluster-platform
spec:
  nameTemplate: "mcp-{{ .source.metadata.namespace }}-{{ .source.metadata.name }}"
```
nameTemplate 只能根据 **Source** 资源来渲染模版，通常根据是否为 Cluster Scoped 资源来设置该字段：
* `nameTemplate: "<prefix>-{{ .source.metadata.namespace}}-{{ .source.metadata.name }}"`
* `nameTemplate: "<prefix>-{{ .source.metadata.name }}"`

**nameTemplate 前缀通常是多云平台或者其他有意义的名字，当然也可以没有前缀。**

`PediaClusterLifecycle` 中会设置具体的 **source** 资源和 **references** 资源以及转换策略
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
`PediaClusterLifecycle` 的 `spec.source` 设置了一个具体的 **Source** 资源，包括了资源的具体版本，命名空间和名称。

`spec.references` 相比 `ClusterImportPolicy` 包含了具体的资源版本，其他字段和 `ClusterImportPolicy` 内的 References 定义相同。
references 资源的命名空间和名字会在转换资源时进行解析。

### PediaClusterLifecycle 和 PediaCluster
`PediaClusterLifecycle` 的名字会和 `PediaCluster` 进行一一对应，`PediaClusterLifecycle` 根据转换策略创建和更新同名的 `PediaCluster`。

## PediaCluster 转换策略
我们在定义转换策略时，主要关注以下方面:
* 用于创建或者更新 `PediaCluster` 的模版
* 什么时候触发 `PediaCluster` 的创建

在 `ClusterImportPolicy` 中，我们使用 `spec.template` 和 `spec.creationCondition` 来定义他们
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

这两个字段都是模版字段，需要根据 **Source** 资源和 **References** 资源来渲染。

当 **Source** 资源被创建后，**ClusterImportPolicy Controller** 会根据 `ClusterImportPolicy` 创建对应的 `PediaClusterLifecycle`，并且 `PediaClusterLifecycle` 中也会包含转换策略。
> 当然如果 `ClusterImportPolicy` 内的策略发生修改，也会同步给所属的所有 `PediaClusterLifecycle` 资源。
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

`PediaClusterLifecycle` 会负责根据 `spec.creationCondition` 和 `spec.template` 来创建更新具体的 `PediaCluster`。

### Creation Condition
我们有时在 **Source** 资源创建后并不会立刻创建 `PediaCluster`, 而是需要等到 **Source** 资源某些字段或者某些状态就绪后再创建 `PediaCluster`。

`spec.creationCondition` 使用模版语法来判断是否满足创建条件，当模版渲染后的值为 `True`（大小写模糊）后则会根据 `spec.Template` 创建 `PediaCluster`。

如果 `PediaCluster` 已经存在，`spec.creationCondition` 不会影响后续对 `PediaCluster` 的更新。

### PediaCluster Template
`spec.template` 定义了 `PediaCluster` 资源模版，在创建或者更新 `PediaCluster` 时，会根据 **Source** 资源和 **References** 资源来渲染出具体的资源。

`PediaCluster` 模版可以分成三部分：
* 元信息：labels 和 annotations
* 集群访问与认证字段：`spec.apiserver`，`spec.caData`，`spec.tokenData`，`spec.certData`，`spec.keyData` 还有 `spec.kubeconfig`
* 资源同步字段：`spec.syncResources`，`spec.syncAllCustomResources`，`spec.syncResourcesRefName`

`PediaCluster` 资源的**元信息**和**资源同步字段**只有在创建 PediaCluser 时才会有效，在更新 `PediaCluster` 资源时，只会更新**集群访问与认证字段**

### PediaCluster 的删除
如果一个 `PediaCluster` 是由 `PediaClusterLifecycle` 创建，那么会将 `PediaClusterLifecycle` 设置为该 `PediaCluster` 资源的 owner
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
当一个 **Source** 资源被删除时，会同步删除 `PediaClusterLifecycle`，`PediaCluster` 会自动删除。

如果 `PediaCluster` 在 `PediaClusterLifecycle` 前已经出现了，那么 **Source** 删除时不会自动的删除 `PediaCluster`。

未来会增加 DeletionCondition 来允许用户强制或者提前删除 `PediaCluster`。
