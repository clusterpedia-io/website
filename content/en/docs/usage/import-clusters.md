---
title: Import a cluster
weight: 1
---

Clusterpedia uses `PediaCluster` to import a cluster and configures cluster APIServer address and authentication information to connect a specific cluster.
```yaml
apiVersion: clusters.clusterpedia.io/v1alpha1
kind: PediaCluster
metadata:
  name: cluster-example
spec:
  apiserverURL: "https://10.30.43.43:6443"
  caData:
  tokenData:
  certData:
  keyData:
```

`apiserverURL` is required as the address of a connected cluster and you have several options to configure the authentication fields:
* `caData` + `tokenData`
* `caData` + `certData` + `keyData`
> `caData` can be left blank if the cluster APIServer allows Insecure connections

All these authentication fields need to be encoded by base64. If the field values are obtained directly from ConfigMap or Secret, they have already been encoded by base64.

## Use ServiceAccount to import a cluster
You can choose to create a ServiceAccount in the **Imported Cluster** and configure the proper RBAC to import the cluster.

``` bash
# Connect the current kubectl to the imported cluster
kubectl apply -f https://raw.githubusercontent.com/clusterpedia-io/clusterpedia/main/examples/clusterpedia_synchro_rbac.yaml

# Get CA and Token for Service Account
SYNCHRO_CA=$(kubectl get secret $(kubectl get serviceaccount clusterpedia-synchro -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.ca\.crt}')
SYNCHRO_TOKEN=$(kubectl get secret $(kubectl get serviceaccount clusterpedia-synchro -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}')
```
Fill `$SYNCHRO_CA` and `SYNCHRO_TOKEN` into `spec.caData` and `spec.tokenData` fields for the PediaCluster resource.

## Create PediaCluster
After completing the cluster authentication fields, you can get a complete `PediaCluster` resource and can directly use `kubectl apply -f` to create it.
```yaml
apiVersion: clusters.clusterpedia.io/v1alpha1
kind: PediaCluster
metadata:
  name: cluster-example
spec:
  apiserverURL: https://10.6.100.10:6443
  caData: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUMvakNDQWVhZ0F3SUJBZ0lCQURBTkJna3Foa2lHOXcwQkFRc0ZBREFWTVJNd0VRWURWUVFERXdwcmRXSmwKY201bGRHVnpNQjRYRFRJeE1Ea3lOREV3TVRNeU5Gb1hEVE14TURreU1qRXdNVE15TkZvd0ZURVRNQkVHQTFVRQpBeE1LYTNWaVpYSnVaWFJsY3pDQ0FTSXdEUVlKS29aSWh2Y05BUUVCQlFBRGdnRVBBRENDQVFvQ2dnRUJBTy9TCnZta1U5bk1uUlRIT3lvK3hjdFRJY0lPYnMzc0F5cTI2djRQYkVtb3ZWM2xPOVQwMTF2cE96S0pyOUFxeVZMRnYKVXFBRHBTakM3WXd3MnZwSld3bDEySlBvUm1xZ1FBSFNkYlJpU3BDTDRudjlvR25VOWI2dllWSy9iRitkUVFCSApnQ1h6NnZoTGY4Wmd2N2tUQ2JBdkFPaE9OSlU3MllYTE8zT0lZQjJva1NCRGFVUjNvNnpwZGVWTkt5V0EyNVA3CkRobk8yTk01QzlpRERqTTRLY2FTa3JPSkJvbUlsSHFZRjRwVXdTTlFvcGVGRVRyZ3ZzcTkwSks2YUJVS0t5ajYKK2NGdjI3S0k4K1ZMUEtaSTE2c25Mbng2RXRTazZtZjJXTHdJZlhyQlgwREsvYXBEQ015R2pEb2dCaGpJSVhoVAp2bjVQZndFWUNsdGZFTEhKSkdVQ0F3RUFBYU5aTUZjd0RnWURWUjBQQVFIL0JBUURBZ0trTUE4R0ExVWRFd0VCCi93UUZNQU1CQWY4d0hRWURWUjBPQkJZRUZJVDhLRHdCbUVvMHladUFEZkhkKzQ1L3ZFYzdNQlVHQTFVZEVRUU8KTUF5Q0NtdDFZbVZ5Ym1WMFpYTXdEUVlKS29aSWh2Y05BUUVMQlFBRGdnRUJBT0F5VHQ4S3ZFN0dvREhQT09pdgoyR2I2WWVsUU5KcUMza1dIOXc1NTFNaGZvS3ZiM21VaUV6ZVMwOUNwZUQrTFh5ZnlqQzhZYkJxQjZXSFhNZWMrCnpPdDNPazRYV0FmZVVZTXhOQ1FJblc4cjI4cmZnblErc1NCdHQyeERQN1RZY09oNVZGZkI2K3JtTmFTblZ1NjgKSFFxdlFMNEFXbVhkR09jRWNBRThYdkdiOWhwSjVNckRHdzQ0UTYyOG9YazZ0N01aWTFOMUNQdW9HZ1VmS1N3bgo1MUFWRTFOVVdNV0tEQXhaa2I4bEhvR3VWaDFzWmd3SnJRQjR5clh1cmxGN0Y2bVRlYm4rcDVKM0toT0V4KzlsCjFXdkwwbWkxL1J2bVJKNm11YmtjWUwzN1FJWjI1YXdyaEZMN0Z1ejNRSTFqTTdYMHZET2VUM2VuVUFCZW5SMS8KUnlnPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg==
  tokenData: ZXlKaGJHY2lPaUpTVXpJMU5pSXNJbXRwWkNJNklrMHRSalJtZGpSdVgxcFljMGxsU1ZneFlXMHpPSFZOY0Zwbk1UTkhiVFpsVFZwQ2JIWk9SbU5XYW5NaWZRLmV5SnBjM01pT2lKcmRXSmxjbTVsZEdWekwzTmxjblpwWTJWaFkyTnZkVzUwSWl3aWEzVmlaWEp1WlhSbGN5NXBieTl6WlhKMmFXTmxZV05qYjNWdWRDOXVZVzFsYzNCaFkyVWlPaUprWldaaGRXeDBJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpXTnlaWFF1Ym1GdFpTSTZJbU5zZFhOMFpYSndaV1JwWVMxemVXNWphSEp2TFhSdmEyVnVMVGsxYTJSNElpd2lhM1ZpWlhKdVpYUmxjeTVwYnk5elpYSjJhV05sWVdOamIzVnVkQzl6WlhKMmFXTmxMV0ZqWTI5MWJuUXVibUZ0WlNJNkltTnNkWE4wWlhKd1pXUnBZUzF6ZVc1amFISnZJaXdpYTNWaVpYSnVaWFJsY3k1cGJ5OXpaWEoyYVdObFlXTmpiM1Z1ZEM5elpYSjJhV05sTFdGalkyOTFiblF1ZFdsa0lqb2lNREl5WXpNMk5USXRPR1k0WkMwME5qSmtMV0l6TnpFdFpUVXhPREF3TnpFeE9HUTBJaXdpYzNWaUlqb2ljM2x6ZEdWdE9uTmxjblpwWTJWaFkyTnZkVzUwT21SbFptRjFiSFE2WTJ4MWMzUmxjbkJsWkdsaExYTjVibU5vY204aWZRLkF4ZjhmbG5oR0lDYjJaMDdkT0FKUW11aHVIX0ZzRzZRSVY5Sm5sSmtPUnF5aGpWSDMyMkVqWDk1bVhoZ2RVQ2RfZXphRFJ1RFFpLTBOWDFseGc5OXpYRks1MC10ZzNfYlh5NFA1QnRFOUpRNnNraUt4dDFBZVJHVUF4bG5fVFU3SHozLTU5Vnl5Q3NwckFZczlsQWQwRFB6bTRqb1dyS1lKUXpPaGl5VjkzOWpaX2ZkS1BVUmNaMVVKVGpXUTlvNEFFY0hMdDlyTEJNMTk2eDRkbzA4ZHFaUnVtTzJZRXFkQTB3ZnRxZ2NGQzdtTGlSVVhkWElkYW9CY1BuWXBwM01MU3B5QjJQMV9vSlRFNS1nd3k4N2Jwb3U1RXo2TElSSExIeW5NWXAtWVRLR2hBbDJwMXdJb0tDZUNnQng4RlRfdzM4Rnh1TnE0UDRoQW5RUUh6bU9Ndw==
```
## View a cluster
After a cluster is successfully imported, you can use `kubectl get pediacluster` to view the imported cluster and check its status
```bash
kubectl get pediacluster
```
```
# Output:
NAME           APISERVER URL               VERSION    STATUS
cluster-1      https://10.6.100.10:6443    v1.22.2    Healthy
cluster-2      https://10.50.10.11:16443   v1.10.11   Healthy
```

## Next
See [Synchronize cluster resources](../sync-resources)
