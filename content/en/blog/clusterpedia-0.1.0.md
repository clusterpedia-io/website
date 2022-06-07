---
title: Clusterpedia v0.1.0 Release — four important functions
date: 2022-02-16
---
**This is the first release of** [**Clusterpedia**](https://clusterpedia.io) **🥳🥳🥳, and it also means that it is officially in the iteration phase.**

Compared to the initial [v0.0.8](https://github.com/clusterpedia-io/clusterpedia/tree/v0.0.8) and [v0.0.9-alpha](https://github.com/clusterpedia-io/clusterpedia/tree/v0.0.9-alpha), v0.1.0 add a lot of features and makes some incompatible updates.

>If upgrading from v0.0.9-alpha or v0.0.8, you can refer to [Upgrade to Clusterpedia 0.1.0](/blog/2022/02/15/upgrade-to-clusterpedia-0.1.0/)

# Features Preview
|Role|Search Label Key|URL Query|
|:-|:-|:-|
|Filter cluster names|`search.clusterpedia.io/clusters`|`clusters`|
|Filter namespaces|`search.clusterpedia.io/namespaces`|`namespaces`|
|Filter resource names|`search.clusterpedia.io/names`|`names`|
|Specified Owner UID|`internalstorage.clusterpedia.io/owner-uid`|\-|
|Specified Owner Seniority|`internalstorage.clusterpedia.io/owner-seniority`|`ownerSeniority`|
|Order by fields|`search.clusterpedia.io/orderby`|`orderby`|
|Set page size|`search.clusterpedia.io/size`|`limit`|
|Set page offset|`search.clusterpedia.io/offset`|`continue`|
|Response include Continue|`search.clusterpedia.io/with-continue`|`withContinue`|
|Response include remaining count|`search.clusterpedia.io/with-remaining-count`|`withRemainingCount`|

Native `Label Selector` and enhanced `Field Selector` supported in addition to search label.

# Important Features
Let's start with the more important features that have been added in 0.1.0

* The number of remaining items carried in response data
* Added warnning when searching for resources in a `Not Ready` cluster
* Enhancements to the native FieldSelector
* Search by Parent or Ancestor Owner

## Warnning alert on resource search

When a cluster is not ready for some reason, resources are often not synchronised properly either.

Warnning alerts are used to alert users of cluster exceptions when searching for resources within the cluster, and the resources searched may not be accurate in real time.

    $ kubectl get pediacluster
    NAME        APISERVER                   VERSION   STATUS
    cluster-1   https://10.6.100.10:6443    v1.22.2   ClusterSynchroStop
    
    $ kubectl --cluster cluster-1 get pods
    Warning: cluster-1 is not ready and the resources obtained may be inaccurate, reason: ClusterSynchroStop
    CLUSTER     NAME                                                 READY   STATUS      RESTARTS   AGE
    cluster-1   fake-pod-698dfbbd5b-64fsx                            1/1     Running     0          68d
    cluster-1   fake-pod-698dfbbd5b-9ftzh                            1/1     Running     0          39d
    cluster-1   fake-pod-698dfbbd5b-rk74p                            1/1     Running     0          39d
    cluster-1   quickstart-ingress-nginx-admission-create--1-kxlnn   0/1     Completed   0          126d

## Field Selector

Native Kubernetes currently only supports field filtering on `metadata.name` and `metadata.namespace`, and the operators only support =, !=, ==\`, which is very limited.

Although some specific resources will support some special fields, the use is still rather limited

    # kubernetes/pkg
    $ grep AddFieldLabelConversionFunc . -r
    ./apis/core/v1/conversion.go:   err := scheme.AddFieldLabelConversionFunc(SchemeGroupVersion.WithKind("Pod"),
    ./apis/core/v1/conversion.go:   err = scheme.AddFieldLabelConversionFunc(SchemeGroupVersion.WithKind("Node"),
    ./apis/core/v1/conversion.go:   err = scheme.AddFieldLabelConversionFunc(SchemeGroupVersion.WithKind("ReplicationController"),
    ./apis/core/v1/conversion.go:   return scheme.AddFieldLabelConversionFunc(SchemeGroupVersion.WithKind("Event"),
    ./apis/core/v1/conversion.go:   return scheme.AddFieldLabelConversionFunc(SchemeGroupVersion.WithKind("Namespace"),
    ./apis/core/v1/conversion.go:   return scheme.AddFieldLabelConversionFunc(SchemeGroupVersion.WithKind("Secret"),
    ./apis/certificates/v1/conversion.go:   return scheme.AddFieldLabelConversionFunc(SchemeGroupVersion.WithKind("CertificateSigningRequest"),
    ./apis/certificates/v1beta1/conversion.go:      return scheme.AddFieldLabelConversionFunc(SchemeGroupVersion.WithKind("CertificateSigningRequest"),
    ./apis/batch/v1/conversion.go:  return scheme.AddFieldLabelConversionFunc(SchemeGroupVersion.WithKind("Job"),
    ./apis/batch/v1beta1/conversion.go:             err = scheme.AddFieldLabelConversionFunc(SchemeGroupVersion.WithKind(kind),
    ./apis/events/v1/conversion.go: return scheme.AddFieldLabelConversionFunc(SchemeGroupVersion.WithKind("Event"),
    ./apis/events/v1beta1/conversion.go:    return scheme.AddFieldLabelConversionFunc(SchemeGroupVersion.WithKind("Event"),
    ./apis/apps/v1beta2/conversion.go:      if err := scheme.AddFieldLabelConversionFunc(SchemeGroupVersion.WithKind("StatefulSet"),
    ./apis/apps/v1beta1/conversion.go:      if err := scheme.AddFieldLabelConversionFunc(SchemeGroupVersion.WithKind("StatefulSet"),

Clusterpedia provides more powerful features based on the compatibility with existing Field Selector features, and supports the same operators as Label Selector: `!`, `=`, `!=`, `==`, `in`, `notin`.

For example, we can filter by annotations, like label selector

    kubectl get deploy --field-selector="metadata.annotations['test.io'] in (value1, value2)"

[Lean More](https://clusterpedia.io/docs/usage/search/multi-cluster/#field-selector)

## Search by Parent or Ancestor Owner

There will usually be an Owner relationship between Kubernetes resources.

    apiVersion: v1
    kind: Pod
    metadata:
      ownerReferences:
      - apiVersion: apps/v1
        blockOwnerDeletion: true
        controller: true
        kind: ReplicaSet
        name: fake-pod-698dfbbd5b
        uid: d5bf2bdd-47d2-4932-84fb-98bde486d244

Searching by Owner is a very useful search function, and Clusterpedia also supports the seniority advancement of Owner to search for grandparents and even higher seniority.

By searching by Owner, we can query all Pods under Deployment at once, without having to query ReplicaSet in between

>Currently only supports query by Owner UID. The feature of using Owner Name for queries is still under discussion, we can join the discussion in the [issue: Support for searching resources by owner](https://github.com/clusterpedia-io/clusterpedia/issues/49)

    $ DEPLOY_UID=$(kubectl --cluster cluster-1 get deploy fake-deploy -o jsonpath="{.metadata.uid}")
    $ kubectl --cluster cluster-1 get pods -l \
        "internalstorage.clusterpedia.io/owner-uid=$DEPLOY_UID,\
         internalstorage.clusterpedia.io/owner-seniority=1"

[Lean More](https://clusterpedia.io/docs/usage/search/multi-cluster/#search-by-parent-or-ancestor-owner)

## The number of remaining items carried in response data

**In some UI cases, it is often necessary to get the total number of resources in the current search condition.**

The RemainingItemCount field exists in the [ListMeta](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#ListMeta) of the Kubernetes List response.

    type ListMeta struct {
        ...
    
            // remainingItemCount is the number of subsequent items in the list which are not included in this
            // list response. If the list request contained label or field selectors, then the number of
            // remaining items is unknown and the field will be left unset and omitted during serialization.
            // If the list is complete (either because it is not chunking or because this is the last chunk),
            // then there are no more remaining items and this field will be left unset and omitted during
            // serialization.
            // Servers older than v1.15 do not set this field.
            // The intended use of the remainingItemCount is *estimating* the size of a collection. Clients
            // should not rely on the remainingItemCount to be set or to be exact.
            // +optional
        RemainingItemCount *int64 `json:"remainingItemCount,omitempty" protobuf:"bytes,4,opt,name=remainingItemCount"`
    }

By reusing this field, the total number of resources can be returned in a Kubernetes OpenAPI-compatible manner:

**offset + len(list.items) + list.metadata.remainingItemCount**

>Use with Paging

    $ kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments?withRemainingCount&limit=1" | jq
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

[Lean More](https://clusterpedia.io/docs/usage/search/multi-cluster/#response-with-remaining-count)

# [Realease v0.1.0](https://github.com/clusterpedia-io/clusterpedia/releases/tag/v0.1.0)
