---
title: "Raw SQL Query"
weight: 2
---

Different users may have different needs, and although Clusterpedia provides many easy search options, such as specifying a set of namespaces or clusters, or specifying an owner for a query, users may still need to use more complex queries to query the resources.

In this case, you can use the `Raw SQL Query` provided by the `default storage layer` to pass more complex search conditions by enabling either of the following Feature Gates:

| desc | Feature Gates | default |
|------|---------------|---------|
| Allow search conditions to be set using raw SQL. | `AllowRawSQLQuery` | `false` |

Note: **This Feature Gate is exclusive to the `clusterpedia apiserver`.**

{{% alert title="Disclaimer" color="warning" %}}

Either of `AllowRawSQLQuery` and [`AllowParameterizedSQLQuery`](./raw-parameterized-sql-query) Feature Gates will potentially introduce vulnerabilities to SQL injection when enabled since the final SQL statements is not possible to be wrapped, escaped, parameterized, or validated by Clusterpedia.

Such vulnerabilities can be exploited by malicious users to access unauthorized data or even causing data leaks, data loss and corruption.

It is always the responsibility and obligation of callers and end users interacting with Clusterpedia to defend incoming SQL WHERE clauses to protect data and resources from malicious access.

Clusterpedia cannot guarantee there will be no potential threats of SQL injections in the future and should never be when raw SQL involves.

If you still need to use the "Raw SQL Query" functionalities while understood the potential security issues, it is recommended to use the [`AllowParameterizedSQLQuery`](./raw-parameterized-sql-query) Feature Gate rather than the `AllowRawSQLQuery` Feature Gate. The introduced parameters of [`AllowParameterizedSQLQuery`](./raw-parameterized-sql-query) Feature Gate allow callers to pass **statements** and **parameters** separately, which can be better protected against SQL injection.

{{% /alert %}}

## `AllowRawSQLQuery` Feature Gate

{{% alert title="Warning" color="warning" %}}

Raw SQL queries are currently in alpha and are not well protected against SQL injection, so you need to enable this feature via Feature Gate.

Due to the introduction of the new [`AllowParameterizedSQLQuery` Feature Gate](./raw-parameterized-sql-query), it is not recommended to use this Feature Gate because this Feature Gate has serious flaws in defending against SQL injection now, and will be deprecated in future versions.

{{% /alert %}}

```bash
URL="/apis/clusterpedia.io/v1beta1/resources/apis/apps/v1/deployments"
kubectl get --raw="$URL?whereSQL=(cluster='global') OR (namespace IN ('kube-system','default'))"
```

In the example, we passed a *WHERE* clause for the query SQL statement

```sql
(cluster='global') OR (namespace IN ('kube-system','default'))
```

as the parameter of `whereSQL`.

Finally, when executing the query, the `WHERE` clause provided will be combined into the SQL statement and then executed to return the result:

```sql
SELECT
    object
FROM resources
WHERE
    `group` = "apps"
    AND `version` = "v1"
    AND `resource` = "deployments"
    -- Here is the SQL statement passed by whereSQL
    AND ((cluster = 'global') OR (namespace IN ('kube-system','default')))
```

This statement will retrieve `deployments` kind of resource under all namespaces in the *global* cluster and under the *kube-system* and *default* namespaces in other clusters.

Note: **The SQL statement needs to conform to the SQL syntax of the specific storage component(MySQL, PostgreSQL).**
