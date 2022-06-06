---
title: "Configure Storage Layer"
weight: 1
---
`Default Storage Layer` of Clusterpedia supports two storage components: **MySQL** and **PostgreSQL**.

When installing Clusterpedia, you can use existing storage component and create [`Default Storage Layer`(ConfigMap)](#configure-the-default-storage-layer) and [`Secret of storage component`](#configure-secret).

## Configure the Default Storage Layer
You shall create `clusterpedia-internalstorage` ConfigMap in the `clusterpedia-system` namespace.
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
    connPool:
      maxIdleConns: 10
      maxOpenConns: 100
      connMaxLifetime: 1h
    log:
      slowThreshold: "100ms"
      logger:
        filename: /var/log/clusterpedia/internalstorage.log
        maxbackups: 3
```

`Default Storage Layer` config supports the following fields:
|field|description|
|-----|-----------|
|`type`|type of storage component such as "postgres" and "mysql" |
|`host`|host for storage component such as IP address or Service Name|
|`port`|port for storage component|
|`user`|user for storage component|
|`password`|password for storage component|
|`database`|the database used by Clusterpedia|

**It is a good choice to store the access password to Secret. For details see [Configure Secret of storage component](#configure-secret)**

### Connection Pool
|field|description|default value|
|-----|-----------|---------|
|`connPool.maxIdleConns`|the maximum number of connections in the idle connection pool.| 10 |
|`connPool.maxOpenConns`|the maximum number of open connections to the database.| 100 |
|`connPool.connMaxLifetime`|the maximum amount of time a connection may be reused. | 1h |

Set up the database connection pool according to the user's current environment.

### Configure log
Clusterpedia supports to configure logs for storage layer, enabling the log to record `slow SQL queries` and `errors` via the `log` field.
|field|description|
|-----|-----------|
|`log.stdout`|Output log to standard device|
|`log.colorful`|Enable color print or not|
|`log.slowThreshold`|Set threshold for slow SQL queries such as "100ms"|
|`log.level`|Set the severity level such as Slient, Error, Warn, Info|
|`log.logger`| configure rolling logger|

After enabling log, if `log.stdout` is not set to true, the log will be output to */var/log/clusterpedia/internalstorage.log*

#### Rolling logger
Write storage lay logs to file, and configure log file rotation
|field|description|
|-----|-----------|
|`log.logger.filename`|the file to write logs to, backup log files will be retained in the same directory, default is  */var/log/clusterpedia/internalstorage.log* |
|`log.logger.maxsize`|the maximum size in megabytes of the log file before it gets rotated. default is 100 MB.|
|`log.logger.maxage`|the maximum number of days to retain old log files based on the timestamp encoded in their filename.|
|`log.logger.maxbackups`|the maximum number of old log files to retain.|
|`log.logger.localtime`|whether it is local time, default is to use UTC time|
|`log.logger.compress`|compress determines if the rotated log files should be compressed using gzip.|

#### Disable log
If the `log` field is not filled in the internalstorage config, log will be ignored, for example:
```yaml
type: "mysql"
host: "clusterpedia-internalstorage-mysql"
port: 3306
user: root
database: "clusterpedia"
```

### More configuration
The default storage layer also provides more configurations about MySQL and PostgreSQL. Refer to [internalstorage/config.go](https://github.com/clusterpedia-io/clusterpedia/blob/main/pkg/storage/internalstorage/config.go).

## Configure Secret
The yaml file that is used to install Clusterpedia may get the password from `internalstroage-password` Secret.

Configure the storage component password to Secret
```bash
kubectl -n clusterpedia-system create secret generic \
    internalstorage-password --from-literal=password=<password to access storage components>
```
