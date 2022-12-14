---
title: "访问 Clusterpedia"
weight: 30
---

Clusterpedia 主要有两个组件：
* `ClusterSynchroManager` 管理 *主集群* 内的 `PediaCluster` 资源，通过 `PediaCluster` 配置认证信息连接到指定集群，并且实时同步相应的资源。
* `APIServer` 同样会监听 *主集群* 内的 `PediaCluster` 资源，并根据集群同步的资源以**兼容 Kubernetes OpenAPI**的方式来提供对资源的复杂检索。
* `ControllerManager`

并且 `Clusterpedia APIServer` 会以[聚合式 API](https://kubernetes.io/zh/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation) 的方式注册到 *主集群* 的 APIServer 中，
这样我们通过和主集群相同的入口便可访问 Clusterpedia

## Resources 和[聚合资源](../../concepts/collection-resource)
Clusterpedia APIServer 会在 Group —— *clusterpedia.io* 下提供两种检索资源：
```bash
kubectl api-resources | grep clusterpedia.io
```
```
# 输出：
NAME                 SHORTNAMES    APIVERSION                 NAMESPACED   KIND
collectionresources                clusterpedia.io/v1beta1    false        CollectionResource
resources                          clusterpedia.io/v1beta1    false        Resources
```
* `Resources` 用于指定资源类型的方式来检索，在使用上兼容 Kubernetes OpenAPI
* `CollectionResource` 用于检索由多个资源类型聚合而成的新的资源类型，以达到同时检索多种资源的目的

> 对于 Collection Resource 的概念和使用可以查看 [什么是聚合资源（Collection Resource）]()，[聚合资源（Collection Resource）检索]()

## 访问 Clusterpedia 资源

在检索指定类型的资源时，可以按照 **Kubernetes OpenAPI** 的 Get/List 规范来请求，
**这样我们不仅仅可以使用 URL 来访问 Clusterpedia Resources，还可以直接使用 kubectl 或者 client-go 来检索资源。**

Clusterpedia 通过 URL Path 来区分请求是多集群资源还是指定集群：

**多集群资源路径** 直接以 Resources 资源路径为前缀
*/apis/clusterpedia.io/v1beta1/resources*
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/version"
```

**指定集群资源路径** 在 Resources 资源路径的基础上设置资源名称来指定集群
*/apis/clusterpedia.io/v1beta1/resources/clusters/<cluster name>*
```bash
kubectl get --raw="/apis/clusterpedia.io/v1beta1/resources/clusters/cluster-1/version"
```

**无论是多集群资源路径还是指定集群的资源路径，都可以在路径后拼接 Kubernetes 的 Get/List Path**

### 为 kubectl 生成集群访问的快捷配置
尽管我们可以使用 URL 来访问 Clusterpedia 资源，但是如果想要更方便的使用 kubectl 来查询的话，就需要配置集群的 kubeconfig cluster 配置。

Clusterpedia 提供了一个简单的脚本来帮助生成 `cluster kube config`
```bash
curl -sfL https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/v0.6.0/hack/gen-clusterconfigs.sh | sh -
```
```
# 输出：
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
> 可以在 [hack/gen-clusterconfigs.sh](https://github.com/clusterpedia-io/clusterpedia/blob/v0.1.0/hack/gen-clusterconfigs.sh) 找到该脚本

脚本会打印当前的集群信息，并将集群中 PediaCluster 配置到 kubeconfig 中。
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
脚本生成了用于多集群访问的 *clusterpedia* cluster 以及其他 `PediaCluster` Name 命名的 cluster config，
而且在访问 Clusterpedia 时会复用主集群的入口以及认证信息，相比主集群入口只是增加了 Clusterpedia Resources 的 path。

多集群的 kubeconfig 信息生成完成后，就可以使用 `kubectl --cluster` 来指定集群访问了
```bash
# 多集群检索时支持的资源
kubectl --cluster clusterpedia api-resources

# cluster-1 支持检索的资源
kubectl --cluster cluster-1 api-resources
```

## 查看支持检索的资源类型
我们可以根据 URL 路径来分别获取全局资源信息和指定集群的资源信息。

**全局资源信息是所有集群同步的资源类型的并集。**

Clusterpedia 开放的 `Discovery API` 同样**兼容 Kubernetes OpenAPI**，可以使用 `kubectl`，[client-go/discovery](https://pkg.go.dev/k8s.io/client-go/discovery#DiscoveryClient)，[client-go/restmapper](https://pkg.go.dev/k8s.io/client-go/restmapper) 或者 [controller-runtime/dynamic-restmapper](https://pkg.go.dev/sigs.k8s.io/controller-runtime/pkg/client/apiutil#NewDynamicRESTMapper) 来访问。

{{< tabs >}}

{{% tab name="URL" %}}
**使用 URL 来获取 APIGroupList 以及 APIGroup 信息**
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
**使用 kubectl 来获取 api-resources**
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
{{< /tab >}}

{{< /tabs >}}
