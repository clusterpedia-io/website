---
title: "Access the Clusterpedia"
weight: 30
---

Clusterpedia has two main components:
* `ClusterSynchroManager` manages the `PediaCluster` resource in the *master cluster*, connects to the specified cluster through the `PediaCluster` authentication information, and synchronizes the corresponding resources in real time.
* `APIServer` also listens to the `PediaCluster` resource in the *master cluster* and provides complex search for resources in a **compatible Kubernetes OpenAPI** manner based on the resources synchronized by the cluster.  
* `Controller Manager`

Also, the `Clusterpedia APIServer` will be registered to the *master cluster* APIServer in the way of [Aggregation API](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation), so that we can access Clusterpedia through the same entry as the master cluster.

## Resources and [Collection Resource](../../concepts/collection-resource)
Clusterpedia APIServer will provide two different resources to search under Group - *clusterpedia.io*:
```bash
kubectl api-resources | grep clusterpedia.io
```
```
# Output:
NAME                 SHORTNAMES    APIVERSION                 NAMESPACED   KIND
collectionresources                clusterpedia.io/v1beta1    false        CollectionResource
resources                          clusterpedia.io/v1beta1    false        Resources
```
* `Resources` is used to specify a resource type to search for, compatible with Kubernetes OpenAPI
* `CollectionResource` is used to search for new resource aggregated by different types to find multiple resource types at one time

> For concepts and usage about Collection Resource, refer to [What is Collection Resource](../../concepts/collection-resource) and [Search for Collection Resource](../search/collection-resource).

## Access the Clusterpedia resources

When searching for a resource of a specific type, you can request it according to the `Get`/`List` specification of **Kubernetes OpenAPI**. **In this way we can not only use the URL to access Clusterpedia resources, but also directly use `kubectl` or `client-go` to search for the resources.**

Clusterpedia uses URL Path to distinguish whether the request is a multi-cluster resource or a specific cluster:

**Multi-cluster resource path**, directly prefix the Resources path:

*/apis/clusterpedia.io/v1beta1/resources*
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/version"
```

**Specific cluster resource path**, specify a cluster by setting the resource name based on the Resources path
*/apis/clusterpedia.io/v1beta1/resources/clusters/<cluster name>*
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/clusters/cluster-1/version"
```

**Regardless of a resource path of multiple clusters or a specific cluster, the path can be spliced and followed by Kubernetes `Get`/`List` Path**

### Configure the cluster shortcut for kubectl
Although we can use URLs to access Clusterpedia resources, if we want to use kubectl to query more conveniently, we need to configure the kubeconfig cluster.

Clusterpedia provides a simple script to generate `cluster config` in the kubeconfig.
```bash
curl -sfL https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/v0.6.0/hack/gen-clusterconfigs.sh | sh -
```
```
# Output:
Current Context: kubernetes-admin@kubernetes
Current Cluster: kubernetes
        Server: https://10.6.100.10:6443
        TLS Server Name:
        Insecure Skip TLS Verify:
        Certificate Authority:
        Certificate Authority Data: ***

Cluster "clusterpedia" set.
Cluster "cluster-1" set.
Cluster "cluster-2" set.
```
> Check the script from [hack/gen-clusterconfigs.sh](https://github.com/clusterpedia-io/clusterpedia/blob/v0.1.0/hack/gen-clusterconfigs.sh)

The script prints the current cluster information and configures the `PediaCluster` into kubeconfig.
```bash
cat ~/.kube/config
```
```yaml
# .kube/config
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeE1Ea3lOREV3TVRNeU5Gb1hEVE14TURreU1qRXdNVE15TkZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTy9TCnZta1U5bk1uUlRIT3lvK3hjdFRJY0lPYnMzc0F5cTI2djRQYkVtb3ZWM2xPOVQwMTF2cE96S0pyOUFxeVZMRnYKVXFBRHBTakM3WXd3MnZwSld3bDEySlBvUm1xZ1FBSFNkYlJpU3BDTDRudjlvR25VOWI2dllWSy9iRitkUVFCSApnQ1h6NnZoTGY4Wmd2N2tUQ2JBdkFPaE9OSlU3MllYTE8zT0lZQjJva1NCRGFVUjNvNnpwZGVWTkt5V0EyNVA3CkRobk8yTk01QzlpRERqTTRLY2FTa3JPSkJvbUlsSHFZRjRwVXdTTlFvcGVGRVRyZ3ZzcTkwSks2YUJVS0t5ajYKK2NGdjI3S0k4K1ZMUEtaSTE2c25Mbng2RXRTazZtZjJXTHdJZlhyQlgwREsvYXBEQ015R2pEb2dCaGpJSVhoVAp2bjVQZndFWUNsdGZFTEhKSkdVQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZJVDhLRHdCbUVvMHladUFEZkhkKzQ1L3ZFYzdNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBT0F5VHQ4S3ZFN0dvREhQT09pdgoyR2I2WWVsUU5KcUMza1dIOXc1NTFNaGZvS3ZiM21VaUV6ZVMwOUNwZUQrTFh5ZnlqQzhZYkJxQjZXSFhNZWMrCnpPdDNPazRYV0FmZVVZTXhOQ1FJblc4cjI4cmZnblErc1NCdHQyeERQN1RZY09oNVZGZkI2K3JtTmFTblZ1NjgKSFFxdlFMNEFXbVhkR09jRWNBRThYdkdiOWhwSjVNckRHdzQ0UTYyOG9YazZ0N01aWTFOMUNQdW9HZ1VmS1N3bgo1MUFWRTFOVVdNV0tEQXhaa2I4bEhvR3VWaDFzWmd3SnJRQjR5clh1cmxGN0Y2bVRlYm4rcDVKM0toT0V4KzlsCjFXdkwwbWkxL1J2bVJKNm11YmtjWUwzN1FJWjI1YXdyaEZMN0Z1ejNRSTFqTTdYMHZET2VUM2VuVUFCZW5SMS8KUnlnPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://10.6.100.10:6443/apis/clusterpedia.io/v1beta1/resources/clusters/cluster-1
  name: cluster-1
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeE1Ea3lOREV3TVRNeU5Gb1hEVE14TURreU1qRXdNVE15TkZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTy9TCnZta1U5bk1uUlRIT3lvK3hjdFRJY0lPYnMzc0F5cTI2djRQYkVtb3ZWM2xPOVQwMTF2cE96S0pyOUFxeVZMRnYKVXFBRHBTakM3WXd3MnZwSld3bDEySlBvUm1xZ1FBSFNkYlJpU3BDTDRudjlvR25VOWI2dllWSy9iRitkUVFCSApnQ1h6NnZoTGY4Wmd2N2tUQ2JBdkFPaE9OSlU3MllYTE8zT0lZQjJva1NCRGFVUjNvNnpwZGVWTkt5V0EyNVA3CkRobk8yTk01QzlpRERqTTRLY2FTa3JPSkJvbUlsSHFZRjRwVXdTTlFvcGVGRVRyZ3ZzcTkwSks2YUJVS0t5ajYKK2NGdjI3S0k4K1ZMUEtaSTE2c25Mbng2RXRTazZtZjJXTHdJZlhyQlgwREsvYXBEQ015R2pEb2dCaGpJSVhoVAp2bjVQZndFWUNsdGZFTEhKSkdVQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZJVDhLRHdCbUVvMHladUFEZkhkKzQ1L3ZFYzdNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBT0F5VHQ4S3ZFN0dvREhQT09pdgoyR2I2WWVsUU5KcUMza1dIOXc1NTFNaGZvS3ZiM21VaUV6ZVMwOUNwZUQrTFh5ZnlqQzhZYkJxQjZXSFhNZWMrCnpPdDNPazRYV0FmZVVZTXhOQ1FJblc4cjI4cmZnblErc1NCdHQyeERQN1RZY09oNVZGZkI2K3JtTmFTblZ1NjgKSFFxdlFMNEFXbVhkR09jRWNBRThYdkdiOWhwSjVNckRHdzQ0UTYyOG9YazZ0N01aWTFOMUNQdW9HZ1VmS1N3bgo1MUFWRTFOVVdNV0tEQXhaa2I4bEhvR3VWaDFzWmd3SnJRQjR5clh1cmxGN0Y2bVRlYm4rcDVKM0toT0V4KzlsCjFXdkwwbWkxL1J2bVJKNm11YmtjWUwzN1FJWjI1YXdyaEZMN0Z1ejNRSTFqTTdYMHZET2VUM2VuVUFCZW5SMS8KUnlnPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://10.6.100.10:6443/apis/clusterpedia.io/v1beta1/resources/clusters/cluster-2
  name: cluster-2
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeE1Ea3lOREV3TVRNeU5Gb1hEVE14TURreU1qRXdNVE15TkZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTy9TCnZta1U5bk1uUlRIT3lvK3hjdFRJY0lPYnMzc0F5cTI2djRQYkVtb3ZWM2xPOVQwMTF2cE96S0pyOUFxeVZMRnYKVXFBRHBTakM3WXd3MnZwSld3bDEySlBvUm1xZ1FBSFNkYlJpU3BDTDRudjlvR25VOWI2dllWSy9iRitkUVFCSApnQ1h6NnZoTGY4Wmd2N2tUQ2JBdkFPaE9OSlU3MllYTE8zT0lZQjJva1NCRGFVUjNvNnpwZGVWTkt5V0EyNVA3CkRobk8yTk01QzlpRERqTTRLY2FTa3JPSkJvbUlsSHFZRjRwVXdTTlFvcGVGRVRyZ3ZzcTkwSks2YUJVS0t5ajYKK2NGdjI3S0k4K1ZMUEtaSTE2c25Mbng2RXRTazZtZjJXTHdJZlhyQlgwREsvYXBEQ015R2pEb2dCaGpJSVhoVAp2bjVQZndFWUNsdGZFTEhKSkdVQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZJVDhLRHdCbUVvMHladUFEZkhkKzQ1L3ZFYzdNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBT0F5VHQ4S3ZFN0dvREhQT09pdgoyR2I2WWVsUU5KcUMza1dIOXc1NTFNaGZvS3ZiM21VaUV6ZVMwOUNwZUQrTFh5ZnlqQzhZYkJxQjZXSFhNZWMrCnpPdDNPazRYV0FmZVVZTXhOQ1FJblc4cjI4cmZnblErc1NCdHQyeERQN1RZY09oNVZGZkI2K3JtTmFTblZ1NjgKSFFxdlFMNEFXbVhkR09jRWNBRThYdkdiOWhwSjVNckRHdzQ0UTYyOG9YazZ0N01aWTFOMUNQdW9HZ1VmS1N3bgo1MUFWRTFOVVdNV0tEQXhaa2I4bEhvR3VWaDFzWmd3SnJRQjR5clh1cmxGN0Y2bVRlYm4rcDVKM0toT0V4KzlsCjFXdkwwbWkxL1J2bVJKNm11YmtjWUwzN1FJWjI1YXdyaEZMN0Z1ejNRSTFqTTdYMHZET2VUM2VuVUFCZW5SMS8KUnlnPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://10.6.100.10:6443/apis/clusterpedia.io/v1beta1/resources
  name: clusterpedia
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeE1Ea3lOREV3TVRNeU5Gb1hEVE14TURreU1qRXdNVE15TkZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTy9TCnZta1U5bk1uUlRIT3lvK3hjdFRJY0lPYnMzc0F5cTI2djRQYkVtb3ZWM2xPOVQwMTF2cE96S0pyOUFxeVZMRnYKVXFBRHBTakM3WXd3MnZwSld3bDEySlBvUm1xZ1FBSFNkYlJpU3BDTDRudjlvR25VOWI2dllWSy9iRitkUVFCSApnQ1h6NnZoTGY4Wmd2N2tUQ2JBdkFPaE9OSlU3MllYTE8zT0lZQjJva1NCRGFVUjNvNnpwZGVWTkt5V0EyNVA3CkRobk8yTk01QzlpRERqTTRLY2FTa3JPSkJvbUlsSHFZRjRwVXdTTlFvcGVGRVRyZ3ZzcTkwSks2YUJVS0t5ajYKK2NGdjI3S0k4K1ZMUEtaSTE2c25Mbng2RXRTazZtZjJXTHdJZlhyQlgwREsvYXBEQ015R2pEb2dCaGpJSVhoVAp2bjVQZndFWUNsdGZFTEhKSkdVQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZJVDhLRHdCbUVvMHladUFEZkhkKzQ1L3ZFYzdNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBT0F5VHQ4S3ZFN0dvREhQT09pdgoyR2I2WWVsUU5KcUMza1dIOXc1NTFNaGZvS3ZiM21VaUV6ZVMwOUNwZUQrTFh5ZnlqQzhZYkJxQjZXSFhNZWMrCnpPdDNPazRYV0FmZVVZTXhOQ1FJblc4cjI4cmZnblErc1NCdHQyeERQN1RZY09oNVZGZkI2K3JtTmFTblZ1NjgKSFFxdlFMNEFXbVhkR09jRWNBRThYdkdiOWhwSjVNckRHdzQ0UTYyOG9YazZ0N01aWTFOMUNQdW9HZ1VmS1N3bgo1MUFWRTFOVVdNV0tEQXhaa2I4bEhvR3VWaDFzWmd3SnJRQjR5clh1cmxGN0Y2bVRlYm4rcDVKM0toT0V4KzlsCjFXdkwwbWkxL1J2bVJKNm11YmtjWUwzN1FJWjI1YXdyaEZMN0Z1ejNRSTFqTTdYMHZET2VUM2VuVUFCZW5SMS8KUnlnPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
    server: https://10.6.100.10:6443
  name: kubernetes
```
The script generates *clusterpedia* clusters for multi-cluster access and other cluster configs in the name of `PediaCluster`, and reuses the master cluster's entry and authentication information when accessing Clusterpedia.

Compared with the master cluster entry, it only adds Clusterpedia Resources path.

After multi-cluster kubeconfig is generated, you can use `kubectl --cluster` to specify the cluster access
```bash
# Supported resources for multi-cluster search
kubectl --cluster clusterpedia api-resources

# Supported resources for cluster-1 search
kubectl --cluster cluster-1 api-resources
```

## What resources are supported for search
We can get the global and specific resource information according to the URL path.

**Global resource information is the union of resource types that are synchronized across all clusters**

 `Discovery API` opened by Clusterpedia is similary **compatible with Kubernetes OpenAPI**. You can use `kubectl`, [client-go/discovery](https://pkg.go.dev/k8s.io/client-go/discovery#DiscoveryClient), [client-go/restmapper](https://pkg.go.dev/k8s.io/client-go/restmapper) or [controller-runtime/dynamic-restmapper](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client/apiutil#NewDynamicRESTMapper) to access it.

{{< tabs >}}

{{% tab name="URL" %}}
**Use URL to get APIGroupList and APIGroup information**
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis" | jq
```
```json
{
  "kind": "APIGroupList",
  "apiVersion": "v1",
  "groups": [
    {
      "name": "apps",
      "versions": [
        {
          "groupVersion": "apps/v1",
          "version": "v1"
        },
        {
          "groupVersion": "apps/v1beta2",
          "version": "v1beta2"
        },
        {
          "groupVersion": "apps/v1beta1",
          "version": "v1beta1"
        }
      ],
      "preferredVersion": {
        "groupVersion": "apps/v1",
        "version": "v1"
      }
    },
    {
      "name": "cert-manager.io",
      "versions": [
        {
          "groupVersion": "cert-manager.io/v1",
          "version": "v1"
        }
      ],
      "preferredVersion": {
        "groupVersion": "cert-manager.io/v1",
        "version": "v1"
      }
    }
  ]
}
```

```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/apis/apps" | jq
```
```json
{
  "kind": "APIGroup",
  "apiVersion": "v1",
  "name": "apps",
  "versions": [
    {
      "groupVersion": "apps/v1",
      "version": "v1"
    },
    {
      "groupVersion": "apps/v1beta2",
      "version": "v1beta2"
    },
    {
      "groupVersion": "apps/v1beta1",
      "version": "v1beta1"
    }
  ],
  "preferredVersion": {
    "groupVersion": "apps/v1",
    "version": "v1"
  }
}
```
{{% /tab %}}

{{% tab name="kubectl" %}}
**Use kubectl to get api-resources**
```bash
kubectl --cluster clusterpedia api-resources
```
```
# Output:
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
{{< /tab >}}

{{< /tabs >}}
