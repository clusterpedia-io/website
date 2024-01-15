---
title: 特性功能
weight: 50
---

用户在使用特性功能时，需要开启相应的特性门控。
例如开启 clustersynchro manager 的 `SyncAllResources` 来允许使用 `全资源通配符`
```bash
# 忽略其他参数
./bin/clustersynchro-manager --feature-gates=SyncAllResources=true
```

Clusterpedia APIServer 和 Clusterpedia ClusterSynchro Manager 分别具有不同的特性门控

## APIServer

| 作用 | feature gate | 默认值 |
|---|--------|----|
| [设置默认返回剩余的资源数量](./remaining-item-count) | `RemainingItemCount` | `false` |
| [原生 SQL 查询](./raw-sql-query) | `AllowRawSQLQuery` | `false` |
| [Raw Parameterized SQL Query](./raw-sql-query) | `AllowRawSQLQueryWithParameter` | `false` |

## ClusterSynchro Manager

| 作用 | feature gate | 默认值 |
|---|--------|----|
| 裁剪 `metadata.managedFields` 字段 | `PruneManagedFields` | `true` |
| 裁剪 `metadata.annotations['lastAppliedConfiguration']` 字段 | `PruneLastAppliedConfiguration` | `true` |
| 允许同步所有类型的自定义资源 | `AllowSyncCustomResources` | `false` |
| 允许同步所有类型的资源 | `AllowSyncAllResources` | `false` |
| 集群健康检查使用独立 TCP 连接 | `HealthCheckerWithStandaloneTCP` | `false` |
