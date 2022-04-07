---
title: "kubectl apply"
weight: 10
---

## Install
The installation of Clusterpedia is divided into several parts:
- [Install storage component](#install-storage-component)
- [Install Clusterpedia](#install-clusterpedia)
- [Final check](#final-check)
> If you use existing storage component (MySQL or PostgreSQL), directly skip the step of installing the storage component.

Pull clusterpedia project:
```bash
git clone https://github.com/clusterpedia-io/clusterpedia.git
cd clusterpedia
git checkout v0.1.0
```

### Install storage component
Clusterpedia installation provides two storage components (**MySQL 8.0** and **PostgreSQL 12**) to choose.
> If you use existing storage components (MySQL or PostgreSQL), directly [skip](#install-clusterpedia) this step

{{< tabs >}}

{{% tab name="PostgreSQL" %}}
**Go to the installation directory of the selected storage component**
```bash
cd ./deploy/internalstorage/postgres
```
{{% /tab %}}

{{% tab name="MySQL" %}}
**Go to the installation directory of the selected storage component**
```bash
cd ./deploy/internalstorage/mysql
```
{{< /tab >}}

{{< /tabs >}}

**The storage component uses the Local PV method to store data, and you shall specify the node where the Local PV is located during deployment**
> You can choose to provide your own PV
```bash
export STORAGE_NODE_NAME=<nodename>
sed "s|__NODE_NAME__|$STORAGE_NODE_NAME|g" `grep __NODE_NAME__ -rl ./templates` > clusterpedia_internalstorage_pv.yaml
```

**Deploy storage component**
```bash
kubectl create -f .

# Go back to Clusterpedia root directory
cd ../../../
```

### Install Clusterpedia
Once the storage component are successfully deployed, you can install the Clusterpedia.

**If you uses existing storage component, refer to [Configure Storage Layer](../configuration/configure-internalstorage) to set the storage component into `Default Storage Layer`**

> Run the following cmd in the clusterpedia root directory
```bash
# Deploy crds
kubectl apply -f ./deploy/crds

# Deploy Clusterpedia components
kubectl apply -f ./deploy
```

### Final check
Check if the component Pods are running properly
```bash
kubectl -n clusterpedia-system get pods
```

## Uninstall
### Clean up PediaCluster
Before uninstalling Clusterpedia, you need to check if PediaCluster resources still exist in your environment, and clean up those resources.

```bash
kubectl get pediacluster
```

### Uninstall Clusterpedia
After the PediaCluster resource cleanup is complete, uninstall the Clusterpedia components.

```bash
kubectl delete -f ./deploy/clusterpedia_apiserver_apiservice.yaml
kubectl delete -f ./deploy/clusterpedia_apiserver_deployment.yaml
kubectl delete -f ./deploy/clusterpedia_clustersynchro_manager_deployment.yaml
kubectl delete -f ./deploy/clusterpedia_apiserver_rbac.yaml
kubectl delete -f ./deploy/crds
```

### Uninstall Storage Component
Remove related resources depending on the type of storage component selected.
```bash
kubectl delete -f ./deploy/internalstorage/<storage type>
```

#### remove Local PV and clean up data
After the storage component is uninstalled, the Local PV and corresponding data will still be left in the node and we need to clean it manually.

View the mounted nodes via Local PV resource details.
```bash
kubectl get pv clusterpedia-internalstorage-<storage type>
```

Once you know the node where the data is stored, you can delete the Local PV.
```bash
kubectl delete pv clusterpedia-internalstorage-<storage type>
```

Log in to the node where the data is located and clean up the data
```bash
# In the node where the legacy data is located
rm /var/local/clusterpedia/internalstorage/<storage type>
```
