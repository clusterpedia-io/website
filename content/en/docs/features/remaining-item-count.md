---
title: "Return RemainingItemCount"
weight: 1
---
We can require the number of remaining resources to be included in the response by search label or url query when querying.
|search label| url query|
|------------|----------|
|search.clusterpedia.io/with-remaining-count|withRemainingCount|
> Detailed use can be referred to [Response With Remaining Count](../../usage/search/multi-cluster/#response-with-remaining-count)

You can set the number of remaining resources to be returned by default via Feature Gates -- `RemainingItemCount`,
so that the user does not need to use a search label or url query to display the request at each request.

When the remaining item count is returned by default,
you can still request that the remaining item count not be returned via search label or url query.
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?withRemainingCount=false&limit=1" | jq
```

**Feature Gate dedicated to `clusterpedia apiserver`**
|desc|feature gate|default|
|----|------------|------|
|Set whether to return the number of remaining resources by default|`RemainingItemCount`|false|

This feature is turned off by default because it may have an impact on the behavior or performance of the storage layer.
> For the default storage tier, returning the number of remaining resources results in an additional COUNT query
