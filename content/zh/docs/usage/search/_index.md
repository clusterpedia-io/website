---
title: 资源检索
---

Clusterpedia 支持对 [多个集群内资源](searching-multi-cluster)，[指定集群的资源](searching-specified-cluster) 以及[聚合资源](searching-collection-resource) 的复杂检索，

并且这些复杂检索的条件可以通过两种方式传递给 `Clusterpedia APIServer`：
* `URL Query`：直接将查询条件作为 Query 来传递
* `Search Labels`：为了**兼容 Kubernetes OpenAPI**，可以将查询条件设置在 Label Selector。

**`Search Labels` 和 `URL Query` 都支持与 Label Selector 相同的操作符：**
* `exist`，`not exist`
* `=`，`==`，`!=`
* `in`，`not in`

除了条件检索，Clusterpedia 还增强了 [Field Selector](#字段过滤)
，满足我们通过 `metadata.annotation` 或者 `status.*` 等字段的过滤需求。
## 元信息检索
> 支持和 Label Selector 相同的操作符：`exist`，`not exist`，`==`，`=`，`!=`，`in`，`not in`。

|作用| search label key|url query|
| -- | --------------- | ------- |
|过滤集群名称|search.clusterpedia.io/clusters|clusters|
|过滤命名空间|search.clusterpedia.io/namespaces|namespaces|
|过滤资源名称|search.clusterpedia.io/names|names|

## Owner 检索
> 只支持操作符：`==`，`=`。

|作用| search label key|url query|
| -- | --------------- | ------- |
|指定 Owner UID|internalstorage.clusterpedia.io/owner-uid|ownerID|
|指定 Owner Key|internalstorage.clusterpedia.io/owner-key|ownerKey|
|指定 Owner 辈分|internalstorage.clusterpedia.io/owner-seniority|ownerSeniority|

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

**`exist`，`not exist`，`==`，`=`，`!=`，`in`，`not in`。**

无论是 URL 还是 kubectl 上的命令参数都原生 Field Selector 一致
|作用|kubectl|url query|
| -- | --------------- | ------- |
|字段过滤|`kubectl --field-selector`|fieldSelector|

详细可以查看：
* [使用字段过滤来检索资源](./searching-multi-cluster#字段过滤)
* [support field selector](https://github.com/clusterpedia-io/clusterpedia/pull/36) 
* [issue: support list field filtering](https://github.com/clusterpedia-io/clusterpedia/issues/48)
