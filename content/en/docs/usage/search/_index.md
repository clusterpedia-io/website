---
title: Search
---

Clusterpedia supports complex search for [multi-cluster resources](multi-cluster), [specified cluster resoruces](specified-cluster), and [Collection Resources](collection-resource). 

And these complex search conditions can be passed to `Clusterpedia APIServer` in two ways:
* `URL Query`: directly pass query conditions as Query
* `Search Labels`: to keep **compatible with Kubernetes OpenAPI**, the search conditions can be set via Label Selector

**Both `Search Labels` and `URL Query` support same operators as Label Selector:**
* `exist`, `not exist`
* `=`, `==`, `!=`
* `in`, `notin`

In addition to conditional retrieval, Clusterpedia also enhances [`Field Selector`](#field-selector) to meet the filtering requirements by fields such as `metadata.annotation` or `status.*`.
## Search by metadata
> Supported Operators: `==`, `=`, `in`.

|Role| search label key|url query|
| -- | --------------- | ------- |
|Filter cluster names|`search.clusterpedia.io/clusters`|`clusters`|
|Filter namespaces|`search.clusterpedia.io/namespaces`|`namespaces`|
|Filter resource names|`search.clusterpedia.io/names`|`names`|
> Current, we don't support operators such as `!=`, `notin` operators,
> if you have these needs or scenarios, you can discuss them in the issue.

## Fuzzy Search
> Supported Operators: `==`, `=`, `in`.

This feature is expermental and only search label are available for now
|Role| search label key|url query|
| -- | --------------- | ------- |
|Fuzzy Search for resource name|internalstorage.clusterpedia.io/fuzzy-name|-|

## Search by creation time interval
> Supported Operators: `==`, `=`.

The search is based on the creation time interval of the resource,
using a left-closed, right-open internval.

|Role| search label key|url query|
| -- | --------------- | ------- |
|Search|search.clusterpedia.io/since|since|
|Before|search.clusterpedia.io/before|before|

There are four formats for creation time:
1. `Unix Timestamp` for ease of use will distinguish between units of `s` or `ms` based on the length of the timestamp. The 10-bit timestamp is in seconds, the 13-bit timestamp is in milliseconds.
2. `RFC3339` *2006-01-02T15:04:05Z* or *2006-01-02T15:04:05+08:00*
3. `UTC Date` *2006-01-02*
4. `UTC Datetime` *2006-01-02 15:04:05*

Because of the limitation of the kube label selector, the search label only supports `Unix Timestamp` and `UTC Date`.

All formats are available using the url query method.

## Search by Owner
> Supported Operators: `==`, `=`.

|Role| search label key|url query|
| -- | --------------- | ------- |
|Specified Owner UID|search.clusterpedia.io/owner-uid|ownerUID|
|Specified Owner Name|search.clusterpedia.io/owner-name|ownerName|
|SPecified Owner Group Resource|search.clusterpedia.io/owner-gr|ownerGR|
|Specified Owner Seniority|`internalstorage.clusterpedia.io/owner-seniority`|`ownerSeniority`|

Note that when specifying `Owner UID`, `Owner Name` and `Owner Group Resource` will be ignored.

The format of the `Owner Group Resource` is `resource.group`, for example *deployments.apps* or *nodes*.

## OrderBy
> Supported Operators: `=`, `==`, `in`.

|Role| search label key|url query|
| -- | --------------- | ------- |
|Order by fields|`search.clusterpedia.io/orderby`|`orderby`|

## Paging
> Supported Operators: `=`, `==`.

|Role| search label key|url query|
| -- | --------------- | ------- |
|Set page size|`search.clusterpedia.io/size`|`limit`|
|Set page offset|`search.clusterpedia.io/offset`|`continue`|
|Response required with Continue|`search.clusterpedia.io/with-continue`|`withContinue`
|Response required with remaining count|`search.clusterpedia.io/with-remaining-count`|`withRemainingCount`

> When you perform operations with kubectl, the page size can only be set via `kubectl --chunk-size`, because kubectl will set the default limit to 500.

## Label Selector
Regardless of kubectl or URL, all Label Selectors that do not contain *clusterpedia.io* in the Key will be used as Label Selectors to filter resources.  

All behaviors are consistent with those provided by Kubernetes.
|Role|kubectl|url query|
| -- | --------------- | ------- |
|Filter by labels|`kubectl -l` or `kubectl --label-selector`|`labelSelector`|

## Field Selector
**Field Selector is consistent with Label Selector in terms of operators, and Clusterpedia also supports:** 

**`exist`, `not exist`, `==`, `=`, `!=`, `in`, `notin`.**

All command parameters for URL and kubectl are same as those for Field Selector.
|Role|kubectl|url query|
| -- | --------------- | ------- |
|Filter by fields|`kubectl --field-selector`|`fieldSelector`|

For details refer to:
* [search for resources by filtering fields](./multi-cluster#field-selector)
* [support field selector](https://github.com/clusterpedia-io/clusterpedia/pull/36) 
* [issue: support list field filtering](https://github.com/clusterpedia-io/clusterpedia/issues/48)

## Advanced Search(Custom Conditional Search)
Custom search is a feature provided by the `default storage layer` to meet more flexible and variable search needs of users.
|Feature| search label key |url query|
| -- | ---------------- | ------- |
|custom SQL used for filter|-|whereSQL|

Custom search is not supported by *search label*, only *url query* can be used to pass custom search SQL.

In addition, this feature is still in alpha stage, you need to open the corresponding Feature Gate in `clusterpedia apiserver`, for details, please refer to [Raw SQL Query](../../features/raw-sql-query)

## CollectionResource URL Query
The following URL Query belongs exclusively to Collection Resource.

|Role|url query| example
| -- | ------- | ------|
|[get only the metadata of the resource](collection-resource#only-metadata) |`onlyMetadata`|*onlyMetadata=true*
|[specify the groups of `any collectionresource`](collection-resource#any-collectionresource) | `groups` | *groups=apps,cert-manager.io/v1*
|[specify the resources of `any collectionresource`](collection-resource#any-collectionresource) | `resources` | *resources=apps/deployments,batch/v1/cronjobs*
