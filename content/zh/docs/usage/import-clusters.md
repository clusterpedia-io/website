---
title: 集群接入
weight: 1
---

Clusterpedia 通过 `PediaCluster` 资源来接入集群，通过配置集群的 APIServer 地址以及验证信息来连接到指定集群中
```yaml
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
```

`apiserver` 作为被接入集群的地址时必不可少的，而验证字段的配置有多种选择：
* `caData` + `tokenData`
* `caData` + `certData` + `keyData`
> `caData` 在集群 APIServer 允许 Insecure 连接的情况下，也可以不填

这些验证字段都需要 base64 编码，如果这些字段的值是直接从 ConfigMap 或者 Secret 中获取的话，那么就已经 base64 过。

## 使用 ServiceAccount 来接入集群
用户可以选择在**被接入集群**中创建 ServiceAccount 并配置相应的 RBAC 来接入集群。

``` bash
# 注意：当前 kubectl 连接到被接入集群
kubectl apply -f https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/main/examples/clusterpedia_synchro_rbac.yaml

# 获取 Service Account 对应 CA 和 Token
SYNCHRO_CA=$(kubectl get secret $(kubectl get serviceaccount clusterpedia-synchro -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.ca\.crt}')
SYNCHRO_TOKEN=$(kubectl get secret $(kubectl get serviceaccount clusterpedia-synchro -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}')
```
将 `$SYNCHRO_CA` 和 `SYNCHRO_TOKEN` 分别填写到 PediaCluster 资源的 `spec.caData` 和 `spec.tokenData` 字段中

## 创建 PediaCluster
完善集群的验证信息后，就可以获得一个完整的 `PediaCluster` 资源了。直接使用 `kubectl apply -f` 直接创建即可
```yaml
apiVersion: cluster.clusterpedia.io/v1alpha2
kind: PediaCluster
metadata:
  name: cluster-example
spec:
  apiserver: https://10.6.100.10:6443
  caData: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeE1Ea3lOREV3TVRNeU5Gb1hEVE14TURreU1qRXdNVE15TkZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTy9TCnZta1U5bk1uUlRIT3lvK3hjdFRJY0lPYnMzc0F5cTI2djRQYkVtb3ZWM2xPOVQwMTF2cE96S0pyOUFxeVZMRnYKVXFBRHBTakM3WXd3MnZwSld3bDEySlBvUm1xZ1FBSFNkYlJpU3BDTDRudjlvR25VOWI2dllWSy9iRitkUVFCSApnQ1h6NnZoTGY4Wmd2N2tUQ2JBdkFPaE9OSlU3MllYTE8zT0lZQjJva1NCRGFVUjNvNnpwZGVWTkt5V0EyNVA3CkRobk8yTk01QzlpRERqTTRLY2FTa3JPSkJvbUlsSHFZRjRwVXdTTlFvcGVGRVRyZ3ZzcTkwSks2YUJVS0t5ajYKK2NGdjI3S0k4K1ZMUEtaSTE2c25Mbng2RXRTazZtZjJXTHdJZlhyQlgwREsvYXBEQ015R2pEb2dCaGpJSVhoVAp2bjVQZndFWUNsdGZFTEhKSkdVQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZJVDhLRHdCbUVvMHladUFEZkhkKzQ1L3ZFYzdNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBT0F5VHQ4S3ZFN0dvREhQT09pdgoyR2I2WWVsUU5KcUMza1dIOXc1NTFNaGZvS3ZiM21VaUV6ZVMwOUNwZUQrTFh5ZnlqQzhZYkJxQjZXSFhNZWMrCnpPdDNPazRYV0FmZVVZTXhOQ1FJblc4cjI4cmZnblErc1NCdHQyeERQN1RZY09oNVZGZkI2K3JtTmFTblZ1NjgKSFFxdlFMNEFXbVhkR09jRWNBRThYdkdiOWhwSjVNckRHdzQ0UTYyOG9YazZ0N01aWTFOMUNQdW9HZ1VmS1N3bgo1MUFWRTFOVVdNV0tEQXhaa2I4bEhvR3VWaDFzWmd3SnJRQjR5clh1cmxGN0Y2bVRlYm4rcDVKM0toT0V4KzlsCjFXdkwwbWkxL1J2bVJKNm11YmtjWUwzN1FJWjI1YXdyaEZMN0Z1ejNRSTFqTTdYMHZET2VUM2VuVUFCZW5SMS8KUnlnPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  tokenData: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklrMHRSalJtZGpSdVgxcFljMGxsU1ZneFlXMHpPSFZOY0Zwbk1UTkhiVFpsVFZwQ2JIWk9SbU5XYW5NaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbU5zZFhOMFpYSndaV1JwWVMxemVXNWphSEp2TFhSdmEyVnVMVGsxYTJSNElpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WlhKMmFXTmxMV0ZqWTI5MWJuUXVibUZ0WlNJNkltTnNkWE4wWlhKd1pXUnBZUzF6ZVc1amFISnZJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpYSjJhV05sTFdGalkyOTFiblF1ZFdsa0lqb2lNREl5WXpNMk5USXRPR1k0WkMwME5qSmtMV0l6TnpFdFpUVXhPREF3TnpFeE9HUTBJaXdpYzNWaUlqb2ljM2x6ZEdWdE9uTmxjblpwWTJWaFkyTnZkVzUwT21SbFptRjFiSFE2WTJ4MWMzUmxjbkJsWkdsaExYTjVibU5vY204aWZRLkF4ZjhmbG5oR0lDYjJaMDdkT0FKUW11aHVIX0ZzRzZRSVY5Sm5sSmtPUnF5aGpWSDMyMkVqWDk1bVhoZ2RVQ2RfZXphRFJ1RFFpLTBOWDFseGc5OXpYRks1MC10ZzNfYlh5NFA1QnRFOUpRNnNraUt4dDFBZVJHVUF4bG5fVFU3SHozLTU5Vnl5Q3NwckFZczlsQWQwRFB6bTRqb1dyS1lKUXpPaGl5VjkzOWpaX2ZkS1BVUmNaMVVKVGpXUTlvNEFFY0hMdDlyTEJNMTk2eDRkbzA4ZHFaUnVtTzJZRXFkQTB3ZnRxZ2NGQzdtTGlSVVhkWElkYW9CY1BuWXBwM01MU3B5QjJQMV9vSlRFNS1nd3k4N2Jwb3U1RXo2TElSSExIeW5NWXAtWVRLR2hBbDJwMXdJb0tDZUNnQng4RlRfdzM4Rnh1TnE0UDRoQW5RUUh6bU9Ndw==
  syncResources: []
```
## 集群查看
集群接入成功后，可以通过 `kubectl get pediacluster` 命令来查看所有接入的集群，以及集群状态
```bash
kubectl get pediacluster
```
```
# 输出：
NAME           APISERVER URL               VERSION    STATUS
cluster-1      https://10.6.100.10:6443    v1.22.2    Healthy
cluster-2      https://10.50.10.11:16443   v1.10.11   Healthy
```

## 接下来
继续查看 [同步集群资源](../sync-resources)
