---
title: "返回剩余资源数量"
weight: 1
---
我们在查询时，可以通过 search label 或者 url query，要求在响应中携带剩余的资源数量。
|search label| url query|
|------------|----------|
|search.clusterpedia.io/with-continue|withContinue|
> 详细使用可以参考 [响应携带剩余资源数量信息](../search/multi-cluster/#响应携带剩余资源数量信息)

可以通过 Feature Gates —— `RemainingItemCount` 来设置默认返回剩余的资源数量，这样用户就不需要在每次请求时使用 search label 或者 url query 来显示要求了。

在默认返回剩余资源数量时，用于依然可以通过 search label 或者 url quera 来不返回剩余的资源数量
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?withRemainingCount=false&limit=1" | jq
```

**专属于 `clusterpedia apiserver` 的 Feature Gate**
|作用|feature gate|默认值|
|----|------------|------|
|设置是否默认返回剩余的资源数量|`RemainingItemCount`|false|

由于该功能可能会对存储层的行为或者性能有影响，所以默认关闭
> 对于默认存储层，返回剩余资源数量会导致额外的 COUNT 查询
