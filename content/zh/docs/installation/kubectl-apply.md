---
title: "使用 kubectl apply"
weight: 10
---

Clusterpedia 的安装分为两个部分：
* [安装存储组件](#安装存储组件)
* [安装 Clusterpedia](#安装-clusterpedia)
> 用户如果使用已有的存储组件（MySQL 或者 PostgreSQL），则直接跳过安装存储组件

拉取项目：
```bash
git clone https://github.com/clusterpedia-io/clusterpedia.git
cd clusterpedia
```

## 安装存储组件
Clusterpedia 安装时提供了 MySQL 8.0 和 PostgreSQL 12 两种存储组件以供选择
> 用户如果使用已有的存储组件（MySQL 或者 PostgreSQL），则直接跳过存储组件安装

**进入所选存储组件的安装目录**，这里选择使用 PostgreSQL
```bash
cd ./deploy/internalstorage/postgres
```

**存储组件使用 local pv 的方式存储数据，部署时，需要指定 local pv 所在节点**
```bash
export STORAGE_NODE_NAME=<节点名称>
sed "s|__NODE_NAME__|$STORAGE_NODE_NAME|g" `grep __NODE_NAME__ -rl ./templates` > clusterpedia_internalstorage_pv.yaml
```

**部署存储组件**
```bash
kubectl create -f .

# 跳回 Clusterpedia 项目根目录
cd ../../
```

## 安装 Clusterpedia
存储组件部署完成后，便可安装 Clusterpedia。

**如果选择使用已存在的存储组件，则需要参考 [配置存储层](../configurate/configurate-internalstorage) 来将存储组件对接到默认存储层中**

> 在 clusterpedia 项目根目录下进行操作
```bash
# 部署 crds
kubectl apply -f ./deploy/crds ./deploy

# 部署 Clusterpedia 组件
kubectl apply -f ./deploy
```

## 安装完成
检查组件 Pods 运行是否正常
```bash
kubectl -n clusterpedia-system get pods
```
