---
title: Interfacing to Multi-Cloud Platforms
weight: 10
---

After 0.4.0, Clusterpedia provides a more friendly way to interface to multi-cloud platforms.

Users can create `ClusterImportPolicy` to automatically discover managed clusters in the multi-cloud platform and automatically synchronize them as `PediaCluster`,
so you don't need to maintain `PediaCluster` manually based on the managed clusters.

We maintain `PediaCluster` for each multi-cloud platform in the [Clusterpedia repository](https://github.com/clusterpedia-io/clusterpedia/tree/main/deploy/clusterimportpolicy). ClusterImportPolicy` for each multi-cloud platform.
**People also submit ClusterImportPolicy to Clusterpedia for interfacing to other multi-cloud platforms.**

After [installing Clusterpedia](../../installation), you can create the appropriate `ClusterImportPolicy`,
or you can [create a new `ClusterImportPolicy`](#new-clusterimportpolicy) according to your needs (multi-cloud platform).

## ClusterAPI ClusterImportPolicy
Users can refer to [Cluster API Quick Start](https://cluster-api.sigs.k8s.io/user/quick-start.html) to install the  Cluster API，or refer to [Quickly deploy Cluster API + Clusterpedia](https://clusterpedia.io/zh-cn/blog/2022/08/04/cluster-api-searching-has-never-been-easier#quickly-deploy-a-sample-environment-for-cluster-api-and-clusterpedia) to deploy a sample environment.

Create `ClusterImportPolicy` for interfacing to the ClusterAPI platform.
```bash
$ kubectl applyf -f https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/main/deploy/clusterimportpolicy/cluster_api.yaml
$ kubectl get clusterimportpolicy
NAME          AGE
cluster-api   4d19h
```

If the clusters created by the ClusterAPI already exists in the management cluster, then you can view the `Cluster` and `PediaCluster` resources.
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
[PediaCluster](../../concepts/pediacluster) is automatically created based on the Cluster, and the kubeconfig of [PediaCluster](../../concepts/pediacluster) is automatically updated when the kubeconfig of the Cluster changes.

When creating a new Cluster, Clusterpedia automatically creates a PediaCluster when **ControlPlaneInitialized** is True according to the [Cluster API ClusterImportPolicy](https://github.com/clusterpedia-io/clusterpedia/blob/main/deploy/clusterimportpolicy/cluster_api.yaml),
and you can check the initialization status of the cluster by using `kubectl get kubeadmcontrolplane`
```bash
NAME                    CLUSTER           INITIALIZED   API SERVER AVAILABLE   REPLICAS   READY   UPDATED   UNAVAILABLE   AGE   VERSION
capi-quickstart-2xcsz   capi-quickstart   true                                 1                  1         1             86s   v1.24.2
```

Once the Cluster has been initialized, you can use kubectl to retrieve multiple cluster resources directly.
> Beforing using kubectl, you need to [generate cluster shortcut configuration](../access-clusterpedia/#configure-the-cluster-shortcut-for-kubectl) for multi-cluster resource retrieval.
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
> For Karmada platform, you need to first deploy Clusterpedia in Karmada APIServer, the deployment steps can be found at https://github.com/Iceber/deploy-clusterpedia-to-karmada

Create `ClusterImportPolicy` for interfacing to the Karmada platform.
```bash
$ kubectl create -f https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/main/deploy/clusterimportpolicy/karmada.yaml
$ kubectl get clusterimportpolicy
NAME      AGE
karmada   7d5h
```

View `Karmada Cluster` and `PediaClusterLifecycle` resources.
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
Clusterpedia creates a corresponding `PediaClusterLifecycle` for each Karmada Cluster,
and you can use `kubectl describe pediaclusterlifecycle <name>` to see the status of the transition between Karmada Cluster and PediaCluster resources.
> The status will be detailed in `kubectl get pediaclusterlifecycle` in the future

View the successfully created `PediaCluster`
```bash
NAME              APISERVER                 VERSION   STATUS
karmada-member1   https://172.18.0.4:6443   v1.23.4   Healthy
```
The karmada clusterimportpolicy requires the karmada cluster to be in Push mode and in Ready state,
so the *karmada-member-1* pediacluster resource is created for the *member-1* cluster.


## New ClusterImportPolicy
If the [Clusterpedia repository](https://github.com/clusterpedia-io/clusterpedia/tree/main/deploy/clusterimportpolicy) does not maintain a ClusterImportPolicy for a platform, then we can create a new ClusterImportPolicy

**A detailed description of the `ClusterImportPolicy` principles and fields can be found in the [Cluster Auto Import Policy](../../concepts/cluster-import-policy)**

Now assume that there is a multi-cloud platform **MCP** that uses a custom resource -- Cluster to represent the managed clusters
and stores the cluster authentication information in a Secret with the same name as the cluster
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

We define a `ClusterImportPolicy` resource for the **MCP** platform and synchronize the pods resource and all resources under the apps group by default.
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
* `spec.source` defines the resource Cluster that needs to be watched to
* `spec.preferences` defines the resources involved in converting an MCP Cluster to a PediaCluster, currently only secrets resources are used
* `spec.nameTemplate` will render the name of the PediaCluster resource based on the MCP Cluster resource
* `spec.template` renders the PediaCluster resource from the resources defined in MCP Cluster and `spec.references`, see [PediaCluster Template](../../concepts/cluster-import-policy#pediacluster-template) for rules
* `spec.creationCondition` determines when a PediaCluster can be created based on the resources defined by the MCP Cluster and `spec.references`, here it defines when the MCP Cluster is Ready before creating the PediaCluster. See [Creation Condition](../../concepts/cluster-import-policy#creation-condition) for details
