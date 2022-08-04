---
title: Cluster API Searching Has Never Been Easier
date: 2022-08-04
---

After 0.4.0, Clusterpedia provides a more friendly way to interface to multi-cloud platforms,
Users simply create or join clusters in the multi-cloud platform and then use Clusterpedia to retrieve the resources within those clusters directly .

> We maintain PediaCluster for each multi-cloud platform in the [Clusterpedia repository](https://github.com/clusterpedia-io/clusterpedia/tree/main/deploy/clusterimportpolicy). You are very welcome to submit ClusterImportPolicy to Clusterpedia for interfacing to other multi-cloud platforms.
>
> After installing Clusterpedia, you can create the appropriate ClusterImportPolicy, or you can [create a new ClusterImportPolicy](https://clusterpedia.io/docs/usage/interfacing-to-multi-cloud-platforms/#new-clusterimportpolicy) according to your needs (multi-cloud platform).

The ClusterImportPolicy for the Cluster API has been submitted in [clusterpedia#288](https://github.com/clusterpedia-io/clusterpedia/pull/288). 
After creating clusters in the Cluster API, you can use Clusterpedia directly to do complex searches of resources within these clusters.

```bash
$ kubectl get cluster
NAME                PHASE         AGE    VERSION
capi-quickstart     Provisioned   10m    v1.24.2
capi-quickstart-2   Provisioned   118s   v1.24.2

$ kubectl get kubeadmcontrolplane
NAME                      CLUSTER             INITIALIZED   API SERVER AVAILABLE   REPLICAS   READY   UPDATED   UNAVAILABLE   AGE   VERSION
capi-quickstart-2-ctm9k   capi-quickstart-2   true                                 1                  1         1             10m   v1.24.2
capi-quickstart-2xcsz     capi-quickstart     true                                 1                  1         1             19m   v1.24.2

$ # the pediacluster resources will automatically create, updates or delete based on cluster resources
$ kubectl get pediacluster -o wide
NAME                        READY   VERSION   APISERVER   VALIDATED   SYNCHRORUNNING   CLUSTERHEALTHY
default-capi-quickstart     True    v1.24.2               Validated   Running          Healthy
default-capi-quickstart-2   True    v1.24.2               Validated   Running          Healthy

$ kubectl --cluster clusterpedia get no
CLUSTER                     NAME                                            STATUS     ROLES           AGE   VERSION
default-capi-quickstart-2   capi-quickstart-2-ctm9k-g2m87                   NotReady   control-plane   12m   v1.24.2
default-capi-quickstart-2   capi-quickstart-2-md-0-s8hbx-7bd44554b5-kzcb6   NotReady   <none>          11m   v1.24.2
default-capi-quickstart     capi-quickstart-2xcsz-fxrrk                     NotReady   control-plane   21m   v1.24.2
default-capi-quickstart     capi-quickstart-md-0-9tw2g-b8b4f46cf-gggvq      NotReady   <none>          20m   v1.24.2
```


## Quickly deploy a sample evironment for Cluster API With Clusterpedia

### Prerequisites
* Install and setup [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) in your local environment
* Install [Kind](https://kind.sigs.k8s.io/) and [Docker](https://www.docker.com/)
* Install [clusterctl](https://cluster-api.sigs.k8s.io/user/quick-start.html#install-clusterctl)

> Minimum kind supported version: v0.14.0

### Create a management cluster, and deploy the Cluster API
> Deploying the Cluster API can also be found in https://cluster-api.sigs.k8s.io/user/quick-start.html

```bash
$ cat > kind-cluster-with-extramounts.yaml <<EOF
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraMounts:
    - hostPath: /var/run/docker.sock
      containerPath: /var/run/docker.sock
EOF

$ kind create cluster --name capi-sample --config kind-cluster-with-extramounts.yaml

$ export CLUSTER_TOPOLOGY=true
$ clusterctl init --infrastructure docker
```

### Deploy Clusterpedia
```bash
$ git clone https://github.com/clusterpedia-io/clusterpedia.git && cd clusterpedia/charts
$ helm install clusterpedia . \
  --namespace clusterpedia-system \
  --create-namespace \
  --set installCRDs=true \
  # --set persistenceMatchNode={{ LOCAL_PV_NODE }}
  --set persistenceMatchNode=capi-sample-control-plane
```
> The Clusterpedia Chart creates a local pv for the storage component, but you need to specify the node using the persistenceMatchNode option, eg. --set persistenceMatchNode=master-1.
> 
> If you don't need to create a local pv, add the --set persistenceMatchNode=None flag.
> [Lean More](https://github.com/clusterpedia-io/clusterpedia/tree/main/charts)

Create [ClusterImportPolicy](https://clusterpedia.io/docs/concepts/cluster-import-policy/) for interfacing to the Cluster API
```bash
$ kubectl apply -f https://raw.githubusercontent.com/Iceber/clusterpedia/add_cluster_api_clusterimportpolicy/deploy/clusterimportpolicy/cluster_api.yaml
```
> Clusterpedia can be integrated into any multi-cloud management platform, [Lean More](https://clusterpedia.io/docs/usage/interfacing-to-multi-cloud-platforms/)

[Gen cluster shortcut for kubectl](https://clusterpedia.io/docs/usage/access-clusterpedia/#configure-the-cluster-shortcut-for-kubectl), If you use client-go or OpenAPI to access, you can omit this step

```bash
$ curl -sfL https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/main/hack/gen-clusterconfigs.sh | sh -

$ # Using kubectl to retrieve multicluster resources, the current Cluster API does not create a cluster, so it returns null
$ kubectl --cluster clusterpedia api-resources
```

## Create a cluster using the Cluster API
> When using the sample enironments' Docker Provider to create a cluster, you need to add `--flavor development` flag.

```bash
$ clusterctl generate cluster capi-quickstart --flavor development \
  --kubernetes-version v1.24.2 \
  --control-plane-machine-count=1 \
  --worker-machine-count=1 \
  > capi-quickstart.yaml
$ kubectl apply -f ./capi-quickstart.yaml
```

### View cluster creation status
```bash
$ kubectl get cluster
NAME              PHASE         AGE   VERSION
capi-quickstart   Provisioned   8s    v1.24.2

$ kubectl get kubeadmcontrolplane -w
NAME                    CLUSTER           INITIALIZED   API SERVER AVAILABLE   REPLICAS   READY   UPDATED   UNAVAILABLE   AGE   VERSION
capi-quickstart-2xcsz   capi-quickstart   True                                 1                  1         1             86s   v1.24.2
```

**when kubeadmcontrolplane's Initialized is true**, lusterpedia will automatically synchronize the resources in the cluster, you can use `kubectl --cluster clusterpedia get` to search the resources.
```bash
$ kubectl get pediacluster
NAME                      READY   VERSION   APISERVER
default-capi-quickstart   True    v1.24.2

$ kubectl --cluster clusterpedia get pod -A
NAMESPACE     CLUSTER                   NAME                                                  READY   STATUS    RESTARTS   AGE
kube-system   default-capi-quickstart   kube-apiserver-capi-quickstart-2xcsz-fxrrk            1/1     Running   0          2m32s
kube-system   default-capi-quickstart   kube-scheduler-capi-quickstart-2xcsz-fxrrk            1/1     Running   0          2m31s
kube-system   default-capi-quickstart   coredns-6d4b75cb6d-lrwj4                              0/1     Pending   0          2m20s
kube-system   default-capi-quickstart   kube-proxy-p8v9m                                      1/1     Running   0          2m20s
kube-system   default-capi-quickstart   kube-controller-manager-capi-quickstart-2xcsz-fxrrk   1/1     Running   0          2m32s
kube-system   default-capi-quickstart   etcd-capi-quickstart-2xcsz-fxrrk                      1/1     Running   0          2m32s
kube-system   default-capi-quickstart   kube-proxy-2ln2w                                      1/1     Running   0          105s
kube-system   default-capi-quickstart   coredns-6d4b75cb6d-2hcmz                              0/1     Pending   0          2m20s
```
The [cluster-api clusterimportpolicy](https://github.com/clusterpedia-io/clusterpedia/pull/288/files) sets the resources to be synchronized by default in the cluster.

Users can also manually modify the configuration of synchronization in pediacluster, [Synchronize Cluster Resources](https://clusterpedia.io/docs/usage/sync-resources/)

When the cluster is deleted in the Cluster API, Clusterpedia also deletes [PeidaCluster](https://clusterpedia.io/docs/concepts/pediacluster/) at the same time.

## Resources retrieval for multiple clusters
Use the above steps to create multiple clusters
```bash
$ kubectl get cluster
NAME                PHASE         AGE    VERSION
capi-quickstart     Provisioned   10m    v1.24.2
capi-quickstart-2   Provisioned   118s   v1.24.2

$ kubectl get kubeadmcontrolplane
NAME                      CLUSTER             INITIALIZED   API SERVER AVAILABLE   REPLICAS   READY   UPDATED   UNAVAILABLE   AGE   VERSION
capi-quickstart-2-ctm9k   capi-quickstart-2   true                                 1                  1         1             10m   v1.24.2
capi-quickstart-2xcsz     capi-quickstart     true                                 1                  1         1             19m   v1.24.2

$ # the pediacluster resources will automatically create, updates or delete based on cluster resources
$ kubectl get pediacluster -o wide
NAME                        READY   VERSION   APISERVER   VALIDATED   SYNCHRORUNNING   CLUSTERHEALTHY
default-capi-quickstart     True    v1.24.2               Validated   Running          Healthy
default-capi-quickstart-2   True    v1.24.2               Validated   Running          Healthy

$ kubectl --cluster clusterpedia get no
CLUSTER                     NAME                                            STATUS     ROLES           AGE   VERSION
default-capi-quickstart-2   capi-quickstart-2-ctm9k-g2m87                   NotReady   control-plane   12m   v1.24.2
default-capi-quickstart-2   capi-quickstart-2-md-0-s8hbx-7bd44554b5-kzcb6   NotReady   <none>          11m   v1.24.2
default-capi-quickstart     capi-quickstart-2xcsz-fxrrk                     NotReady   control-plane   21m   v1.24.2
default-capi-quickstart     capi-quickstart-md-0-9tw2g-b8b4f46cf-gggvq      NotReady   <none>          20m   v1.24.2
```

Clusterpedia supports two types of resource search:
```bash
$ kubectl api-resources | grep clusterpedia.io
collectionresources     clusterpedia.io/v1beta1  false   CollectionResource
resources               clusterpedia.io/v1beta1  false   Resources
```

* [Resources that are compatible with Kubernetes OpenAPI](https://clusterpedia.io/docs/usage/access-clusterpedia/#access-the-clusterpedia-resources)

```bash
$ kubectl --cluster clusterpedia get cm -A
NAMESPACE         CLUSTER                     NAME                                 DATA   AGE
kube-system       default-capi-quickstart     extension-apiserver-authentication   6      19m
kube-system       default-capi-quickstart     kubeadm-config                       1      19m
kube-public       default-capi-quickstart     cluster-info                         2      19m
kube-system       default-capi-quickstart     kube-proxy                           2      19m
kube-node-lease   default-capi-quickstart     kube-root-ca.crt                     1      19m
kube-system       default-capi-quickstart-2   extension-apiserver-authentication   6      10m
kube-system       default-capi-quickstart     kubelet-config                       1      19m
kube-system       default-capi-quickstart     coredns                              1      19m
kube-system       default-capi-quickstart     kube-root-ca.crt                     1      19m
kube-public       default-capi-quickstart     kube-root-ca.crt                     1      19m
kube-system       default-capi-quickstart-2   coredns                              1      10m
default           default-capi-quickstart     kube-root-ca.crt                     1      19m
kube-system       default-capi-quickstart-2   kube-proxy                           2      10m
kube-system       default-capi-quickstart-2   kubeadm-config                       1      10m
kube-system       default-capi-quickstart-2   kubelet-config                       1      10m
kube-system       default-capi-quickstart-2   kube-root-ca.crt                     1      10m
kube-node-lease   default-capi-quickstart-2   kube-root-ca.crt                     1      10m
kube-public       default-capi-quickstart-2   cluster-info                         3      10m
kube-public       default-capi-quickstart-2   kube-root-ca.crt                     1      10m
default           default-capi-quickstart-2   kube-root-ca.crt                     1      10m

$ # gen cluster shortcuts
$ curl -sfL https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/main/hack/gen-clusterconfigs.sh | sh -
$ kubectl --cluster default-capi-quickstart get cm -n kube-system
```

* [Collection Resource](https://clusterpedia.io/docs/concepts/collection-resource/)

Clusterpedia can also perform more advanced aggregation of resources. For example, you can use Collection Resource to get a set of different resources at once.
```bash
$ kubectl get collectionresources
NAME            RESOURCES
any             *
workloads       apps.deployments,apps.daemonsets,apps.statefulsets
kuberesources   .*,admission.k8s.io.*,admissionregistration.k8s.io.*,apiextensions.k8s.io.*,apps.*,authentication.k8s.io.*,authorization.k8s.io.*,autoscaling.*,batch.*,certificates.k8s.io.*,coordination.k8s.io.*,discovery.k8s.io.*,events.k8s.io.*,extensions.*,flowcontrol.apiserver.k8s.io.*,imagepolicy.k8s.io.*,internal.apiserver.k8s.io.*,networking.k8s.io.*,node.k8s.io.*,policy.*,rbac.authorization.k8s.io.*,scheduling.k8s.io.*,storage.k8s.io.*

$ kubectl get collectionresources workloads
```

### [Search](https://github.com/clusterpedia-io/clusterpedia#search-label-and-url-query)
* [Search by metadata(clusters, namespaces, resource names, creation time interval](https://clusterpedia.io/docs/usage/search/#search-by-metadata)
```bash
$ kubectl --cluster clusterpedia get cm -A -l \
    "search.clusterpedia.io/clusters in (default-capi-quickstart,default-capi-quickstart-2),\
    search.clusterpedia.io/namespaces in (kube-system,default)"
NAMESPACE     CLUSTER                     NAME                                 DATA   AGE
kube-system   default-capi-quickstart     extension-apiserver-authentication   6      23m
kube-system   default-capi-quickstart     kubeadm-config                       1      23m
kube-system   default-capi-quickstart     kube-proxy                           2      23m
kube-system   default-capi-quickstart-2   extension-apiserver-authentication   6      14m
kube-system   default-capi-quickstart     kubelet-config                       1      23m
kube-system   default-capi-quickstart     coredns                              1      23m
kube-system   default-capi-quickstart     kube-root-ca.crt                     1      23m
kube-system   default-capi-quickstart-2   coredns                              1      14m
default       default-capi-quickstart     kube-root-ca.crt                     1      23m
kube-system   default-capi-quickstart-2   kube-proxy                           2      14m
kube-system   default-capi-quickstart-2   kubeadm-config                       1      14m
kube-system   default-capi-quickstart-2   kubelet-config                       1      14m
kube-system   default-capi-quickstart-2   kube-root-ca.crt                     1      14m
default       default-capi-quickstart-2   kube-root-ca.crt                     1      14m
```

* [Fuzzy Search](https://clusterpedia.io/docs/usage/search/multi-cluster/#fuzzy-search)
* [Enhanced Field Selector](https://clusterpedia.io/docs/usage/search/multi-cluster/#field-selector)
* [Search by Parent or Ancestor Owner](https://clusterpedia.io/docs/usage/search/multi-cluster/#search-by-parent-or-ancestor-owner)
* [Paging and Sorting](https://clusterpedia.io/docs/usage/search/multi-cluster/#paging-and-sorting)
* [Advanced Search](https://clusterpedia.io/docs/usage/search/#advanced-searchcustom-conditional-search)
