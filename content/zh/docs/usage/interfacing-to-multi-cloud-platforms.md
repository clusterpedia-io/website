---
title: 接入多云平台
weight: 10
---

0.4.0 后，Clusterpedia 提供了更加友好的接入多云平台的方式。

用户通过创建 `ClusterImportPolicy` 来自动发现多云平台中纳管的集群，并将这些集群自动同步为 `PediaCluster`，用户不需要根据纳管的集群来手动去维护 `PediaCluster` 了。

我们在 [Clusterpedia 仓库](https://github.com/clusterpedia-io/clusterpedia/tree/main/deploy/clusterimportpolicy) 中维护了各个多云平台的 `ClusterImportPolicy`。
**大家也提交用于对接其他多云平台的 `ClusterImportPolicy`。**

用户在[安装 Clusterpedia](../../installation) 后，创建合适的 `ClusterImportPolicy` 即可，用户也可以根据自己的需求来[创建新的 `ClusterImportPolicy`](#新建-clusterimportpolicy)

## Cluster API ClusterImportPolicy
用户可以参考 [Cluster API Quick Start](https://cluster-api.sigs.k8s.io/user/quick-start.html) 来安装 Cluster API，或者参考 [快速部署 Cluster API + Clusterpedia](https://clusterpedia.io/zh-cn/blog/2022/08/04/cluster-api-searching-has-never-been-easier/#%E5%BF%AB%E9%80%9F%E9%83%A8%E7%BD%B2%E4%B8%80%E5%A5%97-cluster-api-and-clusterpedia-%E7%9A%84%E7%A4%BA%E4%BE%8B%E7%8E%AF%E5%A2%83) 来搭建示例环境。

创建用于对接 Cluster API 平台的 `ClusterImportPolicy`。
```bash
$ kubectl applyf -f https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/main/deploy/clusterimportpolicy/cluster_api.yaml
$ kubectl get clusterimportpolicy
NAME          AGE
cluster-api   4d19h
```

如果集群中已经存在由 Cluster API 创建的集群，那么可以查看 `Cluster` 与 `PediaCluster` 资源
```bash
$ kubectl get cluster
NAME                PHASE         AGE     VERSION
capi-quickstart     Provisioned   3d23h   v1.24.2
capi-quickstart-2   Provisioned   3d23h   v1.24.2

$ kubectl get pediaclusterlifecycle
NAME                        AGE
default-capi-quickstart     3d23h
default-capi-quickstart-2   3d23h

$ kubectl get pediacluster
NAME                        READY   VERSION   APISERVER
default-capi-quickstart     True    v1.24.2
default-capi-quickstart-2   True    v1.24.2
```
[PediaCluster](../../concepts/pediacluster) 会根据 Cluster 自动创建，并且当 Cluster 的 kubeconfig 发生变动时会自动更新 [PediaCluster](../../concepts/pediacluster).

新建一个 Cluster 资源时，Clusterpedia 根据 [Cluster API ClusterImportPolicy](https://github.com/clusterpedia-io/clusterpedia/blob/main/deploy/clusterimportpolicy/cluster_api.yaml) 等到 ControlPlaneInitialized 为 True 时才会自动创建 PediaCluster，可以通过 `kubectl get kubeadmcontrolplane` 来查看集群的初始化状态
```bash
NAME                    CLUSTER           INITIALIZED   API SERVER AVAILABLE   REPLICAS   READY   UPDATED   UNAVAILABLE   AGE   VERSION
capi-quickstart-2xcsz   capi-quickstart   true                                 1                  1         1             86s   v1.24.2
```

Cluster 初始化完成后，就可以直接使用 kubectl 来检索多集群资源了
> 使用 kubectl 前，需要为多集群资源检索生成集群快捷配置 —— [为 kubectl 生成集群访问的快捷配置](../access-clusterpedia#为-kubectl-生成集群访问的快捷配置)
```bash
$ # Since CNI is not installed, the nodes are not ready.
$ kubectl --cluster clusterpedia get no
CLUSTER                     NAME                                            STATUS     ROLES           AGE   VERSION
default-capi-quickstart-2   capi-quickstart-2-ctm9k-g2m87                   NotReady   control-plane   12m   v1.24.2
default-capi-quickstart-2   capi-quickstart-2-md-0-s8hbx-7bd44554b5-kzcb6   NotReady   <none>          11m   v1.24.2
default-capi-quickstart     capi-quickstart-2xcsz-fxrrk                     NotReady   control-plane   21m   v1.24.2
default-capi-quickstart     capi-quickstart-md-0-9tw2g-b8b4f46cf-gggvq      NotReady   <none>          20m   v1.24.2
```

## Karmada ClusterImportPolicy
> 对于 Karmada 平台，用户首先需要先将 Clusterpedia 部署在 Karmada APIServer 中，部署步骤可以参考 https://github.com/Iceber/deploy-clusterpedia-to-karmada

创建用于对接 Karmada 平台的 `ClusterImportPolicy`。
```bash
$ kubectl create -f https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/main/deploy/clusterimportpolicy/karmada.yaml
$ kubectl get clusterimportpolicy
NAME      AGE
karmada   7d5h
```

查看 `Karmada Cluster` 与 `PediaClusterLifecycle`
```bash
$ kubectl get cluster
NAME      VERSION   MODE   READY   AGE
argocd              Push   False   8d
member1   v1.23.4   Push   True    22d
member3   v1.23.4   Pull   True    22d

$ kubectl get pediaclusterlifecycle
NAME              AGE
karmada-argocd    7d5h
karmada-member1   7d5h
karmada-member3   7d5h
```
Clusterpedia 会为每一个 Karmada Cluster 创建对应的 `PediaClusterLifecycle`，可以使用 `kubectl describe pediaclusterlifecycle <name>` 来查看 Karmada Cluster 与 PediaCluster 的转换状态
> 未来会在 `kubectl get pediaclusterlifecycle` 中详细状态

查看成功创建的 `PediaCluster`
```bash
NAME              APISERVER                 VERSION   STATUS
karmada-member1   https://172.18.0.4:6443   v1.23.4   Healthy
```
karmada clusterimportpolicy 要求 karmada 集群为 Push 模式，并且处于 Ready 状态，所以为 *member-1* 集群创建了 *karmada-member-1* pediacluster 资源。

## 新建 ClusterImportPolicy
如果 [Clusterpedia 仓库](https://github.com/clusterpedia-io/clusterpedia/tree/main/deploy/clusterimportpolicy) 中没有维护某个平台的 ClusterImportPolicy，那么我们可以新建 ClusterImportPolicy

**关于 ClusterImportPolicy 原理和字段的详细描述可以参考 [集群自动接入策略](../../concepts/cluster-import-policy)**

现在假定有一个多云平台 **MCP**，该平台使用自定义资源 —— Cluster 来代表被纳管的集群，并且将集群的认证信息保存在和集群同名的 Secret 中
```yaml
apiVersion: cluster.mcp.io
kind: Cluster
metadata:
  name: cluster-1
spec:
  apiEndpoint: "https://172.10.10.10:6443"
  authSecretRef:
    namespace: "default"
    name: "cluster-1"
status:
  conditions:
    - type: Ready
      status: True
---
apiVersion: v1
kind: Secret
metadata:
  name: cluster-1
data:
  ca: **cluster ca bundle**
  token: **cluster token**
```

我们为 **MCP** 平台定义一个 `ClusterImportPolicy` 资源，并且默认同步 pods 和 apps group 下的资源
```yaml
apiVersion: policy.clusterpedia.io/v1alpha1
kind: ClusterImportPolicy
metadata:
  name: mcp
spec:
  source:
    group: "cluster.mcp.io"
    resource: clusters
    versions: []
  references:
    - group: ""
      resource: secrets
      versions: []
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
      syncResourcesRefName: ""
  creationCondition: |
    {{ if ne .source.spec.apiEndpoint "" }}
      {{ range .source.status.conditions }}
        {{ if eq .type "Ready" }}
          {{ if eq .status "True" }} true {{ end }}
        {{ end }}
      {{ end }}
    {{ end }}
```
* `spec.source` 定义了需要监听的资源 Cluster
* `spec.references` 定义了将 MCP Cluster 转换为 PediaCluster 时涉及到的资源，当前只有 secrets 资源
* `spec.nameTemplate` 会根据 MCP Cluster 资源渲染出 PediaCluster 资源的名字
* `spec.template` 通过 MCP Cluster 和 `spec.references` 中定义的资源来渲染出 PediaCluster 资源，具体规则可以参考 [PediaCluster Template](../../concepts/cluster-import-policy#pediacluster-template)
* `spec.creationCondition` 可以根据 MCP Cluster 和 `spec.references` 定义的资源来可以决定了什么时候可以创建 PediaCluster，这里定义了当 MCP Cluster 处于 Ready 后再创建 PediaCluster，具体使用可以参考 [Creation Condition](../../concepts/cluster-import-policy#creation-condition)
