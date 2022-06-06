---
title: "Raw SQL Query"
weight: 2
---
Different users may have different needs, and although clusterpedia provides many easy search options, such as specifying a set of namespaces or clusters, or specifying an owner for a query,
users may still have more complex queries.

In this case, you can use the `Raw SQL Query` provided by the `default storage layer` to pass more complex search conditions.
```bash
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQL=(cluster='global') OR (namespace IN ('kube-system','default'))"
```
In the example, we pass a SQL statement for a *WHERE* query —— **(cluster='global') OR (namespace IN ('kube-system','default'))**,

This statement will retrieve `deployments` under all namespaces in the *global* cluster and under the *kube-system* and *default* namespaces in other clusters.

**The sql statement needs to conform to the SQL syntax of the specific storage component(MySQL, PostgreSQL).**

This feature gate is exclusive to the `clusterpedia apiserver`
|desc|feature gates|默认值
|-----|-------------| ------ |
|Allow search conditions to be set using raw SQL|`AllowRawSQLQuery`|false|

Raw SQL queries are currently in alpha and are not well protected against SQL injection, so you need to enable this feature via Feature Gate.
