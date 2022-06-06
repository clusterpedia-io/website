---
title: "原生 SQL 查询"
weight: 2
---
不同的用户的需求可能是不同的，尽管 clusterpedia 提供了很多简便的检索条件，例如指定一组命名空间或者资源名称，也可以指定 owner 进行查询，但是用户依然可能会有更加复杂的查询。

这时，用户可以使用默认存储层提供的 `原生 SQL 条件查询` 来传递更加复杂的检索条件
```bash
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQL=(cluster='global') OR (namespace IN ('kube-system','default'))"
```
示例中，我们传递一个用于 *WHERE* 查询的 SQL 语句 —— **(cluster='global') OR (namespace IN ('kube-system','default'))**,

这个语句会检索 *global* 集群内所有命名空间下以及其他集群中 *kube-system* 和 *default* 命名空间下的 `deployments`。

**sql 语句需要符合具体的存储组件的 SQL 语法**

该特性门控专属于 `clusterpedia apiserver`
|作用|feature gates|默认值
|-----|-------------| ------ |
|允许使用原生 SQL 设置检索条件|`AllowRawSQLQuery`|false|

原生 SQL 查询当前还在 alpha 阶段，并且对 SQL 注入没有很好的防范，所以需要用户通过 Feature Gates 来开启该功能
