---
title: "Configure storage layers"
weight: 1
---
`Default storage layer` of Clusterpedia supports two storage components: **MySQL** and **PostgreSQL**.

When installing Clusterpedia, you can use existing storage components and create [`default storage layer` (ConfigMap)](#configuration of default storage layer) and [`Secret of storage component`](#Configure the secret for storage component).

## Configure the default storage layer
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
    log:
      slowThreshold: "100ms"
```

internalstorage config supports the following fields:
|field|description|
|-----|-----------|
|`type`|type of storage components such as "postgres" and "mysql" |
|`host`|host for storage components such as IP address or Service Name|
|`port`|port for storage components|
|`user`|user for storage components|
|`password`|password for storage components|
|`database`|the database used by Clusterpedia|

**It is a good choice to store the access password to Secret. For details see [Configure Secret of storage components](#Configure password-secret)**

### Configure log
Clusterpedia supports to configure logs for storage layers, enabling the log to record slow SQL queries and errors via the `log` field.
|field|description|
|-----|-----------|
|`log.stdout`|Output log to standard device|
|`log.colorful`|Enable color print or not|
|`log.slowThreshold`|Set threshold for slow SQL queries such as "100ms"|
|`log.level`|Set the severity level such as Slient, Error, Warn, Info|

After enabling log, if `log.stdout` is not set to true, the log will be output to */var/log/clusterpedia/internalstorage.log*

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
The default storage layer also provides more configurations. Refer to [internalstorage/config.go](https://github.com/clusterpedia-io/clusterpedia/blob/main/pkg/storage/internalstorage/config.go).

## Configure Secret
The yaml file that is used to install Clusterpedia may get the password from `internalstroage-password` Secret.

Configure the storage components password to Secret
```bash
kubectl -n clusterpedia-system create secret generic \
    internalstorage-password --from-literal=password=<password to access storage components>
```
