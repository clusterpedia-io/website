---
title: 资源检索
---

Clusterpedia 支持对 [多个集群内资源](multi-cluster)，[指定集群的资源](specified-cluster) 以及[聚合资源](collection-resource) 的复杂检索，

并且这些复杂检索的条件可以通过两种方式传递给 `Clusterpedia APIServer`：
* `URL Query`：直接将查询条件作为 Query 来传递
* `Search Labels`：为了**兼容 Kubernetes OpenAPI**，可以将查询条件设置在 Label Selector。

**`Search Labels` 和 `URL Query` 都支持与 Label Selector 相同的操作符：**
* `exist`，`not exist`
* `=`，`==`，`!=`
* `in`，`notin`

除了条件检索，Clusterpedia 还增强了 [Field Selector](#字段过滤)
，满足我们通过 `metadata.annotation` 或者 `status.*` 等字段的过滤需求。
## 元信息检索
> 支持的操作符：`==`，`=`，`in`。

|作用| search label key|url query|
| -- | --------------- | ------- |
|过滤集群名称|search.clusterpedia.io/clusters|clusters|
|过滤命名空间|search.clusterpedia.io/namespaces|namespaces|
|过滤资源名称|search.clusterpedia.io/names|names|

> 暂时不支持例如 `!=`，`notin` 操作符，如果有这些需求或者场景，可以在 issue 中讨论

## 模糊搜索
> 支持的操作符：`==`，`=`，`in`。

该功能暂时为试验性功能，暂时只提供 search label
|作用| search label key|url query|
| -- | --------------- | ------- |
|模糊搜索资源名称|internalstorage.clusterpedia.io/fuzzy-name|-|

## 创建时间区间检索
> 支持的操作符：`==`，`=`。

基于资源的创建时间区间进行检索，采用的是左闭右开的区间

|作用| search label key|url query|
| -- | --------------- | ------- |
|指定 Since|search.clusterpedia.io/since|since|
|指定 Before|search.clusterpedia.io/before|before|

创建时间的格式有四种：
1. `Unix 时间戳格式` 为了方便使用会根据时间戳的长度来区分单位为 s 还是 ms。
10 位时间戳单位为秒，13 位时间戳单位为毫秒。
2. `RFC3339` *2006-01-02T15:04:05Z* or *2006-01-02T15:04:05+08:00*
3. `UTC Date` *2006-01-02*
4. `UTC Datetime` *2006-01-02 15:04:05*

由于 kube label selector 的限制，search label 只支持 `Unix 时间戳`，`UTC Date`.

使用 url query 的方式可以所有的格式

## Owner 检索
> 只支持操作符：`==`，`=`。

|作用| search label key|url query|
| -- | --------------- | ------- |
|指定 Owner UID|search.clusterpedia.io/owner-uid|ownerUID|
|指定 Owner Name|search.clusterpedia.io/owner-name|ownerName|
|指定 Owner Group Resource|search.clusterpedia.io/owner-gr|ownerGR|
|指定 Owner 辈分|internalstorage.clusterpedia.io/owner-seniority|ownerSeniority|

需要注意指定 `Owner UID` 时，`Owner Name` 和 `Owner Group Resource` 会被忽略

Owner Group 的格式为 `resource.group`，例如 *deployments.apps* 或者 *nodes*

## 排序
> 只支持操作符：`=`，`==`，`in`。

|作用| search label key|url query|
| -- | --------------- | ------- |
|多字段排序|search.clusterpedia.io/orderby|orderby|

## 分页
> 只支持操作符 `=`，`==`。

|作用| search label key|url query|
| -- | --------------- | ------- |
|设置分页 size|search.clusterpedia.io/size|limit|
|设置分页 offset|search.clusterpedia.io/offset|continue|
|要求响应携带 Continue|search.clusterpedia.io/with-continue|withContinue
|要求响应携带资源剩余数量|search.clusterpedia.io/with-remaining-count|withRemainingCount

> 在使用 kubectl 操作时，分页 size 只能通过 `kubectl --chunk-size` 来设置，因为 kubectl 会将 limit 默认设置为 500。

## Label 过滤
无论使用 kubectl 还是 URL，所有 Key 中不包含 *clusterpedia.io* 的 Label Selector 都会作为 Label Selector 来过滤资源。

所有行为和原生 Kubernetes 一致。
|作用|kubectl|url query|
| -- | --------------- | ------- |
|Label 过滤|`kubectl -l` or `kubectl --label-selector`|labelSelector|

## 字段过滤
**Clusterpedia 的 Field Selector 在操作符上与 Label Selector 保持一致，同样支持：** 

**`exist`，`not exist`，`==`，`=`，`!=`，`in`，`notin`。**

无论是 URL 还是 kubectl 上的命令参数都原生 Field Selector 一致
|作用|kubectl|url query|
| -- | --------------- | ------- |
|字段过滤|`kubectl --field-selector`|`fieldSelector`|

详细可以查看：
* [使用字段过滤来检索资源](./multi-cluster#字段过滤)
* [support field selector](https://github.com/clusterpedia-io/clusterpedia/pull/36) 
* [issue: support list field filtering](https://github.com/clusterpedia-io/clusterpedia/issues/48)

## 高级检索(自定义条件检索)
自定义检索为 `默认存储层` 提供的功能，目的是为了满足用户更加灵活多变的检索需求
|作用| search label key |url query|
| -- | ---------------- | ------- |
|自定义检索语句|-|whereSQL|

自定义检索并不支持通过 search lable，只能使用 url query 来传递自定义搜素的语句。

另外该功能暂时还是处于 alpha 阶段，需要用户在 `clusterpedia apiserver` 中开启相应的 Feature Gate，详细可以参考 [自定义条件检索](../../features/raw-sql-query)
