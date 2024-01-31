---
title: "Raw Parameterized SQL Query"
weight: 2
---

Different users may have different needs, and although Clusterpedia provides many easy search options, such as specifying a set of namespaces or clusters, or specifying an owner for a query, users may still need to use more complex queries to query the resources.

In this case, you can use the `Raw Parameterized SQL Query` provided by the `default storage layer` to pass more complex search conditions by enabling either of the following Feature Gates:

| desc | Feature Gates | default |
|------|---------------|---------|
| Allow querying by the parameterized SQL (for better defense against SQL injection). | `AllowParameterizedSQLQuery` | `false` |

Note: **This Feature Gate is exclusive to the `clusterpedia apiserver`.**

{{% alert title="Disclaimer" color="warning" %}}

Either of  [`AllowRawSQLQuery`](./raw-sql-query) and `AllowParameterizedSQLQuery` Feature Gates will potentially introduce vulnerabilities to SQL injection when enabled since the final SQL statements is not possible to be wrapped, escaped, parameterized, or validated by Clusterpedia.

Such vulnerabilities can be exploited by malicious users to access unauthorized data or even causing data leaks, data loss and corruption.

It is always the responsibility and obligation of callers and end users interacting with Clusterpedia to defend incoming SQL WHERE clauses to protect data and resources from malicious access.

Clusterpedia cannot guarantee there will be no potential threats of SQL injections in the future and should never be when raw SQL involves.

If you still need to use the "Raw Parameterized SQL Query" functionalities while understood the potential security issues, it is recommended to use the `AllowParameterizedSQLQuery` Feature Gate rather than the [`AllowRawSQLQuery` Feature Gate](./raw-sql-query). The introduced parameters of `AllowParameterizedSQLQuery` Feature Gate allow callers to pass **statements** and **parameters** separately, which can be better protected against SQL injection.

{{% /alert %}}

# `AllowParameterizedSQLQuery` Feature Gate

Once enabled, three additional query parameters will be introduced:

| field | type | description |
| :--- | :--- | :--- |
| `whereSQLStatement` | `string` | The `WHERE` clause body of the SQL statement. A drop-in replacement of the existing `whereSQL` field. Callers are allowed to send raw queries more safely by using `?` as the placeholder of parameters (which is compatible for GORM). |
| `whereSQLParam` | `string`，`string[]` | The actual parameter of the SQL statement that will be used in the `WHERE` clause of the SQL statement. This field is only valid when `whereSQLStatement` is set and will be ignored otherwise. |
| `whereSQLJSONParams` | `string` | Works the same as `whereSQLParam` but enables callers to send arrays and more complex parameters when `IN` or `JSON` operators is used. Supports any marshaled valid JSON array string. Needs to be `base64` encoded since query strings cannot contain `[]` (square brackets). This field is only valid when `whereSQLStatement` is set and will be ignored otherwise. |

Assumes we have the following resources:

## Use cases

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

### Use of `whereSQLStatement` and single `whereSQLParam`

When querying the resources, callers should use `?` (question mark) to represent the parameters in the SQL statement, and pass the actual parameters in the `whereSQLParam` field.

For example:

| field | value |
| :--- | :--- |
| `whereSQLStatement` | `(namespace IN (?))` |
| `whereSQLParam` | `default` |

When interacting with Clusterpedia apiserver, the caller will send the following request:

```shell
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQLStatement=(namespace IN (?))&whereSQLParam=default"
```

therefore it will eventually be sent to database and got executed with the following SQL statement:

```sql
SELECT
    `object`
FROM `resources`
WHERE
    `group` = "apps"
    AND `resource` = "deployments"
    AND `version` = "v1"
    -- Here is the SQL statement passed by whereSQLStatement and whereSQLParam
    AND (namespace IN ("default"))
```

This statement will retrieve `deployments` kind of resource under the `default` namespace.

Note the `default` value will be interpreted as a string literal in the SQL statement by databases.

Note: **The SQL statement needs to conform to the SQL syntax of the specific storage component(MySQL, PostgreSQL).**

You will get the following resources that returned by database:

| name | namespace |
| :--- | :--- |
| `hello-node` | `default` |

### Use of `whereSQLStatement` and multiple `whereSQLParam`

#### Case 1

When querying the resources, callers should use `?` (question mark) to represent the parameters in the SQL statement, and pass the actual parameters in the `whereSQLParam` field.

For example:

| field | value |
| :--- | :--- |
| `whereSQLStatement` | `(namespace IN (?, ?))` |
| `whereSQLParam` | `default` |
| `whereSQLParam` | `kube-system` |

{{% alert title="Note" color="primary" %}}

**The reason why the query in `whereSQLStatement` consists two `?` (question mark) is because `whereSQLParam` will be parsed as array of strings by default and use the spread syntax in Go to pass the values in to the actual query.**

Therefore it will be equivalent to

```go
db.Where("(namespace IN (?, ?))", "default", "kube-system")
```

instead of
```go
db.Where("(namespace IN (?, ?))", []string{"default", "kube-system"})
```

that you may think of.

{{% /alert %}}

When interacting with Clusterpedia apiserver, the caller will send the following request:

```shell
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQLStatement=(namespace IN (?, ?))&whereSQLParam=default&whereSQLParam=kube-system"
```

therefore it will eventually be sent to database and executed like this:

```sql
SELECT
    `object`
FROM `resources`
WHERE
    `group` = "apps"
    AND `resource` = "deployments"
    AND `version` = "v1"
    -- Here is the SQL statement passed by whereSQLStatement and whereSQLParam
    AND (namespace IN ("default", "kube-system"))
```

This statement will retrieve `deployments` kind of resource under the `default` and `kube-system` namespaces.

Note the `default` and `kube-system` values will be interpreted as a string literal in the SQL statement by databases.

Note: **The SQL statement needs to conform to the SQL syntax of the specific storage component(MySQL, PostgreSQL).**

You will get the following resources that returned by database:

| name | namespace |
| :--- | :--- |
| `hello-node` | `default` |
| `coredns` | `kube-system` |

#### Case 2

When querying the resources, callers should use `?` (question mark) to represent the parameters in the SQL statement, and pass the actual parameters in the `whereSQLParam` field.

For example:

| field | value |
| :--- | :--- |
| `whereSQLStatement` | `(namespace = (?) OR namespace = (?))` |
| `whereSQLParam` | `default` |
| `whereSQLParam` | `kube-system` |

When interacting with Clusterpedia apiserver, the caller will send the following request:

```shell
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQLStatement=(namespace = (?)  OR namespace = (?))&whereSQLParam=default&whereSQLParam=kube-system"
```

therefore it will eventually be sent to database and executed like this:

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

This statement will retrieve `deployments` kind of resource under the `default` and `kube-system` namespaces.

Note the `default` and `kube-system` values will be interpreted as a string literal in the SQL statement by databases.

Note: **The SQL statement needs to conform to the SQL syntax of the specific storage component(MySQL, PostgreSQL).**

You will get the following resources that returned by database:

| name | namespace |
| :--- | :--- |
| `hello-node` | `default` |
| `coredns` | `kube-system` |

### Use of `whereSQLStatement` and `whereSQLJSONParams`

As you may notice, users must explicitly write down how many variables/parameters have with the same number of `?` (question mark) when using  `whereSQLParam`.

This is not flexible enough for users who may want to treat any length of arrays as a single variable/parameter and represent it with a single `?` (question mark). This is where `whereSQLJSONParams` might be helpful for such use case and scenario.

For example:

| field | value |
| :--- | :--- |
| `whereSQLStatement` | `(namespace IN (?))` |
| `whereSQLJSONParams` | `[ [ "default", "kube-system" ] ]` |

In this example, a valid array is required for `IN` operator, and `whereSQLParams` cannot be used since it will be parsed as two parameters instead of one array. Therefore, `whereSQLJSONParams` is introduced to solve this problem.

Before sending the request, the array needs to be marshaled into a valid JSON array string and then `base64` encoded since query strings cannot contain `[]` (square brackets).

Which is

```
W1siZGVmYXVsdCIsImt1YmUtc3lzdGVtIl1d
```

When interacting with Clusterpedia apiserver, the caller will send the following request:

```shell
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQLStatement=(namespace IN (?))&whereSQLJSONParams=W1siZGVmYXVsdCIsImt1YmUtc3lzdGVtIl1d"
```

therefore it will eventually be sent to database and executed like this:

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

This statement will retrieve `deployments` kind of resource under the `default` and `kube-system` namespaces.

Note the `default` and `kube-system` values will be interpreted as a string literal in the SQL statement by databases.

Note: **The SQL statement needs to conform to the SQL syntax of the specific storage component(MySQL, PostgreSQL).**

You will get the following resources that returned by database:

| name | namespace |
| :--- | :--- |
| `hello-node` | `default` |

## How to defense against SQL injection when `AllowParameterizedSQLQuery` Feature Gate is enabled?

The most common SQL injection is to use the `OR` operator to bypass the original SQL statement and return all the resources in the database.

Let's say when [`AllowRawSQLQuery` Feature Gate](./raw-sql-query) is enabled, the caller sends the following request:

```shell
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQL=namespace = \"\" OR 1 = 1"
```

therefore it will eventually be sent to database and executed like this:

```sql
SELECT
    `object`
FROM `resources`
WHERE
    `group` = "apps"
    AND `resource` = "deployments"
    AND `version` = "v1"
    -- Hard string concatenation
    AND namespace = "" OR 1 = 1
```

The problem is that **the `OR` operator with statement of `1 = 1` will always be true**, which will bypass the original `WHERE` clause and return all the resources in the database.

And this is quite hard to wrap, escape, and validate the SQL statement since in some extreme cases, such as resource search by users, the caller may need to pass the SQL statement directly to the apiserver without any modification.

However, such SQL injection can be easily avoided by using `whereSQLStatement` and `whereSQLParam` by enabling `AllowParameterizedSQLQuery` Feature Gate instead.

For example, with the same constructed payload:

| field | value |
| :--- | :--- |
| `whereSQLStatement` | `namespace = (?)` |
| `whereSQLParam` | `"" OR 1 = 1` |

When interacting with Clusterpedia apiserver, the caller will send the following request:

```shell
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQLStatement=namespace = (?)&whereSQLParam=\"\" OR 1 = 1"
```

therefore it will eventually be sent to database and executed like this:

```sql
SELECT
    `object`
FROM `resources`
WHERE
    `group` = "apps"
    AND `resource` = "deployments"
    AND `version` = "v1"
    -- Here is the SQL statement passed by whereSQLStatement and whereSQLParam
    AND namespace = "\"\" OR 1 = 1"
```

you will **get none (0) resources that returned** by database instead of all the resources in the database.

This is because when using either the `whereSQLParam` or the `whereSQLJSONParams` parameter, the values passed in are parsed as strings before it got executed inside of the database, therefore `"" OR 1 = 1` will be treated as string literals rather than as part of the SQL statement, thus avoiding SQL injection.

## Security Considerations

1. While `AllowParameterizedSQLQuery` Feature Gate offers better defense against SQL injection, the potentials of SQL injection will **NEVER** be resolved if implementations of caller still use string concatenation to build the raw SQL query with parameters directly without `whereSQLParam` or `whereSQLJSONParams` when using `whereSQLStatement`.
2. It is recommended to update all the existing SQLs of raw queries to pass the parameters of raw SQL in `whereSQLParam` or `whereSQLJSONParams` field instead of `whereSQL` field when string concatenation is used to construct SQL statements or fuzzy search is implemented.
3. For more about the SQL injection of GORM, please read it more here: [https://gorm.io/docs/security.html](https://gorm.io/docs/security.html).
