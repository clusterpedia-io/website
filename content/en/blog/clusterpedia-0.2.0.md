---
title: Clusterpedia v0.2.0 Release
date: 2022-04-12
---
## Use helm to install

Users can already use Helm to install Clusterpedia:

    $ helm install clusterpedia . \
      --namespace clusterpedia-system \
      --create-namespace \
      --set persistenceMatchNode={{ LOCAL_PV_NODE }} \
      # --set installCRDs=true

[Lean More](https://github.com/clusterpedia-io/clusterpedia/tree/main/charts)

## Use the Kube Config to import a cluster

For v0.1.0, users need to Configure the address for the imported cluster and the authentication information.

    apiVersion: cluster.clusterpedia.io/v1alpha2
    kind: PediaCluster
    metadata:
      name: cluster-example
    spec:
      apiserver: "https://10.30.43.43:6443"
      caData:
      tokenData:
      certData:
      keyData:
      syncResources: []

In v0.2.0, the `PediaCluster` added the `spec.kubeconfig` field so that users can use `kube config` to import the cluster directly.

First you need to base64 encode the kube config for the imported cluster.

    $ base64 ./kubeconfig.yaml

Set the content after the base64 to PediaCluster `spec.kubeconfig`, in addition `spec.apiserver` and other authentication fields don’t need to set.

    apiVersion: cluster.clusterpedia.io/v1alpha2
    kind: PediaCluster
    metadata:
      name: cluster-example
    spec:
      kubeconfig: **base64 kubeconfig**
      syncResources: []

However, since the cluster address is configured in kube config, the **APIServer** is empty when you use kubectl get pediacluster.

    $ kubectl get pediacluster
    NAME              APISERVER   VERSION   STATUS
    cluster-example               v1.22.2   Healthy

Mutating addmission webhooks will be added in the future to automatically set `spec.apiserver`, currently if you want to show the cluster apiserver address when `kubectl get pediacluster`, then you need to manually configure the `spec.apiserver` field additionally.

## New Search Feature

### Search by creation time interval

|Description|Search Label Key|URL Query|
|:-|:-|:-|
|Since|search.clusterpedia.io/since|since|
|Before|search.clusterpedia.io/before|before|

The creation time interval used for the search is left closed and right open, **since <= creation time < before**.

There are four formats for creation time:

1. `Unix Timestamp` for ease of use will distinguish between units of `s` or `ms` based on the length of the timestamp. The 10-bit timestamp is in seconds, the 13-bit timestamp is in milliseconds.
2. `RFC3339` *2006-01-02T15:04:05Z* or *2006-01-02T15:04:05+08:00*
3. `UTC Date` *2006-01-02*
4. `UTC Datetime` *2006-01-02 15:04:05*

>Because of the limitation of the kube label selector, the search label only supports `Unix Timestamp` and `UTC Date`.  
>  
>All formats are available using the url query method.

Look at what resources are under the default namespace

    $ kubectl --cluster clusterpedia get pods
    CLUSTER           NAME                                                 READY   STATUS      RESTARTS       AGE
    cluster-example   quickstart-ingress-nginx-admission-create--1-kxlnn   0/1     Completed   0              171d
    cluster-example   fake-pod-698dfbbd5b-wvtvw                            1/1     Running     0              8d
    cluster-example   fake-pod-698dfbbd5b-74cjx                            1/1     Running     0              21d
    cluster-example   fake-pod-698dfbbd5b-tmcw7                            1/1     Running     0              8d

We use the creation time to filter the resources.

    $ kubectl --cluster clusterpedia get pods -l "search.clusterpedia.io/since=2022-03-20"
    CLUSTER           NAME                        READY   STATUS    RESTARTS   AGE
    cluster-example   fake-pod-698dfbbd5b-wvtvw   1/1     Running   0          8d
    cluster-example   fake-pod-698dfbbd5b-tmcw7   1/1     Running   0          8d
    
    
    $ kubectl --cluster clusterpedia get pods -l "search.clusterpedia.io/before=2022-03-20"
    CLUSTER           NAME                                                 READY   STATUS      RESTARTS       AGE
    cluster-example   quickstart-ingress-nginx-admission-create--1-kxlnn   0/1     Completed   0              171d
    cluster-example   fake-pod-698dfbbd5b-74cjx                            1/1     Running     0              21d

### Search by Owner Name

As of v0.1.0, we can specify ancestor or parent `Owner UID` to query resources, but `Owner UID` is not convenient to use, after all, you still need to know the UID of the Owner resource in advance.

In v0.2.0, we support querying directly with `Owner Name`, and the Owner query has been moved from experimental to released functionality, the prefix of **Search Label** has been upgraded from *internalstorage.c lusterpedia.io* to \*search.clusterpedia.io \*, and URL Query is provided.

|Role|search label key|url query|
|:-|:-|:-|
|Specified Owner UID|search.clusterpedia.io/owner-uid|ownerUID|
|Specified Owner Name|search.clusterpedia.io/owner-name|ownerName|
|SPecified Owner Group Resource|search.clusterpedia.io/owner-gr|ownerGR|
|Specified Owner Seniority|`internalstorage.clusterpedia.io/owner-seniority`|`ownerSeniority`|

>Note that when specifying `Owner UID`, `Owner Name` and `Owner Group Resource` will be ignored.

    $ kubectl --cluster cluster-example get pods -l \
        "search.clusterpedia.io/owner-name=fake-pod, \
         search.clusterpedia.io/owner-seniority=1"
    CLUSTER           NAME                        READY   STATUS    RESTARTS   AGE
    cluster-example   fake-pod-698dfbbd5b-wvtvw   1/1     Running   0          8d
    cluster-example   fake-pod-698dfbbd5b-74cjx   1/1     Running   0          21d
    cluster-example   fake-pod-698dfbbd5b-tmcw7   1/1     Running   0          8d

**In addition, to avoid multiple types of owner resources in some cases, we can use the** `Owner Group Resource` **to restrict the type of owner.**

    $ kubectl --cluster cluster-example get pods -l \
        "search.clusterpedia.io/owner-name=fake-pod,\
         search.clusterpedia.io/owner-gr=deployments.apps,\
         search.clusterpedia.io/owner-seniority=1"
    ... some output

### Fuzzy Search base on resource names

Since fuzzy search needs to be discussed further, it is temporarily provided as an experimental feature.

Only the Search Label method is supported, URL Query isn’t supported. |Role| search label key|url query| | -- | --------------- | ------- | |Fuzzy Search for resource name|internalstorage.clusterpedia.io/fuzzy-name|-|

    $ kubectl --cluster clusterpedia get deployments -l "internalstorage.clusterpedia.io/fuzzy-name=fake"
    CLUSTER           NAME       READY   UP-TO-DATE   AVAILABLE   AGE
    cluster-example   fake-pod   3/3     3            3           113d

You can use the in operator to pass multiple fuzzy arguments, so that you can filter out resources that have all strings in their names.

## Other Features

In v0.1.0, searching the resources allow the number of remaining resources to be returned so that the user can calculate the total number of resources.

This feature has been enhanced in v0.2.0. When offset is too large, `remainingItemCount` may be negative, ensuring that the total number of resources can always be calculated.

[Lean More](https://clusterpedia.io/docs/usage/search/multi-cluster/#response-with-remaining-count)

## [Release Notes](https://github.com/clusterpedia-io/clusterpedia/releases)

* Support of using Helm Charts for installation ([\#53](https://github.com/clusterpedia-io/clusterpedia/pull/53), [\#125](https://github.com/clusterpedia-io/clusterpedia/pull/125), @calvin0327, @wzshiming)
* `PediaCluster` supports for importing a cluster using the kubeconfig ([\#115](https://github.com/clusterpedia-io/clusterpedia/pull/115), @wzshiming)

### APIServer

* Support for filtering resources by a period of creation ([\#113](https://github.com/clusterpedia-io/clusterpedia/pull/113), @cleverhu)
* Support for searching for resources by an Owner name. Now, the feature of `Search by Owner` is officially released. ([\#91](https://github.com/clusterpedia-io/clusterpedia/pull/91), @Iceber)

### Default Storage Layer

* Support for fuzzy search by a resource name ([\#117](https://github.com/clusterpedia-io/clusterpedia/pull/117), @cleverhu)
* `RemainingItemCount` can be a negative number. We can still use `offset + len(items) + remainingItemCount` to calculate the total amount of resources if the `Offset` is too large. ([\#123](https://github.com/clusterpedia-io/clusterpedia/pull/123), @cleverhu)

### Bug Fixes

* Fixed unnecessary `json.Unmarshal` and improved performance when searching ([\#89](https://github.com/clusterpedia-io/clusterpedia/pull/89), [\#92](https://github.com/clusterpedia-io/clusterpedia/pull/92), @Iceber)

### Deprecation

* `Search by Owner` has been released as an official feature. `internalstorage.clusterpedia.io/owner-name` and `internalstorage.clusterpedia.io/owner-seniority` will be removed in the next release. ([\#91](https://github.com/clusterpedia-io/clusterpedia/pull/91), @Iceber)

### Other

* `golangci-lint` is used as a static checking tool ([\#86](https://github.com/clusterpedia-io/clusterpedia/pull/86), [\#88](https://github.com/clusterpedia-io/clusterpedia/pull/88), @Iceber)
* Added CI Workloads such as static checking and unit testing for code [\#87](https://github.com/clusterpedia-io/clusterpedia/pull/87), @Iceber)
