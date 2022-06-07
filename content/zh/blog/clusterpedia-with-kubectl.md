---
title: Clusterpedia 加持 kubectl，检索多集群资源
date: 2021-12-03
---

在多集群时代，我们可以通过 cluster-api 来批量创建管理集群，使用 Karmada/Clusternet 来分发部署应用。

不过我们貌似还是缺少了什么功能，我们要如何去统一的查看多个集群中的资源呢？

对于单个集群的资源，我们可以使用 kubectl 来查看搜索资源，但是在想要检索多集群的资源时，貌似没有什么趁手的产品可以使用。

不过从今天开始，这个问题不会再困扰你，因为**在 Clusterpedia 的加持下，你手上的 kubectl 已经可以用来检索多集群资源啦。**

例如，使用 kubectl 来获取多个集群下 kube-system 命名空间内的 deployments。
```bash
$ kubectl get deployments -n kube-system
CLUSTER     NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
cluster-1   calico-kube-controllers      1/1     1            1           63d
cluster-1   coredns                      2/2     2            2           63d
cluster-2   calico-kube-controllers      1/1     1            1           109d
cluster-2   coredns-coredns              2/2     2            2           109d
cluster-2   dce-chart-manager            1/1     1            1           109d
cluster-2   dce-clair                    1/1     1            1           109d
```

## Clusterpedia 介绍
Clusterpedia，名字借鉴自 Wikipedia，同样也展现了 Clusterpedia 的核心理念 —— 多集群的百科全书。

通过聚合多集群资源，在兼容 Kubernetes OpenAPI 的基础上额外提供了更加强大的检索功能，让用户更快更方便的在多集群中获取到想要的任何资源。

当然 Clusterpedia 的能力并不仅仅只是检索查看，未来还会支持对资源的简单控制，就像 wiki 同样支持编辑词条一样。

### 架构设计
<div align="center"><img src="https://github.com/clusterpedia-io/clusterpedia/blob/main/docs/images/arch.png?raw=true" style="width:900px;" /></div>
Clusterpedia 在架构上分为四个部分：

* `Clusterpedia APIServer`：以 Aggregated API 的方式注册到 Kube APIServer，通过统一的入口来提供服务。
* `ClusterSynchro Manager`：管理用于同步集群资源的 Cluster Synchro。
* `Storage Layer (存储层)`：用来连接操作具体的存储组件，然后通过存储层接口注册到 Clusterpedia APIServer 和 ClusterSynchro Manager 中。
* `存储组件`：具体的存储设施，例如 mysql， postgres，redis 或者其他图数据库。
另外，Clusterpedia 会使用 **PediaCluster** 这个自定义资源来实现集群认证和资源收集配置

Clusterpedia 还提供了可以接入 mysql 和 postgres 的默认存储层。

Clusterpedia 并不关心用户所使用的具体存储设置是什么，用户可以根据自己的需求来选择或者实现存储层，然后将存储层以插件的形式注册到 Clusterpedia 中来使用。

### 特性和功能
* 支持复杂的检索条件，过滤条件，排序，分页等等
* 支持查询资源时请求附带关系资源
* 统一主集群和多集群资源检索入口
* 兼容 kubernetes OpenAPI, 可以直接使用 kubectl 进行多集群检索, 而无需第三方插件或者工具
* 兼容收集不同版本的集群资源，不受主集群版本约束，
* 资源收集高性能，低内存
* 根据集群当前的健康状态，自动启停资源收集
* 插件化存储层，用户可以根据自己需求使用其他存储组件来自定义存储层
* 高可用

## 部署
关于部署的详细流程，可以查看 [安装 Clusterpedia](/zh-cn/docs/installation)，这里着重介绍了如何使用 clusterpedia。

## 集群资源收集
clusterpedia 部署完成后，我们可以通过 kubectl 来操作 **PediaCluster** 资源。
```bash
$ kubectl get pediaclusters
```
在 examples 目录下，可以看到 PediaCluster 的示例
```yaml
apiVersion: clusters.clusterpedia.io/v1alpha1
kind: PediaCluster
metadata:
  name: cluster-example
spec:
  apiserverURL: "https://172.30.43.41:6443"
  caData: ""
  tokenData: ""
  certData: ""
  keyData: ""
  resources:
  - group: apps
    resources:
     - deployments
  - group: ""
    resources:
     - pods
```

**PediaCluster** 在配置上可以分成两部分
* 集群认证
* 指定资源收集 *.spec.resources*

### 集群认证
caData , tokenData , certData , keyData 字段用于集群的验证。

当前暂时不支持从 ConfigMap 或者 Secret 中获取验证相关的信息，不过已经在 Roadmap 中了。

**在设置验证字段时，注意要使用 base64 后的字符串**

在 examples 目录下提供了生成用于访问子集群的 rbac yaml *clusterpedia_synchro_rbac.yaml*，来方便的获取子集群的权限 token。

在子集群中部署该 yaml，然后获取对应的 token 和 ca 证书。
```bash
$ # 当前 kubectl 连接到子集群中
$ kubectl apply -f examples/clusterpedia_synchro_rbac.yaml
clusterrole.rbac.authorization.k8s.io/clusterpedia-synchro created
serviceaccount/clusterpedia-synchro created
clusterrolebinding.rbac.authorization.k8s.io/clusterpedia-synchro created
$ SYNCHRO_TOKEN=$(kubectl get secret $(kubectl get serviceaccount clusterpedia-synchro -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}')
$ SYNCHRO_CA=$(kubectl get secret $(kubectl get serviceaccount clusterpedia-synchro -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.ca\.crt}')
```
复制 ./examples/pediacluster.yaml, 并修改 .spec.apiserverURL 和 .metadata.name 字段，并且将 $SYNCHRO_TOKEN 和 $SYNCHRO_CA 填写到 tokenData 和 caData 中。

使用 kubectl apply 创建。
```bash
$ kubectl apply -f cluster-1.yaml
pediacluster.clusters.clusterpedia.io/cluster-1 created
```
为了方便后续使用，建议再创建一个 cluster-2

### 集群收集
可以通过设置 *spec.resources* 字段的 group 和 group 下的 resources 来进行指定收集的资源。

在 status 中我们也可以看到资源的收集状态。
```yaml
status:
  conditions:
  - lastTransitionTime: "2021-12-02T04:00:45Z"
    message: ""
    reason: Healthy
    status: "True"
    type: Ready
  resources:
  - group: ""
    resources:
    - kind: Pod
      namespaced: true
      resource: pods
      syncConditions:
      - lastTransitionTime: "2021-12-02T04:00:45Z"
        status: Syncing
        storageVersion: v1
        version: v1
  - group: apps
    resources:
    - kind: Deployment
      namespaced: true
      resource: deployments
      syncConditions:
      - lastTransitionTime: "2021-12-02T04:00:45Z"
        status: Syncing
        storageVersion: v1
        version: v1
  version: v1.22.2
```

## 资源检索
配置好我们需要收集的资源后，我们就可以进行重头戏了 —— 集群检索

clusterpedia 支持两种资源检索:

* 兼容 Kubernetes OpenAPI 的资源检索
* 集合资源 (Collection Resource) 的检索
```bash
$ kubectl api-resources | grep pedia.clusterpedia.io
collectionresources     pedia.clusterpedia.io/v1alpha1  false   CollectionResource
resources               pedia.clusterpedia.io/v1alpha1  false   Resources
```

为了方便我们更好的使用 kubectl 来进行检索，我们可以先通过 make gen-clusterconfig 来为子集群创建用于检索的 '快捷方式'。
```bash
$ make gen-clusterconfigs
./hack/gen-clusterconfigs.sh
Current Context: kubernetes-admin@kubernetes
Current Cluster: kubernetes
        Server: https://10.9.11.11:6443
        TLS Server Name:
        Insecure Skip TLS Verify:
        Certificate Authority:
        Certificate Authority Data: ***

Cluster "clusterpedia" set.
Cluster "cluster-1" set.
```
使用 kubectl config get-clusters 可以查看当前支持的集群。

其中 clusterpedia 是一个特殊的 cluster，用于多集群检索，以 kubectl --cluster clusterpedia 的方式来检索多个集群的资源。

### 多集群资源检索
我们先看一下我们都收集了哪些资源，只有被收集的资源才可以进行检索。
```bash
$ kubectl --cluster clusterpedia api-resources
NAME          SHORTNAMES   APIVERSION   NAMESPACED   KIND
pods          po           v1           true         Pod
deployments   deploy       apps/v1      true         Deployment
```
可以看到当前收集并支持 pods 和 deployments.apps 两种资源

**查看所有集群的 kube-system 命名空间下的 deployments**
```bash
$ kubectl --cluster clusterpedia get deployments -n kube-system
CLUSTER     NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
cluster-1   calico-kube-controllers      1/1     1            1           63d
cluster-1   coredns                      2/2     2            2           63d
cluster-2   calico-kube-controllers      1/1     1            1           109d
cluster-2   coredns-coredns              2/2     2            2           109d
cluster-2   dce-chart-manager            1/1     1            1           109d
cluster-2   dce-clair                    1/1     1            1           109d
```

**查看所有集群的 kube-system, default 命名空间下的 deployments**
```bash
$ kubectl --cluster clusterpedia get deployments -A -l "search.clusterpedia.io/namespaces in (kube-system, default)"
```

**查看 cluster-1, cluster-2 两个集群下的 kube-system, default 命名空间下中的 deployments**
```bash
$ kubectl --cluster clusterpedia get deployments -A -l "search.clusterpedia.io/clusters in (cluster-1, cluster-2),\
        search.clusterpedia.io/namespaces in (kube-system,default)"
NAMESPACE     CLUSTER     NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   cluster-1   calico-kube-controllers      1/1     1            1           63d
kube-system   cluster-1   coredns                      2/2     2            2           63d
default       cluster-1   dao-2048-2048                1/1     1            1           20d
default       cluster-1   hello-world-server           1/1     1            1           26d
default       cluster-1   my-nginx                     1/1     1            1           39d
default       cluster-1   phpldapadmin                 1/1     1            1           40d
kube-system   cluster-2   calico-kube-controllers      1/1     1            1           109d
kube-system   cluster-2   coredns-coredns              2/2     2            2           109d
kube-system   cluster-2   dce-chart-manager            1/1     1            1           109d
kube-system   cluster-2   dce-clair                    1/1     1            1           109d
```

*显示数据有删减，略多*

**查看 cluster-1, cluster-2 两个集群下的 kube-system, default 命名空间下中的 deployments，并根据资源的名字排序**
```bash
$ kubectl --cluster clusterpedia get deployments -A -l "search.clusterpedia.io/clusters in (cluster-1, cluster-2),\
        search.clusterpedia.io/namespaces in (kube-system,default),\
        search.clusterpedia.io/orderby=name"
kube-system   cluster-1   calico-kube-controllers      1/1     1            1           63d
kube-system   cluster-2   calico-kube-controllers      1/1     1            1           109d
kube-system   cluster-1   coredns                      2/2     2            2           63d
kube-system   cluster-2   coredns-coredns              2/2     2            2           109d
default       cluster-1   dao-2048-2048                1/1     1            1           20d
kube-system   cluster-2   dce-chart-manager            1/1     1            1           109d
kube-system   cluster-2   dce-clair                    1/1     1            1           109d
kube-system   cluster-2   dce-registry                 1/1     1            1           109d
kube-system   cluster-2   dce-uds-storage-server       1/1     1            1           109d
default       cluster-1   dd-airflow-scheduler         0/1     1            0           53d
default       cluster-1   dd-airflow-web               0/1     1            0           53d
kube-system   cluster-2   metrics-server               1/1     1            1           109d
default       cluster-1   my-nginx                     1/1     1            1           39d
default       cluster-1   nginx-dev                    1/1     1            1           14d
default       cluster-1   openldap                     1/1     1            1           40d
default       cluster-1   phpldapadmin                 1/1     1            1           40d
```
*显示数据有删减，略多*

### 指定集群检索
**我们如果想要检索指定集群的资源的话，我们可以使用 --cluster 来指定具体的集群名称**
```bash
$ kubectl --cluster cluster-1 get deployments -A
NAMESPACE                     CLUSTER     NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
kubeapps-oidc                 cluster-1   apach2-apache                                 1/1     1            1           35d
kube-system                   cluster-1   calico-kube-controllers                       1/1     1            1           63d
cert-manager                  cluster-1   cert-manager                                  1/1     1            1           42d
cert-manager                  cluster-1   cert-manager-cainjector                       1/1     1            1           42d
cert-manager                  cluster-1   cert-manager-webhook                          1/1     1            1           42d
kube-system                   cluster-1   coredns                                       2/2     2            2           63d
default                       cluster-1   dao-2048-2048                                 1/1     1            1           20d
kubernetes-dashboard          cluster-1   dashboard-metrics-scraper                     1/1     1            1           54d
default                       cluster-1   dd-airflow-scheduler                          0/1     1            0           53d
default                       cluster-1   dd-airflow-web                                0/1     1            0           53d
```
*显示数据有删减，略多*

除了 http://search.clusterpedia.io/clusters 外其余的复杂查询的支持和多集群检索相同。

如果我们要获取一个资源的详情，那么也是需要指定集群才可以。
```bash
$ kubectl --cluster cluster-1 -n kube-system get deployments coredns
CLUSTER     NAME            READY   UP-TO-DATE   AVAILABLE   AGE
cluster-1   apach2-apache   1/1     1            1           35d
```
复杂检索clusterpedia 支持以下复杂检索：
* 指定一个或者多个**集群名称**
* 指定一个或者多个**命名空间**
* 指定一个或者多个**资源名称**
* 指定多个字段的**排序**
* **分页**功能，可以指定 size 和 offset
* **labels 过滤**

对于字段的排序，实际的效果是根据存储层来决定的，默认存储层支持根据 cluster , name , namespace , created_at , resource_version 进行正序或者倒序的排序。

### 检索条件的传递方式
上面实例中，演示了使用 kubectl 来进行检索，而这些复杂的检索条件通过 label 来传递的。实际上 clusterpedia 还支持直接通过 url query 的传递这些检索条件。

|功能|label key|url query|example|
|---|---|---|---|
|Specified resource name|search.clusterpedia.io/names|names|`?names=pod-1,pod-2`
|Specified namespace|search.clusterpedia.io/namespaces|namespaces|`?namespaces=kube-system,default`
|Specified cluster name|search.clusterpedia.io/clusters|clusters|`?clusters=cluster-1,cluster-2`
|Sort by specified fileds|search.clusterpedia.io/orderby|orderby|`?orderby=name desc,namespace`
|Specified size |search.clusterpedia.io/size|size|`?size=100`
|Specified offset |search.clsuterpedia.io/offset|offset|`?offset=10`

search label key 的操作符支持 ==, =, !=, in, not in 对于 size 这个条件，实际上 kubectl 可以通过 --chunk-size 来指定，而不需要通过 label key。

### 聚合资源(Collection Resource)
在 clusterpedia 还有对资源更加高级的聚合，使用 **Collection Resource** 可以一次性获取到一组不同类型的资源。

可以先查看一下当前 clusterpedia 支持哪些 **Collection Resource**。
```bash
$ kubectl get collectionresources
NAME        RESOURCES
workloads   deployments.apps,daemonsets.apps,statefulsets.apps
```
通过获取 workloads 便可获取到一组 deployment, daemonset, statefulset 聚合在一起的资源 而且 Collection Resource 同样支持所有的复杂查询。

`kubectl get collectionresources workloads` 会默认获取所有集群下所有命名空间的相应资源。
```bash
$ kubectl get collectionresources workloads
CLUSTER     GROUP   VERSION   KIND         NAMESPACE                     NAME                                          AGE
cluster-1   apps    v1        DaemonSet    kube-system                   vsphere-cloud-controller-manager              63d
cluster-2   apps    v1        Deployment   kube-system                   calico-kube-controllers                       109d
cluster-2   apps    v1        Deployment   kube-system                   coredns-coredns                               109d
cluster-2   apps    v1        Deployment   dce-acm-agent                 dce-acm-agent                                 84d
```
*在 cluster-1 中增加收集 Daemonset, 输出有删减，太多*

由于 kubectl 的限制所以无法在 kubectl 来使用复杂查询，只能通过 url query 的方式来查询。

### 自定义 Collection Resource

**Collection Resource** 支持哪些资源是由存储层来提供，而默认存储层未来会支持自定义组合 **Collection Resource**。

## 新特性议题
### 对资源进行更复杂的操作
clusterpedia 不仅仅只是用来做资源检索，和 wiki 一样，它也应该具有对资源简单的控制能力，例如 watch, create, delete, update 等操作。

对于写操作，实际会采用双写 + 响应 warning 的方式来完成。

感兴趣的话可以在 issue 中一起讨论。

### 集群的自动发现与收集
clusterpedia 中用来表示集群的资源叫做 PediaCluster, 而不是简单的 Cluster，最主要的原因便是 clusterpedia 设计初衷便是让 clusterpedia 可以建立在已有的多集群管理平台之上。

为了遵循初衷，第一个问题便是不能和已有的多集群平台中的资源冲突， Cluster 便是一个最通用的代表集群的资源名称。

另外为了更好的去接入到已有的多集群平台上，让已经接入的集群可以自动的完成资源收集，我们需要另外的一个集群发现机制。这个发现机制需要解决以下问题：

* 能够获取到访问集群的认证信息
* 可以配置触发 PediaCluster 生命周期的 Condition 条件
* 设置默认的资源收集策略，以及名称前缀等

这个功能会在 Q1 或者 Q2 中开始详细讨论实现。

##  当前进展
clusterpedia 当前处于比较早期的阶段 (v0.0.9-alpha)，核心功能刚刚完成，还有很多可以优化的地方，对于这些优化点也都提了对应的 issues，欢迎大家一起讨论

这里简单说一些进入 v0.1.0 版本前的优化点:

* 从具有 Server-Side Apply 特性的集群中收集到的资源会带有很臃肿的 managedFields 字段， clustersynchro manager 模块会增加相应 feature gate，来允许用户在收集时裁减掉这个字段
* 同样的臃肿字段 annotations 中的 http://kubectl.kubernetes.io/last-applied-configuration，也要允许裁剪这个字段
* 在指定集群获取资源时，如果集群处于异常状态时，应该在响应中添加 warning 来提醒用户
* 对 PediaCluster 的状态信息有更准确的更新
* 弱网环境下，资源收集的优化

更多的优化项，大家可以在 issue 中提出新的想法。

## Roadmap
当前只是暂定的 Roadmap，具体的排期还要看社区的需求程度

### 2021 Q4
在 2021 的 Q4 阶段会完成上述的优化项，并且完成对自定义资源的收集

* 详细化资源收集状态
* 自定义资源的收集

### 2022 Q1
* 支持插件化存储层
* 实现集群的自动发现和收集

### 2022 Q3
* 支持对集群资源更多的控制，例如 watch/create/update/delete 等操作
* 默认存储层支持自定义 Collection Resource
* 支持请求附带关系资源

## 使用注意
### 多集群网络连通
clusterpedia 实际并不会解决多集群环境下的网络连通问题，用户可以使用tower等工具来连接访问子集群，也可以借助 submariner 或者 skupper 来解决跨集群网络问题。
