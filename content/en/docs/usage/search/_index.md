---
title: Search for resources
---

Clusterpedia supports complex search for [multi-cluster resources](multi-cluster), [specific cluster resoruces](specified-cluster), and [collection resources](collection-resource). And these complex search conditions can be passed to `Clusterpedia APIServer` in two ways:
* `URL Query`: directly pass query conditions as Query
* `Search Labels`: to keep **compatible with Kubernetes OpenAPI**, the search conditions can be set via Label Selector

**Both `Search Labels` and `URL Query` support same operators as Label Selector:**
* `exist`, `not exist`
* `=`, `==`, `!=`
* `in`, `not in`

In addition to conditional retrieval, Clusterpedia also enhances [`Field Selector`](#field filtering) to meet the filtering requirements by fields such as `metadata.annotation` or `status.*`.
## Search by metadata
> It supports same operators as Label Selector: `exist`, `not exist`, `==`, `=`, `!=`, `in`, `not in`.

|Role| search label key|url query|
| -- | --------------- | ------- |
|Filter cluster names|search.clusterpedia.io/clusters|clusters|
|Filter namespaces|search.clusterpedia.io/namespaces|namespaces|
|Filter resource names|search.clusterpedia.io/names|names|

## Search by Owner
> It only supports operators: `==`, `=`.

|Role| search label key|url query|
| -- | --------------- | ------- |
|Specified Owner UID|internalstorage.clusterpedia.io/owner-uid|ownerID|
|Specified Owner Key|internalstorage.clusterpedia.io/owner-key|ownerKey|
|Specified Owner seniority|internalstorage.clusterpedia.io/owner-seniority|ownerSeniority|

## Sort
> It only supports operators: `=`, `==`, `in`.

|Role| search label key|url query|
| -- | --------------- | ------- |
|Sort by fields|search.clusterpedia.io/orderby|orderby|

## Page
> It only supports operators: `=`, `==`.

|Role| search label key|url query|
| -- | --------------- | ------- |
|Set page size|search.clusterpedia.io/size|limit|
|Set page offset|search.clusterpedia.io/offset|continue|
|Response required with Continue|search.clusterpedia.io/with-continue|withContinue
|Response required with remaining count|search.clusterpedia.io/with-remaining-count|withRemainingCount

> When you perform operations with kubectl, the page size can only be set via `kubectl --chunk-size`, because kubectl will set the default limit to 500.

## Filter by Label
Regardless of kubectl or URL, all Label Selectors that do not contain *clusterpedia.io* in the Key will be used as Label Selectors to filter resources.  

All behaviors are consistent with those provided by Kubernetes.
|Role|kubectl|url query|
| -- | --------------- | ------- |
|Filter by Label|`kubectl -l` or `kubectl --label-selector`|labelSelector|

## Filter by fields
**Field Selector is consistent with Label Selector in terms of operators, and Clusterpedia also supports:** 

**`exist`, `not exist`, `==`, `=`, `!=`, `in`, `not in`.**

All command parameters for URL and kubectl are same as those for Field Selector.
|Role|kubectl|url query|
| -- | --------------- | ------- |
|Filter by fields|`kubectl --field-selector`|fieldSelector|

For details refer to:
* [search for resources by filtering fields](./multi-cluster#filtering fields)
* [support field selector](https://github.com/clusterpedia-io/clusterpedia/pull/36) 
* [issue: support list field filtering](https://github.com/clusterpedia-io/clusterpedia/issues/48)
