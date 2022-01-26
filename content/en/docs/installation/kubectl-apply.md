---
title: "By kubectl apply"
weight: 10
---

The installation of Clusterpedia is divided into several parts:
- [Install storage components](#install-storage-components)
- [Install Clusterpedia](#install-clusterpedia)
- [Final check](#final-check)
> If you use existing storage components (MySQL or PostgreSQL), directly skip the step of installing the storage component

Pull items:
```bash
git clone https://github.com/clusterpedia-io/clusterpedia.git
cd clusterpedia
```

## Install storage components
Clusterpedia installation provides two storage components (**MySQL 8.0** and **PostgreSQL 12**) to choose.
> If you use existing storage components (MySQL or PostgreSQL), directly [skip] this step

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

**Deploy storage components**
```bash
kubectl create -f .

# Go back to Clusterpedia root directory
cd ../../
```

## Install Clusterpedia
Once the storage components are successfully deployed, you can install the Clusterpedia .

**If you uses existing storage components, refer to [Configure storage layer](../configurate/configurate-internalstorage) to set the storage components into `default storage layer`**

> Run the following code in the clusterpedia root directory
```bash
# Deploy crds
kubectl apply -f ./deploy/crds

# Deploy Clusterpedia components
kubectl apply -f ./deploy
```

## Final check
Check if the component Pods are running properly
```bash
kubectl -n clusterpedia-system get pods
```
