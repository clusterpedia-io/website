---
title: Cluster API Searching Has Never Been Easier
date: 2022-08-04
---

0.4.0 后，Clusterpedia 提供了更加友好的接入多云平台的方式，用户在多云平台创建或者纳管集群后，便可以直接使用 kubectl 来检索这些集群内的资源。

> 我们在 [Clusterpedia 仓库](https://github.com/clusterpedia-io/clusterpedia/tree/main/deploy/clusterimportpolicy) 中维护了各个多云平台的 [ClusterImportPolicy](https://clusterpedia.io/zh-cn/docs/concepts/cluster-import-policy/)。 非常欢迎大家提交用于对接其他多云平台的 ClusterImportPolicy。
>
> 用户在安装 Clusterpedia 后，创建合适的 ClusterImportPolicy 即可，用户也可以根据自己的需求来[创建新的 ClusterImportPolicy](https://clusterpedia.io/docs/usage/interfacing-to-multi-cloud-platforms/#new-clusterimportpolicy)

Cluster API 的 ClusterImportPolicy 已经在 [clusterpedia#288](https://github.com/clusterpedia-io/clusterpedia/pull/288) 中提交, 在 Cluster API 中创建集群后，可以直接使用 Clusterpedia 来对这些集群内的资源进行复杂检索。

```bash
$ kubectl get cluster
NAME                PHASE         AGE    VERSION
capi-quickstart     Provisioned   10m    v1.24.2
capi-quickstart-2   Provisioned   118s   v1.24.2

$ kubectl get kubeadmcontrolplane
NAME                      CLUSTER             INITIALIZED   API SERVER AVAILABLE   REPLICAS   READY   UPDATED   UNAVAILABLE   AGE   VERSION
capi-quickstart-2-ctm9k   capi-quickstart-2   true                                 1                  1         1             10m   v1.24.2
capi-quickstart-2xcsz     capi-quickstart     true                                 1                  1         1             19m   v1.24.2

$ # pediacluster 会根据 cluster 资源自动创建，更新和删除
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


## 快速部署一套 Cluster API And Clusterpedia 的示例环境

### 预备条件
* 安装 [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) 到本地环境
* 安装 [Kind](https://kind.sigs.k8s.io/) and [Docker](https://www.docker.com/)
* 安装 [clusterctl](https://cluster-api.sigs.k8s.io/user/quick-start.html#install-clusterctl)

> Minimum kind supported version: v0.14.0

### 创键管理集群并部署 Cluster API
> 部署 Cluster API 也可以参考 https://cluster-api.sigs.k8s.io/user/quick-start.html

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

### 部署 Clusterpedia
```bash
$ git clone https://github.com/clusterpedia-io/clusterpedia.git && cd clusterpedia/charts
$ helm install clusterpedia . \
  --namespace clusterpedia-system \
  --create-namespace \
  --set installCRDs=true \
  # --set persistenceMatchNode={{ LOCAL_PV_NODE }}
  --set persistenceMatchNode=capi-sample-control-plane
```
> clusterpedia charts 提供了 Local PV，需要创建 LOCAL PV 绑定的节点.
> 如果不需要 charts 来创建 LOCAL PV，可以使用 `--set persistenceMatchNode=None`.
> [详见](https://github.com/clusterpedia-io/clusterpedia/tree/main/charts)

创建用于接入 Cluster API 的[集群自动导入策略](https://clusterpedia.io/docs/concepts/cluster-import-policy/)
```bash
$ kubectl apply -f https://raw.githubusercontent.com/Iceber/clusterpedia/add_cluster_api_clusterimportpolicy/deploy/clusterimportpolicy/cluster_api.yaml
```
> Clusterpedia 可以接入任何的多云管理平台，接入方式可以参考 [Interfacing to Multi-Cloud Platforms](https://clusterpedia.io/docs/usage/interfacing-to-multi-cloud-platforms/)

[生成 kubectl cluster shortcut](https://clusterpedia.io/docs/usage/access-clusterpedia/#configure-the-cluster-shortcut-for-kubectl)，如果使用 client-go 或者 OpenAPI 来访问，可以省略该步骤
```bash
$ curl -sfL https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/main/hack/gen-clusterconfigs.sh | sh -

$ # 使用 kubectl 检索多集群资源，当前 Cluster API 未创建集群，所以返回空
$ kubectl --cluster clusterpedia api-resources
```

## 使用 Cluster API 创建集群
使用示例环境的 Docker Provider 来创建集群时，需要添加 `--flavor development`
```bash
$ clusterctl generate cluster capi-quickstart --flavor development \
  --kubernetes-version v1.24.2 \
  --control-plane-machine-count=1 \
  --worker-machine-count=1 \
  > capi-quickstart.yaml
$ kubectl apply -f ./capi-quickstart.yaml
```

### 观察集群创建情况
```bash
$ kubectl get cluster
NAME              PHASE         AGE   VERSION
capi-quickstart   Provisioned   8s    v1.24.2

$ kubectl get kubeadmcontrolplane -w
NAME                    CLUSTER           INITIALIZED   API SERVER AVAILABLE   REPLICAS   READY   UPDATED   UNAVAILABLE   AGE   VERSION
capi-quickstart-2xcsz   capi-quickstart   true                                 1                  1         1             86s   v1.24.2
```

**当 kubeadmcontrolplane 的 Initialized 为 True 后**，clusterpedia 会自动同步该集群内的资源，可以使用 `kubectl --cluster clusterpedia get po -A` 来查看资源
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
自动创建的 pediacluster 默认的同步资源在 cluster-api [clusterimportpolicy 中设置](https://clusterpedia.io/docs/concepts/cluster-import-policy/#pediacluster-template)，

用户也可以手动修改 pediacluster 中同步的配置, [Synchronize Cluster Resources](https://clusterpedia.io/docs/usage/sync-resources/)

在 Cluster API 中删除集群时，Clusterpedia 也同步删除 PeidaCluster，不会继续同步该集群

## 对多个集群的资源检索
使用上述步骤创建多个集群
```bash
$ kubectl get cluster
NAME                PHASE         AGE    VERSION
capi-quickstart     Provisioned   10m    v1.24.2
capi-quickstart-2   Provisioned   118s   v1.24.2

$ kubectl get kubeadmcontrolplane
NAME                      CLUSTER             INITIALIZED   API SERVER AVAILABLE   REPLICAS   READY   UPDATED   UNAVAILABLE   AGE   VERSION
capi-quickstart-2-ctm9k   capi-quickstart-2   true                                 1                  1         1             10m   v1.24.2
capi-quickstart-2xcsz     capi-quickstart     true                                 1                  1         1             19m   v1.24.2

$ # pediacluster 会根据 cluster 资源自动创建
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

**clusterpedia 提供了两种资源检索方式**
* [兼容 Kubernetes OpenAPI 的资源检索](https://clusterpedia.io/zh-cn/docs/usage/access-clusterpedia/#%E8%AE%BF%E9%97%AE-clusterpedia-%E8%B5%84%E6%BA%90)
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

* [Collection Resource](https://clusterpedia.io/zh-cn/docs/concepts/collection-resource/)
```bash
$ kubectl get collectionresources
NAME            RESOURCES
any             *
workloads       apps.deployments,apps.daemonsets,apps.statefulsets
kuberesources   .*,admission.k8s.io.*,admissionregistration.k8s.io.*,apiextensions.k8s.io.*,apps.*,authentication.k8s.io.*,authorization.k8s.io.*,autoscaling.*,batch.*,certificates.k8s.io.*,coordination.k8s.io.*,discovery.k8s.io.*,events.k8s.io.*,extensions.*,flowcontrol.apiserver.k8s.io.*,imagepolicy.k8s.io.*,internal.apiserver.k8s.io.*,networking.k8s.io.*,node.k8s.io.*,policy.*,rbac.authorization.k8s.io.*,scheduling.k8s.io.*,storage.k8s.io.*

$ kubectl get collectionresources workloads
```

### 检索条件
* [元信息过滤(资源名称，命名空间，集群，创建时间区间)](https://clusterpedia.io/docs/usage/search/#search-by-metadata)
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

* [模糊搜索](https://clusterpedia.io/docs/usage/search/multi-cluster/#fuzzy-search)
* [增强的 Field Selector](https://clusterpedia.io/docs/usage/search/multi-cluster/#field-selector)
* [根据父辈或者祖辈 Owner 检索](https://clusterpedia.io/docs/usage/search/multi-cluster/#search-by-parent-or-ancestor-owner)
* [分页和排序](https://clusterpedia.io/docs/usage/search/multi-cluster/#paging-and-sorting)
* [自定义条件搜索](https://clusterpedia.io/docs/usage/search/#advanced-searchcustom-conditional-search)
