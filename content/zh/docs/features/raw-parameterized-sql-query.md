---
title: "原生参数化 SQL 查询"
weight: 2
---

不同的用户的需求可能是不同的，尽管 Clusterpedia 提供了很多简便的检索条件，例如指定一组命名空间或者资源名称，也可以指定 owner 进行查询，但是用户依然可能会有更加复杂的查询。

这时，用户可以通过开启以下任意一个 Feature Gates 使用默认存储层提供的 `原生 SQL 条件查询` 来传递更加复杂的检索条件：

| 作用 |  Feature Gate  | 默认值 |
|-----|---------------|-------|
| 允许使用参数化原生 SQL 进行查询（更易于防御 SQL 注入） | `AllowParameterizedSQLQuery` | `false` |

{{% alert title="免责声明" color="warning" %}}

由于最终 SQL 语句不可能被 Clusterpedia 封装、转义、参数化或验证，因此无论是启用 [`AllowRawSQLQuery`](./raw-sql-query) 还是 `AllowParameterizedSQLQuery`  Feature Gate 都将可能引入 SQL 注入漏洞。

恶意用户可以利用这些漏洞访问未经授权的数据，甚至造成数据泄露、数据丢失和损坏。

调用方和最终用户在与 Clusterpedia 交互的时候总是有责任和义务对传入的 SQL Where 子句进行防御以保护数据和资源，从而阻止恶意访问。

Clusterpedia 不能保证将来不会有 SQL 注入的潜在威胁，而且在涉及原始 SQL 时也不应该能够保证。

如果已经知晓上述提及的安全问题，并且还是希望使用「原生参数化 SQL 查询」的能力，建议使用 `AllowParameterizedSQLQuery`  Feature Gate 而不是 [`AllowRawSQLQuery`  Feature Gate](./raw-sql-query) 。这是因为 `AllowParameterizedSQLQuery` Feature Gate 引入的参数允许调用方分别传递**语句**和**参数**，这可以更易于防御 SQL 注入。

{{% /alert %}}

## `AllowParameterizedSQLQuery`  Feature Gate

启用后，将引入三个额外的查询参数：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| `whereSQLStatement` | `string` | SQL 语句的 `WHERE` 子句正文。可直接替换现有的 `whereSQL` 字段。通过使用 `?` 作为参数占位符（与 GORM 兼容），允许调用方更安全地发送原生 SQL 查询。 |
| `whereSQLParam` | `string`，`string[]` | 将在 SQL 语句的 `WHERE` 子句中使用的 SQL 语句的实际参数。此字段仅在设置了 `whereSQLStatement` 时有效，否则将被忽略。 |
| `whereSQLJSONParams` | `any[]` | 工作原理与 `whereSQLParam`相同，但在使用 `IN` 或 `JSON` 操作符时，调用方可以通过这个参数发送数组和更复杂的参数。支持任意的 JSON 数组字符串，并且由于查询字符串不能包含 `[]`（方括号），因此需要进行 `base64` 编码。该字段仅在设置了 `whereSQLStatement` 时有效，否则将被忽略。 |

### 使用案例

假设我们拥有以下资源

```shell
$ kubectl get all -A
NAMESPACE              NAME                                                      READY   STATUS    RESTARTS   AGE
clusterpedia-test-ns   pod/hello-node-in-clusterpedia-test-ns-6fbb8854b5-tvncw   1/1     Running   0          26s
default                pod/hello-node-7579565d66-mvmt8                           1/1     Running   0          4h9m

NAMESPACE              NAME                                         TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
clusterpedia-test-ns   service/hello-node-in-clusterpedia-test-ns   LoadBalancer   10.96.252.86    <pending>     8080:30323/TCP           6s
default                service/hello-node                           LoadBalancer   10.96.134.228   <pending>     8080:32712/TCP           4h8m

NAMESPACE              NAME                                                 READY   UP-TO-DATE   AVAILABLE   AGE
clusterpedia-test-ns   deployment.apps/hello-node-in-clusterpedia-test-ns   1/1     1            1           26s
default                deployment.apps/hello-node                           1/1     1            1           4h9m

NAMESPACE              NAME                                                            DESIRED   CURRENT   READY   AGE
clusterpedia-test-ns   replicaset.apps/hello-node-in-clusterpedia-test-ns-6fbb8854b5   1         1         1       26s
default                replicaset.apps/hello-node-7579565d66                           1         1         1       4h9m
```

#### `whereSQLStatement` 参数搭配单个 `whereSQLParam` 参数使用

查询资源时，调用方应使用 `?`（问号）来表示 SQL 语句中的参数，并在 `whereSQLParam` 字段中传递实际参数。

例如：

| 字段 | 值 |
| :--- | :--- |
| `whereSQLStatement` | `(namespace IN (?))` |
| `whereSQLParam` | `default` |

与 Clusterpedia apiserver 交互时，调用方将会像这样发送请求：

```shell
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQLStatement=(namespace IN (?))&whereSQLParam=default"
```

它最终将被发送到数据库，并通过以下 SQL 语句执行：

```sql
SELECT
    `object`
FROM `resources`
WHERE
    `group` = "apps"
    AND `resource` = "deployments"
    AND `version` = "v1"
    -- 以下是由 whereSQLStatement 和 whereSQLParam 传递的 SQL 语句
    AND (namespace IN ("default"))
```

这个语句会检索 `default` 命名空间下的 `deployments` 类型的资源。

注意：**SQL 语句需要符合具体的存储组件的 SQL 语法**

最终将获得数据库返回的以下资源：

| 资源名称 | Namespace |
| :--- | :--- |
| `hello-node` | `default` |

#### `whereSQLStatement` 参数搭配多个 `whereSQLParam` 参数使用

##### 案例 1

查询资源时，调用方应使用 `?`（问号）来表示 SQL 语句中的参数，并在 `whereSQLParam` 字段中传递实际参数。

例如：

| 字段 | 值 |
| :--- | :--- |
| `whereSQLStatement` | `(namespace IN (?, ?))` |
| `whereSQLParam` | `default` |
| `whereSQLParam` | `kube-system` |

{{% alert title="说明" color="primary" %}}

**之所以这里的 `whereSQLStatement` 中的查询包含两个 `?`（问号），是因为 `whereSQLParam` 默认会被解析为字符串数组，并使用 Go 中的展开语法将值传递给实际查询。**

因此，如果用 Go 的代码来说明的话，它将等同于

```go
db.Where("(namespace IN (?, ?))", "default", "kube-system")
```

而非一般情况下会被理解成的

```go
db.Where("(namespace IN (?, ?))", []string{"default", "kube-system"})
```

{{% /alert %}}

与 Clusterpedia apiserver 交互时，调用方将会像这样发送请求：

```shell
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQLStatement=(namespace IN (?, ?))&whereSQLParam=default&whereSQLParam=kube-system"
```

它最终将被发送到数据库，并通过以下 SQL 语句执行：

```sql
SELECT
    `object`
FROM `resources`
WHERE
    `group` = "apps"
    AND `resource` = "deployments"
    AND `version` = "v1"
    -- 以下是由 whereSQLStatement 和 whereSQLParam 传递的 SQL 语句
    AND (namespace IN ("default", "kube-system"))
```

这个语句会检索 `default` 和 `kube-system` 命名空间下的 `deployments` 类型的资源。

注意：**SQL 语句需要符合具体的存储组件的 SQL 语法**

最终将获得数据库返回的以下资源：

| 资源名称 | Namespace |
| :--- | :--- |
| `hello-node` | `default` |
| `coredns` | `kube-system` |

##### 案例 2

查询资源时，调用方应使用 `?`（问号）来表示 SQL 语句中的参数，并在 `whereSQLParam` 字段中传递实际参数。

例如：

| 字段 | 值 |
| :--- | :--- |
| `whereSQLStatement` | `(namespace = (?) OR namespace = (?))` |
| `whereSQLParam` | `default` |
| `whereSQLParam` | `kube-system` |

与 Clusterpedia apiserver 交互时，调用方将会像这样发送请求：

```shell
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQLStatement=(namespace = (?)  OR namespace = (?))&whereSQLParam=default&whereSQLParam=kube-system"
```

它最终将被发送到数据库，并通过以下 SQL 语句执行：

```sql
SELECT
    `object`
FROM `resources`
WHERE
    `group` = "apps"
    AND `resource` = "deployments"
    AND `version` = "v1"
    AND ((namespace = ("default") OR namespace = ("kube-system")))
```

这个语句会检索 `default` 和 `kube-system` 命名空间下的 `deployments` 类型的资源。

注意：**SQL 语句需要符合具体的存储组件的 SQL 语法**

最终将获得数据库返回的以下资源：

| 资源名称 | Namespace |
| :--- | :--- |
| `hello-node` | `default` |
| `coredns` | `kube-system` |

#### `whereSQLStatement` 参数搭配 `whereSQLJSONParams` 参数使用

您可能会注意到，在使用 `whereSQLParam` 时，用户必须明确写出有与变量/参数带有相同数量的 `?`（问号）作为占位符。

对于可能希望将任意长度的数组视为单个变量/参数并用单个 `?`（问号）表示的用户来说，这样做不够灵活。这时，`whereSQLJSONParams` 参数可能会对此类用例和场景有所帮助。

例如：

| field | value |
| :--- | :--- |
| `whereSQLStatement` | `(namespace IN (?))` |
| `whereSQLJSONParams` | `[ [ "default", "kube-system" ] ]` |

在这个例子中，需要给 `IN` 操作符传递一个数组，而 `whereSQLParam` 无法满足这个需求，因为它只能传递一个字符串数组。因此此处使用 `whereSQLJSONParams` 参数传递一个完整的嵌套数组来实现。

由于查询字符串不能包含 `[]`（方括号），因此在发送请求之前，需要将数组转换为有效的 JSON 数组字符串，然后进行 `base64` 编码。

也就是

```
W1siZGVmYXVsdCIsImt1YmUtc3lzdGVtIl1d
```

与 Clusterpedia apiserver 交互时，调用方将会像这样发送请求：

```shell
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQLStatement=(namespace IN (?))&whereSQLJSONParams=W1siZGVmYXVsdCIsImt1YmUtc3lzdGVtIl1d"
```

它最终将被发送到数据库，并通过以下 SQL 语句执行：

```sql
SELECT
    `object`
FROM `resources`
WHERE
    `group` = "apps"
    AND `resource` = "deployments"
    AND `version` = "v1"
    AND (namespace IN ("default", "kube-system"))
```

这个语句会检索 `default` 和 `kube-system` 命名空间下的 `deployments` 类型的资源。

注意：**SQL 语句需要符合具体的存储组件的 SQL 语法**

最终将获得数据库返回的以下资源：

| 资源名称 | Namespace |
| :--- | :--- |
| `hello-node` | `default` |

### 启用 `AllowParameterizedSQLQuery` Feature Gate 时，如何防御 SQL 注入？

最常见的 SQL 注入是使用 `OR` 操作符绕过原始 SQL 语句，返回数据库中的所有资源。

假设启用了 [`AllowRawSQLQuery` Feature Gate](./raw-sql-query)，调用方将会像这样发送请求：

```shell
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQL=namespace = \"\" OR 1 = 1"
```

它最终将被发送到数据库，并通过以下 SQL 语句执行：

```sql
SELECT
    `object`
FROM `resources`
WHERE
    `group` = "apps"
    AND `resource` = "deployments"
    AND `version` = "v1"
    -- 硬拼接的 SQL 语句
    AND namespace = "" OR 1 = 1
```

问题在于，**语句为 `1 = 1` 的 `OR` 操作符将始终为真**，这将绕过原始的 `WHERE` 子句，返回数据库中的所有资源。

这就很难对 SQL 语句进行包装、转义和验证，因为在某些极端情况下，比如支持模糊搜索任意资源时，调用方可能需要将 SQL 语句不加任何修改地直接传递给 apiserver。

不过，通过启用 `AllowParameterizedSQLQuery`  Feature Gate ，在使用 `whereSQLStatement` 参数的时候搭配 `whereSQLParam` 参数就可以轻松避免此类 SQL 注入。

依然以相同的 SQL 注入示例为例：

| field | value |
| :--- | :--- |
| `whereSQLStatement` | `namespace = (?)` |
| `whereSQLParam` | `"" OR 1 = 1` |

与 Clusterpedia apiserver 交互时，调用方将会像这样发送请求：

```shell
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQLStatement=namespace = (?)&whereSQLParam=\"\" OR 1 = 1"
```

它最终将被发送到数据库，并通过以下 SQL 语句执行：

```sql
SELECT
    `object`
FROM `resources`
WHERE
    `group` = "apps"
    AND `resource` = "deployments"
    AND `version` = "v1"
    -- 以下是由 whereSQLStatement 和 whereSQLParam 传递的 SQL 语句
    AND namespace = "\"\" OR 1 = 1"
```

这个语句会检索 `"" OR 1 = 1` 命名空间下的 `deployments` 类型的资源。

最终将获得数据库**返回 0 个资源，而不是所有资源**。

这是因为在使用 `whereSQLParam` 参数或是 `whereSQLJSONParams` 参数时，传入的值将在数据库执行前被解析为字符串，因此 `"" OR 1 = 1` 将被视为字符串字面形式，而不是 SQL 语句的一部分，从而避免了 SQL 注入。

## 安全注意事项

1. 虽然 `AllowParameterizedSQLQuery` Feature Gate 提供了更好的 SQL 注入防御，但如果调用者的实现在使用 `whereSQLStatement` 时仍使用字符串连接来直接构建带参数的原始 SQL 查询，而不使用 `whereSQLParam` 或 `whereSQLJSONParams` 参数，那么 SQL 注入的隐患将**永远**无法解决。
2. 在涉及到模糊搜索、字符串拼接的使用场景的时候，建议更新所有现有的原生 SQL 查询，将原始 SQL 的参数传递到 `whereSQLParam` 或 `whereSQLJSONParams` 字段，而不是硬拼接为 `whereSQL` 字段的一部分。
3. 有关 GORM 的 SQL 注入的更多信息，请在此处阅读更多信息：[https://gorm.io/docs/security.html](https://gorm.io/docs/security.html)。
