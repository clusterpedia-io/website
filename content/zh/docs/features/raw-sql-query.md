---
title: "原生 SQL 查询"
weight: 2
---

不同的用户的需求可能是不同的，尽管 Clusterpedia 提供了很多简便的检索条件，例如指定一组命名空间或者资源名称，也可以指定 owner 进行查询，但是用户依然可能会有更加复杂的查询。

这时，用户可以通过开启以下任意一个 Feature Gates 使用默认存储层提供的 `原生 SQL 条件查询` 来传递更加复杂的检索条件：

| 作用 |  Feature Gate  | 默认值 |
|-----|---------------|-------|
| 允许使用原生 SQL 设置检索条件 | `AllowRawSQLQuery` | `false` |

注意：**上述 Feature Gates 专属于 `clusterpedia apiserver`。**

{{% alert title="免责声明" color="warning" %}}

由于最终 SQL 语句不可能被 Clusterpedia 封装、转义、参数化或验证，因此无论是启用 `AllowRawSQLQuery` 还是 [`AllowParameterizedSQLQuery`](./raw-parameterized-sql-query)  Feature Gate 都将可能引入 SQL 注入漏洞。

恶意用户可以利用这些漏洞访问未经授权的数据，甚至造成数据泄露、数据丢失和损坏。

调用方和最终用户在与 Clusterpedia 交互的时候总是有责任和义务对传入的 SQL Where 子句进行防御以保护数据和资源，从而阻止恶意访问。

Clusterpedia 不能保证将来不会有 SQL 注入的潜在威胁，而且在涉及原始 SQL 时也不应该能够保证。

如果已经知晓上述提及的安全问题，并且还是希望使用「原生 SQL 查询」的能力，建议使用 [`AllowParameterizedSQLQuery`  Feature Gate](./raw-parameterized-sql-query) 而不是 `AllowRawSQLQuery`  Feature Gate 。这是因为 [`AllowParameterizedSQLQuery` Feature Gate](./raw-parameterized-sql-query) 引入的参数允许调用方分别传递**语句**和**参数**，这可以更易于防御 SQL 注入。

{{% /alert %}}

## `AllowRawSQLQuery`  Feature Gate

{{% alert title="警告" color="warning" %}}

原生 SQL 查询当前还在 alpha 阶段，并且对 SQL 注入没有很好的防范，所以需要用户通过 Feature Gates 来开启该功能。

并且由于后来 [`AllowParameterizedSQLQuery`  Feature Gate](./raw-parameterized-sql-query) 的引入，该 Feature Gate 由于在防御 SQL 注入方面有很严重的缺陷，会在将来的版本中被废弃，不再建议使用这个 Feature Gate 。

{{% /alert %}}

假设与 Clusterpedia apiserver 交互时，调用方将会像这样发送请求：

```bash
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQL=(cluster='global') OR (namespace IN ('kube-system','default'))"
```

在这个示例中，我们传递了一个用于查询 SQL 语句的 *WHERE* 子句：

```sql
(cluster='global') OR (namespace IN ('kube-system','default'))
```

作为 `whereSQL` 的参数。

最终在执行查询的时候，会将提供的 WHERE 子句组合到 SQL 语句中，然后执行查询，返回结果：

```sql
SELECT
    object
FROM resources
WHERE
    ('group' = 'apps' AND 'version' = 'v1' AND 'resource' = 'deployments')
    -- 这里是 whereSQL 传递的 SQL 语句
    AND ((cluster = 'global') OR (namespace IN ('kube-system','default')))
```

这个语句会检索 *`global`* 集群内所有命名空间下以及其他集群中 *`kube-system`* 和 *`default`* 命名空间下的 `deployments` 类型的资源。

注意：**SQL 语句需要符合具体的存储组件的 SQL 语法**
