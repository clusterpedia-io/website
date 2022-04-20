---
title: 多集群检索
weight: 10
---
多集群资源检索可以满足我们**根据查询条件一次过滤多个集群内的资源，并提供对这些资源的分页排序的能力**

在使用 kubectl 操作时，可以查看一下当前可以检索哪些资源
```bash
kubectl --cluster clusterpedia api-resources
```
```
# 输出：
NAME          SHORTNAMES   APIVERSION           NAMESPACED   KIND
configmaps    cm           v1                   true         ConfigMap
namespaces    ns           v1                   false        Namespace
nodes         no           v1                   false        Node
pods          po           v1                   true         Pod
secrets                    v1                   true         Secret
daemonsets    ds           apps/v1              true         DaemonSet
deployments   deploy       apps/v1              true         Deployment
replicasets   rs           apps/v1              true         ReplicaSet
issuers                    cert-manager.io/v1   true         Issuer
```
Clusterpedia 根据所有集群同步的资源来提供多集群的资源检索，可以查看 [同步集群资源](../../sync-resources) 来更新需要同步的资源

## 基本功能
### 指定集群
多集群检索时，会默认检索所有的集群，我们也可以指定单个或者一组集群

{{< tabs >}}

{{% tab name="kubectl" %}}
使用 [Search Label](../#元信息检索) `search.clusterpedia.io/clusters` 来指定一组集群
```bash
kubectl --cluster clusterpedia get deployments -l "search.clusterpedia.io/clusters in (cluster-1,cluster-2)"
```
```
# 输出：
NAMESPACE     CLUSTER     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   cluster-1   coredns                   2/2     2            2           68d
kube-system   cluster-2   coredns                   2/2     2            2           64d
```

对于指定单个集群的检索，同样可以使用 Search Label 来设置，也可以查看 [指定集群检索](../specified-cluster) 来使用 URL Path 的方式指定集群
```bash
# 指定单个集群
kubectl --cluster clusterpedia get deployments -l "search.clusterpedia.io/clusters=cluster-1"

# 指定集群也可以使用 --cluster <cluster name> 来指定
kubectl --cluster cluster-1 get deployments
```
{{% /tab %}}

{{% tab name="URL" %}}
使用 URL 时，使用 `clusters` 作为 [URL Query](../#元信息检索) 来传递
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?clusters=cluster-1"
```

如果指定单个集群，也可以将 cluster name 放到 URL 路径中
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/clusters/cluster-1/apis/apps/v1/deployments"
```
了解更多[指定集群检索](../specified-cluster)
{{< /tab >}}

{{< /tabs >}}

### 指定命名空间
可以像查看原生 Kube 一样来**指定单个命名空间或者所有命名空间**

{{< tabs >}}

{{% tab name="kubectl" %}}
使用 `-n <namespace>` 来指定命名空间，默认在 *default* 命名空间
```bash
kubectl --cluster clusterpedia get deployments -n kube-system
```
```
# 输出：
CLUSTER     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
cluster-1   coredns                   2/2     2            2           68d
cluster-2   calico-kube-controllers   1/1     1            1           64d
cluster-2   coredns                   2/2     2            2           64d
```

使用 `-A` 或者 `--all-namespaces` 来查看所有集群的所有命名空间下的资源
```bash
kubectl --cluster clusterpedia get deployments -A
```
```
# 输出：
NAMESPACE     CLUSTER     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   cluster-1   coredns                   2/2     2            2           68d
kube-system   cluster-2   calico-kube-controllers   1/1     1            1           64d
kube-system   cluster-2   coredns                   2/2     2            2           64d
default       cluster-2   dd-airflow-scheduler      0/1     1            0           54d
default       cluster-2   dd-airflow-web            0/1     1            0           54d
```
{{% /tab %}}

{{% tab name="URL" %}}
获取资源的 **URL Path** 和原生 Kubernetes 一样 */apis/apps/v1/deployments*，

只是需要加上 Clusterpedia Resources 的路径前缀 */apis/clusterpedia.io/v1beta1/resources* 来表示当前是 Clusterpedia 请求。
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"

# 指定命名空间
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/namespaces/kube-system/deployments"
```
{{< /tab >}}

{{< /tabs >}}

---

除了指定单个命名空间，还可以**指定查看一组命名空间下的资源**
{{< tabs >}}

{{% tab name="kubectl" %}}
使用 [Search Label](../#元信息检索) `search.clusterpedia.io/namespaces` 来指定一组命名空间
> 一定要指定 -A 参数，避免 kubectl 在路径中设置 default namespace
```bash
kubectl --cluster clusterpedia get deployments -A -l "search.clusterpedia.io/namespaces in (kube-system, default)"
```
```
# 输出：
NAMESPACE     CLUSTER     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   cluster-1   coredns                   2/2     2            2           68d
kube-system   cluster-2   calico-kube-controllers   1/1     1            1           64d
kube-system   cluster-2   coredns                   2/2     2            2           64d
default       cluster-2   dd-airflow-scheduler      0/1     1            0           54d
default       cluster-2   dd-airflow-web            0/1     1            0           54d
```
{{% /tab %}}

{{% tab name="URL" %}}
使用 URL 时，就不需要使用 Label Selector 来传递参数了，直接使用 URL Query `namespaces` 即可
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?namespaces=kube-system,default"
```
{{< /tab >}}

{{< /tabs >}}

### 指定资源名称
用户可以通过一组资源名称来过滤资源

{{< tabs >}}

{{% tab name="kubectl" %}}
使用 Search Label `search.clusterpedia.io/names` 来指定一组资源名称
**注意：如果在所有命名空间下检索资源，需要指定 -A 参数，或者使用 -n 来指定命名空间**
```bash
kubectl --cluster clusterpedia get deployments -A -l "search.clusterpedia.io/names=coredns"
```
```
# 输出：
NAMESPACE     CLUSTER     NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   cluster-1   coredns                   2/2     2            2           68d
kube-system   cluster-2   coredns                   2/2     2            2           64d
```
{{% /tab %}}

{{% tab name="URL" %}}
使用 URL 时，使用 `names` 作为 URL Query 来传递，如果需要指定命名空间，那么就在路径中加上 namespace。
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?names=kube-coredns,dd-airflow-web"

# 在 default 命名空间下检索指定名字的资源
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/namespaces/default/deployments?names=kube-coredns,dd-airflow-web"
```

在多集群检索时，返回的数据实际是以类似 `DeploymentList` 的结构封装的数据。

如果我们想要获取到单个的 `Deployment` 那么就需要在 URL 路径中指定 cluster name，参考[获取单个资源](../specified-cluster#获取单个资源)
{{< /tab >}}

{{< /tabs >}}

### 创建时间的区间
创建时间的区间以左闭右开的方式来进行检索，**since <= creation time < before**

关于详细的时间区间参数可以查看 [创建时间区间检索](../#创建时间区间检索)

{{< tabs >}}

{{% tab name="kubectl" %}}
使用 Search Label `search.clusterpedia.io/since` 和 `search.clusterpedia.io/before` 来指定时间区间
```bash
kubectl --cluster clusterpedia get deployments -A -l "search.clusterpedia.io/since=2022-03-24, \
    search.clusterpedia.io/before=2022-04-10"
```
{{% /tab %}}

{{% tab name="URL" %}}
直接使用 URL 时，可以 Query `since` 和 `before` 来分别指定时间的区间
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?since=2022-03-24&before=2022-04-10"
```
{{< /tab >}}

{{< /tabs >}}

## 模糊搜索
当前支持根据资源名称进行模糊搜索，由于模糊搜索还需要继续讨论，所以暂时以试验性功能来提供

只支持 Search Label 的方式，不支持 URL Query
```bash
kubectl --cluster clusterpedia get deployments -A -l "internalstorage.clusterpedia.io/fuzzy-name=test"
```
过滤出名字中包含 *test* 字符串的 deployments。

可以使用 `in` 操作符来传递多个参数，这样可以过滤出名字中包含所有字符串的资源

## 字段过滤
原生 Kubernetes 当前只支持对 `metadata.name` 和 `metadata.namespace` 的字段过滤，而且操作符只支持 `=`，`!=`，`==`，能力非常有限。

**Clusterpedia 在兼容已有的 Field Selector 功能的基础上，提供了更加强大的功能，支持和 `Label Selector` 相同的操作符。**

Field Selector 的 key 当前支持三种格式：
* **使用 `.` 分隔字段**
```bash
kubectl --cluster clusterpedia get pods --field-selector="status.phase=Running"

# 也可以在首字符添加 `.`
kubectl --cluster clusterpedia get pods --field-selector=".status.phase notin (Running,Succeeded)"
```

* **字段名称使用 `''` 或者 `""` 来包裹**，可以用于带 `.` 之类的非法字符的字段
```bash
kubectl --cluster clusterpedia get deploy \
    --field-selector="metadata.annotations['test.io'] in (value1,value2),spec.replica=3"
```

* **使用 `[]` 来分隔字段**，`[]` 内字符串必须使用 `''` 或者 `""` 来包裹
```bash
kubectl --cluster clusterpedia get pods --field-selector="status['phase']!=Running"
```

### 列表字段支持
实际在字段过滤的设计时考虑到了对`列表元素`内字段过滤，不过由于使用场景是否真正有意义还需要更多的讨论 [issue: support list field filtering](https://github.com/clusterpedia-io/clusterpedia/issues/48)

示例：
```bash
kubectl get po --field-selector="spec.containers[].name!=container1"

kubectl get po --field-selector="spec.containers[].name == container1"

kubectl get po --field-selector="spec.containers[1].name in (container1,container2)"
```

## 根据父辈以及祖辈 Owner 查询
通过 Owner 检索是一个非常有用的检索功能，
并且 Clusterpedia 在 Owner 的基础上还支持**对 Owner 进行辈分提升来进行祖辈甚至更高辈分的检索**。

通过 Owner 检索，可以一次查询到 `Deployment` 下的所有 `Pods`，无需中间再查询 `ReplicaSet`。

**Owner 查询必须指定单个集群，可以使用 [Serach Label](../#owner-检索) 或者 [URL Query](../#owner-检索) 来指定，也可以在 URL Path 中指定集群名称**

关于**根据 Owner 检索**的具体使用方法，可以参考[指定集群内根据父辈或者祖辈 Owenr 进行检索](../specified-cluster#根据父辈或者祖辈-owner-进行检索)

## 分页与排序
分页和排序是资源检索必不可少的功能

### 根据多个字段进行排序
可以指定多个字段进行排序，而对排序字段的支持是由存储层来决定。

当前默认存储层支持对 `cluster`，`namespace`，`name`，`created_at`，`resource_version` 进行正序和倒序的排序，字段也支持随意的组合
{{< tabs >}}

{{% tab name="kubectl" %}}
使用多个字段进行正序排序
```bash
kubectl --cluster clusterpedia get pods -l \
    "search.clusterpedia.io/orderby in (cluster, name)"
```

由于 Label Selector 对 value 的限制，倒序时需要在字段结尾加上 `_desc`
```bash
kubectl --cluster clusterpedia get pods -l \
    "search.clusterpedia.io/orderby in (namespace_desc, cluster, name)"
```
{{% /tab %}}

{{% tab name="URL" %}}
使用 [URL Query](../#排序) 来指定排序字段
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?orderby=namespace,cluster"
```

指定倒序字段时，在字段后添加 desc，以空格分隔
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?orderby=namespace desc,cluster"
```
{{< /tab >}}

{{< /tabs >}}

### 分页
原生 Kubernetes 实际是支持分页的，[ListOptions](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#ListOptions) 中便已经存在用于分页查询的字段。

Clusterpedia 复用 `ListOptions.Limit` 和 `ListOptions.Continue` 字段作为分页的 `size` 和 `offset`。

{{< tabs >}}

{{% tab name="kubectl" %}}
kubectl 的 `--chunk-size` 实际通过设置 `limit` 来用于分片拉取。

原生的 Kubernetes APIServer 会在返回的响应中携带用于下一次拉取的 `continue`，
并根据 `--chunk-size` 和 `conintue` 进行下一次拉取，直到相应的数据中 Conintue 为空。

Clusterpedia 为了保证在 kubectl 中实现分页检索，默认并不会在响应中返回 `continue` 字段，这样避免了 kubectl 使用分片拉取全部数据
```bash
kubectl --cluster cluster-1 get pods --chunk-size 10
```
需要注意 kubectl 在不设置 `--chunk-size` 的情况下，`limit` 会被设置成默认值 `500`，
也就是说 `search.clusterpedia.io/size` 实际是无法生效的，只是用于和 `search.clusterpedia.io/offset` 形成对应关系
> URL Query 的优先级大于 Search Label

在 kubectl 中 continue 是没有 flag 可以设置的。所以还是要使用 [Search Label](../#分页) 来传递。
```bash
kubectl --cluster clusterpedia get pods --chunk-size 10 -l \
    "search.clusterpedia.io/offset=10"
```
{{% /tab %}}

{{% tab name="URL" %}}
对资源进行分页检索，只需要在 URL 中设置 `limit` 和 `continue` 即可
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?limit=10&continue=5"
```
{{< /tab >}}

{{< /tabs >}}

### 响应携带 Continue 信息
响应数据的 [ListMeta.Continue](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#ListMeta) 可以用于 [ListOptions.Continue](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#ListOptions) 中作为下一次拉取的 offset

分页功能中我们提到，为了避免 kubectl 进行对全量数据的分片拉取，Clusterepdia 不会在响应中携带 Continue 信息。

不过如果用户有需求那么可以要求响应中携带 Continue 信息
{{< tabs >}}

{{% tab name="URL" %}}
在使用 URL 访问 Clusterepdia 时，响应的 Continue 可以作为下一次请求的 offset
> 搭配分页功能使用

```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?withContinue=true&limit=1" | jq
```
```json
{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "continue": "1"
  },
  "items": [
    ...
  ]
}
```
{{% /tab %}}

{{% tab name="kubectl" %}}
**在 kubectl 设置 `search.clusterpedia.io/with-continue` 会导致以分片拉取的形式拉取全量资源。**
```bash
kubectl --cluster clusterpedia get deploy -l \
    "search.clusterpedia.io/with-continue=true"
```
{{< /tab >}}

{{< /tabs >}}

### 响应携带剩余资源数量信息
**在一些 UI 场景下，往往会需要获取到当前检索条件下的资源总量。**

Kubernetes List 响应的 [ListMeta](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#ListMeta) 中存在 `RemainingItemCount` 字段，

通过复用该字段，便可在兼容 Kubernetes OpenAPI 的基础下计算出资源总量：

**`offset + len(list.items) + list.metadata.remainingItemCount`**

> 在 offset 过大时，`remainingItemCount` 可能为负数，保证总是可以计算出资源总量

{{< tabs >}}

{{% tab name="URL" %}}
在 [URL Query](../#分页) 设置 `withRemainingCount` 即可要求响应携带剩余资源数量
> 搭配[分页](#分页)功能使用
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?withRemainingCount&limit=1" | jq
```
```json
{
  "kind": "DeploymentList",
  "apiVersion": "apps/v1",
  "metadata": {
    "remainingItemCount": 23
  },
  "items": [
    ...
  ]
}
```
{{% /tab %}}

{{% tab name="kubectl" %}}
需要以 URL 的方式使用该功能
{{< /tab >}}

{{< /tabs >}}

