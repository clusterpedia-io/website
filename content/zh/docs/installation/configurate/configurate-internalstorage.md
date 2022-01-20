---
title: "配置存储层"
weight: 1
---
Clusterpedia 的`默认存储层`支持 MySQL 和 PostgreSQL 两种存储组件。

用户在安装 Clusterpedia 时，可以使用已有存储组件，不过需要创建相应的`默认存储层`配置（ConfigMap） 和存储组件密码（Secret）

## 默认存储层配置
用户需要在 `clusterpedia-system` 命名空间下创建 `clusterpedia-internalstorage` ConfigMap。
```yaml
# internalstorage configmap example
apiVersion: v1
kind: ConfigMap
metadata:
  name: clusterpedia-internalstorage
  namespace: clusterpedia-system
data:
  internalstorage-config.yaml: |
    type: "mysql"
    host: "clusterpedia-internalstorage-mysql"
    port: 3306
    user: root
    database: "clusterpedia"
    log:
      slowThreshold: "100ms"
```

internalstorage config 支持以下基本字段:
|field|description|
|-----|-----------|
|`type`|存储组件的类型，支持 "postgres" 和 "mysql" |
|`host`|存储组件地址，可以使用 IP 或者 Service Name|
|`port`|存储组件端口|
|`user`|存储组件用户|
|`password`|存储组件密码|
|`database`|Clusterpedia 所使用的 database|

### 日志配置
支持配置存储层日志，通过 `log` 字段来开启日志打印慢 SQL 和错误
|field|description|
|-----|-----------|
|`log.stdout`|打印日志到标准输出|
|`log.colorful`|是否开启彩色打印|
|`log.slowThreshold`|设置慢 SQL 阀值，例如 "100ms"|
|`log.level`|设置日志级别，支持 Slient, Error, Warn, Info|

开启日志打印后，如果 `log.stdout` 不为 true，则将日志输出到 */var/log/clusterpedia/internalstorage.log* 文件中

#### 关闭日志打印
在 internalstorage config 不填写 `log` 字段，便会忽略日志打印，例如：
```yaml
type: "mysql"
host: "clusterpedia-internalstorage-mysql"
port: 3306
user: root
database: "clusterpedia"
```

### 更多配置
默认存储层还提供了更多的配置，可以参考 [internalstorage/config.go](https://github.com/clusterpedia-io/clusterpedia/blob/main/pkg/storage/internalstorage/config.go)

## 配置存储组件 Secret
Clusterpedia 的安装 yaml 会从 `internalstroage-password` 的 Secret 中获取密码。

将存储组件密码配置到 Secret 中
```bash
kubectl -n clusterpedia-system create secret generic internalstorage-password --from-literal=password=dangerous0
```
