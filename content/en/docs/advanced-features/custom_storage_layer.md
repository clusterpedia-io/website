---
title: Custom Storage Layer Plugin
weight: 15
---

Clusterpedia can use different storage components such as MySQL/PostgreSQL, Memory, Elasticsearch through the storage layer.

Currently, Clusterpedia has two built-in storage layers:
* The [internalstorage](https://clusterpedia.io/en/docs/installation/configurate/configurate-internalstorage/) storage layer for accessing relational databases.
* The [memory](https://github.com/clusterpedia-io/clusterpedia/tree/main/pkg/storage/memorystorage) storage layer based on Memory 

Although Clusterpedia already supports relational databases and memory by default, user requirements are often variable and complex, and a fixed storage layer may not match the requirements of different users for storage components and performance, so Clusterpedia supports access to user-implemented storage layers by means of plugins, which we call `custom storage layer plugins`, or `storage plugins` for short.

With the `storage plugin`, users can do the following things:
* **Use any storage component**, such as `Elasticsearch`, `RedisGraph`, `Etcd`, or even `MessageQueue` with no problem
* Allow users to optimize the storage format and query performance of resources for their business
* Implement more advanced retrieval features for storage components

Clusterpedia also maintains a number of `storage plugins` that users can choose from, depending on your needs:
* [Sample Storage](https://github.com/clusterpedia-io/sample-storage): Example of a storage plugin that can connect to **relational databases**
* [Elasticsearch Storage](https://github.com/clusterpedia-io/elasticsearch-storage): Storage plugin for connecting to **Elasticsearch**

`Storage plugins` are loaded by Clusterpedia components via **Go Plugin**, which provides very flexible plug-in access without any performance loss compared to RPC or other methods.
> The performance impact of the Go Plugin can be found at https://github.com/uberswe/goplugins

As we all know, Go Plugin is troublesome to develop and use, but Clusterpedia cleverly optimizes the use and development of storage plugins through some mechanisms, and provides clusterpedia-io/sample-storage plugin as a reference.

Here we take [clusterpedia-io/sample-storage](https://github.com/clusterpedia-io/sample-storage) as an example to introduce.

* [How to use the `custom storage layer plugin`](#use-the-custom-storage-layer-plugin)
* [How to implement the `custom storage layer plugin`](#developing-custom-storage-layer-plugins)

## Use the `custom storage layer plugin`
The use of the storage plugin can be broadly divided into three ways:
1. Run the Clusterpedia component binary and load the storage plugins
2. Use the base Chart -- clusterpedia-core to set up the `storage plugin image` and configure the storage layer
3. Use the Clusterpedia **Advanced Chart** to not care about the storage plugin settings

By running the component binary locally, we can get a better understanding of how the Cluserpedia component loads and runs the `storage plugins`.

Users can actually use the `storage plugin image` already built, or deploy Clusterpedia **Advanced Chart** directly
### Local Run
#### Building Plugins
A storage plugin is actually a dynamic link library with a *.so* suffix.
Clusterpedia components can load `storage plugins` at startup and use specific `storage plugins` depending on the specified storage layer name.

Let's take [clusterpedia-io/sample-storage](https://github.com/clusterpedia-io/sample-storage) as an example and build a `storage plugin binary`
```bash
$ git clone --recursive https://github.com/clusterpedia-io/sample-storage.git && cd sample-storage
$ make build-plugin
```

Use the **file** command to view storage plugin information
```
$ file ./plugins/sample-storage-layer.so
./plugins/sample-storage-layer.so: Mach-O 64-bit dynamically linked shared library x86_64
```

Clusterpedia's `ClusterSynchro Manager` and `APIServer` components can load and use storage plugins via environment variables and command flags:
* `STORAGE_PLUGINS=<plugins dir>` environment variable sets the directory where the plugins are located, and Clusterpedia will load all plugins in that directory into the component
* `--storage-name=<storage name>` command flag, set the storage layer name
* `--storage-config=<storage config path>` command flag, set the storage layer configuration

#### Building components
To ensure consistent dependencies when running locally, clusterpedia components need to be built locally with the `make build-components` command
> For more information on building storage plugins and Clusterpedia components see [Developing custom storage layer plugins](#local-development-run)

```bash
$ # cd sample-storage
$ make build-components
$ ls -al ./bin
-rwxr-xr-x   1 icebergu  staff  90707488 11  7 11:15 apiserver
-rwxr-xr-x   1 icebergu  staff  91896016 11  7 11:16 binding-apiserver
-rwxr-xr-x   1 icebergu  staff  82769728 11  7 11:16 clustersynchro-manager
-rwxr-xr-x   1 icebergu  staff  45682000 11  7 11:17 controller-manager
```

#### Storage plugin runtime configuration file
Before running clusterpedia you also need to prepare the runtime configuration file for the storage plugin.
*sample-storage* provides an example configuration [example-config.yaml](https://github.com/clusterpedia-io/sample-storage/blob/main/example-config.yaml)

When running the clusterpedia component, set the configuration file via `--storage-config=./config.yaml` to specify the runtime configuration file
```yaml
# example-config.yaml
type: mysql
host: 127.0.0.1
port: "3306"
user: root
password: dangerous0
database: clusterpedia
log:
  stdout: true
  colorful: true
  slowThreshold: 100ms
```

The user needs to configure the runtime configuration according to the selected storage layer

#### Run clusterpedia clustersynchro manager
```bash
$ STORAGE_PLUGINS=./plugins ./bin/clustersynchro-manager --kubeconfig ~/.kube/config \
    --storage-name=sample-storage-layer \
    --storage-config ./config.yaml
```

#### Run clusterpedia apiserver
You can choose not to use your own generated certificate, which requires running apiserver without the `-client-ca-file ca.crt` flag.
```bash
$ openssl req -nodes -new -x509 -keyout ca.key -out ca.crt
$ openssl req -out client.csr -new -newkey rsa:4096 -nodes -keyout client.key -subj "/CN=development/O=system:masters"
$ openssl x509 -req -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 01 -sha256 -out client.crt
```

run apiserver
```bash
$ STORAGE_PLUGINS=./plugins ./bin/apiserver --client-ca-file ca.crt  --secure-port 8443 \
    --kubeconfig ~/.kube/config \
    --authentication-kubeconfig ~/.kube/config \
    --authorization-kubeconfig ~/.kube/config \
    --storage-name=sample-storage-layer \
    --storage-config ./config.yaml
```

### Storage Plugin Image + Helm Charts
Clusterpeida already provides several Charts:
* [charts/clusterpedia](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia) is a Chart using the internalstorage storage layer, which can be optionally deployed with MySQL or PostgreSQL, but does not support setting up storage plugins
* [charts/clusterpedia-core](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-core) supports configuration of any storage layer Chart, usually used as a child Chart
* [charts/clusterpedia-mysql](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-mysql) is an **advanced Chart** using MySQL as the storage component, based on clusterpedia-core implementation
* [charts/clusterpedia-postgresql](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-postgresql) is an **advanced Chart** using PostgreSQL as the storage component, based on the clusterpedia-core implementation
* [charts/clusterpedia-elasticsearch](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-elasticsearch) uses Elasticsearch as the **advanced Chart** for the storage component, based on the clusterpedia-core implementation

If you don't need a `storage plugin` and the *internalstorage* storage layer and **relational database** are sufficient, you can use [charts/clusterpedia](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia) directly, **out of the box**.

[clusterpedia-mysql](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-mysql)，[clusterpedia-postgresql](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-postgresql) and [clusterpedia-elasticsearch](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-elasticsearch) are **advanced Charts** based on [charts/clusterpedia-core](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-core), in which the user is shielded from the complex concept of `storage plugins` by default configuration of *clusterpedia-core*'s storage plugin image and storage layer configuration, right **out of the box**.

Although we usually use **Advanced Charts** directly in our usage, knowing how to use [charts/clusterpedia-core](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-core) to set up `storage plugin images` gives us a better understanding of how plugin images work.

#### clusterpedia-core
Let's take [clusterpedia-io/sample-storage](https://github.com/clusterpedia-io/sample-storage) as an example and deploy Clusterpedia using the [ghcr.io/clusterpedia-io/clusterpedia/sample-storage-layer](https://github.com/clusterpedia-io/sample-storage/pkgs/container/clusterpedia%2Fsample-storage-layer) plugin image.

The *clusterpedia-core* does not involve the deployment and installation of any storage components, so users need to configure the storage layer according to the deployed storage components
```yaml
# myvalues.yaml
storage:
  name: "sample-storage-layer"
  image:
    registry: ghcr.io
    repository: clusterpedia-io/clusterpedia/sample-storage-layer
    tag: v0.0.0-v0.6.0
  config:
    type: "mysql"
    host: "10.111.94.196"
    port: 3306
    user: root
    password: dangerous0
    database: clusterpedia
```
`storage.name` sets the storage layer name of the `storage image plugin`

clusterpedia-core copies the `storage plugins` from the plugin image defined by `storage.image` to the component's plugin directory.
```yaml
# helm template clusterpedia -n clusterpedia-system -f myvalues.yaml ./clusterpedia-core
...
      initContainers:
      - name: copy-storage-plugin
        image: ghcr.io/clusterpedia-io/clusterpedia/sample-storage-layer:v0.0.0-v0.6.0
        imagePullPolicy: IfNotPresent
        command:
          - /bin/sh
          - -ec
          - cp /plugins/* /var/lib/clusterpedia/plugins/
        volumeMounts:
        - name: storage-plugins
          mountPath: /var/lib/clusterpedia/plugins
      containers:
      - name: clusterpedia-clusterpedia-core-apiserver
        image: ghcr.io/clusterpedia-io/clusterpedia/apiserver:v0.6.0
        imagePullPolicy: IfNotPresent
        command:
        - /usr/local/bin/apiserver
        - --secure-port=443
        - --storage-name=sample-storage-layer
        - --storage-config=/etc/clusterpedia/storage/config.yaml
        env:
        - name: STORAGE_PLUGINS
          value: /var/lib/clusterpedia/plugins
        volumeMounts:
        - name: storage-config
          mountPath: /etc/clusterpedia/storage
          readOnly: true
        - name: storage-plugins
          mountPath: /var/lib/clusterpedia/plugins
          readOnly: true
      volumes:
      - name: storage-config
        configMap:
          name: clusterpedia-clusterpedia-core-sample-storage-layer-config
      - name: storage-plugins
        emptyDir: {}
...
```
In addition to using `storage.config` to define the runtime configuration *config.yaml* for the storage layer, you can also use the existing configmap and secret.
```yaml
# myvalues.yaml
storage:
  name: "sample-storage-layer"
  image:
    registry: ghcr.io
    repository: clusterpedia-io/clusterpedia/sample-storage-layer
    tag: v0.0.0-v0.6.0

  configMap: "sample-storage-config"
  componentEnv:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: "sample-storage-password"
        key: password
```

*clusterpedia-core* is very flexible in configuration of the storage layer in order to be referenced by other **advanced Charts** as a child Chart, but users do not necessarily need to use clusterpedia-core directly in practice, just the **advanced Charts** deployed for the specific storage component, such as *clusterpedia-mysql* and *clusterpedia-postgresql*.

In the next section we will also describe how to implement **Advanced Charts** for specific storage components based on *clusterpedia-core*.

## Developing `custom storage layer plugins`
[clusterpedia-io/sample-storage](https://github.com/clusterpedia-io/sample-storage) is not only a storage plugin example, but also a template repository where the project structure and most of the build tools can be used in other storage plugin projects

We first clone *sample-storage*, or generate a new storage plugin repository based on *sample-storage*

```bash
$ git clone --recursive https://github.com/clusterpedia-io/sample-storage.git && cd sample-storage
```
Note that when pulling the repository, you need to specify `--recursive` to pull the sub-repository

```bash
$ ls -al
...
-rw-r--r--   1 icebergu  staff    260 12 13 15:14 Dockerfile
-rw-r--r--   1 icebergu  staff   1836 12 13 16:03 Makefile
-rw-r--r--   1 icebergu  staff   2219 11 23 10:25 README.md
drwxr-xr-x  32 icebergu  staff   1024 11 23 10:30 clusterpedia
-rw-r--r--   1 icebergu  staff    156 11 23 10:25 example-config.yaml
-rw-r--r--   1 icebergu  staff   2376 12 13 15:33 go.mod
-rw-r--r--   1 icebergu  staff  46109 12 13 15:33 go.sum
-rw-r--r--   1 icebergu  staff    139 11 23 10:25 main.go
drwxr-xr-x  16 icebergu  staff    512 12 13 15:33 storage
drwxr-xr-x   9 icebergu  staff    288 12 13 15:33 vendor
```
The entire project structure is divided into three categories:
* *main.go*, *storage* package: the core logic of the custom storage plugin
* `clusterpedia` local repository: used for local development and testing
* *Dockerfile* and *Makefile* for project build and image packaging, which can be applied to any storage plugin project

### core logic
*main.go* is the main storage plugin file, mainly used to call the registration function in the storage package -- `RegisterStorageLayer`.
```go
package main

import (
	plugin "github.com/clusterpedia-io/sample-storage-layer/storage"
)

func init() {
	plugin.RegisterStorageLayer()
}
```
The storage package contains the core logic of the storage plugin:
1. Implementing the clusterpedia storage layer interface [`storage.StorageFactory`](https://github.com/clusterpedia-io/clusterpedia/blob/4608c8d13101d82960525dfe39f51e4f64ed49b3/pkg/storage/storage.go#L13)
```go
import (
	"gorm.io/gorm"

	"github.com/clusterpedia-io/clusterpedia/pkg/storage"
)

type StorageFactory struct {
	db *gorm.DB
}

var _ storage.StorageFactory = &StorageFactory{}
```

2. [NewStorageFactory](https://github.com/clusterpedia-io/sample-storage/blob/main/storage/register.go#L33) function to return an instance of `storage.StorageFactory`
```go
func NewStorageFactory(configPath string) (storage.StorageFactory, error)
```

3. The RegisterStorageLayer function registers the NewStorageFactory with clusterpedia
```go
const StorageName = "sample-storage-layer"

func RegisterStorageLayer() {
	storage.RegisterStorageFactoryFunc(StorageName, NewStorageFactory)
}
```

The registered **NewStorageFactory** is automatically called when the user specifies the storage layer with `--storage-name` to create an instance of `storage.StorageFactory`.

### Local development run
To facilitate development and testing, we have added the clusterpedia repository as a subrepository to the storage plugin repository
```bash
$ git submodule status
+4608c8d13101d82960525dfe39f51e4f64ed49b3 clusterpedia (v0.6.0)
```
and replace the clusterpedia repository in go.mod with a local subrepository
```
# go.mod
replace (
        github.com/clusterpedia-io/api => ./clusterpedia/staging/src/github.com/clusterpedia-io/api
        github.com/clusterpedia-io/clusterpedia => ./clusterpedia
)
```
The local cluserpedia subrepository will not be used when building the `storage layer image`

#### Build `storage plugin`
The build of the storage plugin is divided into two parts, building the components in the clusterpedia repository and building the `storage plugin`.
```bash
$ make build-components
OUTPUT_DIR=/Users/icebergu/workspace/clusterpedia/sample-storage-layer ON_PLUGINS=true \
                /Library/Developer/CommandLineTools/usr/bin/make -C clusterpedia all
hack/builder.sh apiserver
hack/builder.sh binding-apiserver
hack/builder.sh clustersynchro-manager
hack/builder-nocgo.sh controller-manager

$ ls -al ./bin
-rwxr-xr-x   1 icebergu  staff  90724968 12 15 09:51 apiserver
-rwxr-xr-x   1 icebergu  staff  91936472 12 15 09:52 binding-apiserver
-rwxr-xr-x   1 icebergu  staff  82826584 12 15 09:52 clustersynchro-manager
-rwxr-xr-x   1 icebergu  staff  45677904 12 15 09:52 controller-manager
```

The `make build-components` command will call `make all` from the clusterpedia repository and output the result to *./bin* directory of the storage plugin project.
> If the clusterpedia subrepository has not changed, then you only need to build the components once

Build the storage plugin
```bash
$ make build-plugin
CLUSTERPEDIA_REPO=/Users/icebergu/workspace/clusterpedia/sample-storage/clusterpedia \
                clusterpedia/hack/builder.sh plugins sample-storage-layer.so

$ ls -al ./plugins
-rw-r--r--   1 icebergu  staff  53354352 12 15 09:47 sample-storage-layer.so
```

Building the storage plugin locally also requires using the [builder.sh](https://github.com/clusterpedia-io/clusterpedia/blob/main/hack/builder.sh) script of the clusterpedia repository to build the plugin binary.

For running storage plugins, see [Running storage plugins locally](#local-run)

### storage plugin image
As mentioned above, `storage plugins` are shared with clusterpedia by image in a real deployment.

The Makefile provides `make image-plugin` to build images and `make push-images` to publish them.

#### Building images
To build a plugin image, we need to use the [clusterpedia/builder](https://github.com/clusterpedia-io/clusterpedia/pkgs/container/clusterpedia%2Fbuilder) image as the base image to build the plugin, and the builder image needs to be the same version as the clusterpedia component that uses the plugin
```bash
$ BUILDER_IMAGE=ghcr.io/clusterpedia-io/clusterpedia/builder:v0.6.0 make image-plugin
```
clusterpedia maintains a builder image of the published version, and users can also use their own locally built builder image

Build the builder image locally
```bash
$ cd clusterpedia
$ make image-builder
docker buildx build \
                -t "ghcr.io/clusterpedia-io/clusterpedia"/builder-amd64:4608c8d13101d82960525dfe39f51e4f64ed49b3 \
                --platform=linux/amd64 \
                --load \
                -f builder.dockerfile . ; \
```

The tag format for `storage plugin images` is < stroage version >-<clusterpedia-version/commit>, for example: *ghcr.io/clusterpedia-io/clusterpedia/sample-storage-layer:v0.0.0-v0.6.0*

The storage plugin image can be deployed in the <clusterpedia-version/commit> version of Clusterpedia

#### push images
`make image-plugin` builds the storage plugin image based on the manually set builder image

While pushing images with `make push-images` automatically builds images for all compatible versions and architectures
```makefile
# Makefile
CLUSTERPEDIA_VERSIONS = v0.6.0-beta.1 v0.6.0
RELEASE_ARCHS ?= amd64 arm64
```

Once the image is built, [the storage plugin image can be used via cluserpedia-core](#clusterpedia-core)

## Advanced Chart based on clusterpedia-core
After implementing our own storage plugin, we still need to provide an **Advanced Chart** based on *clusterpedia-core* Chart to make it easier to use.

**Advanced Chart** needs to provide the following capabilities:
* Set the default storage plugin image
* Set the storage layer name
* Support dynamic setting of the runtime configuration of the storage layer
* Provide configuration and installation of storage components

Create a new Chart using the *sample-storage* storage plugin -- *clusterpedia-sample-mysql*, which will use mysql as the storage component.
```yaml
# Chart.yaml
dependencies:
  - name: mysql
    repository: https://charts.bitnami.com/bitnami
    version: 9.x.x
  - name: common
    repository: https://charts.bitnami.com/bitnami
    version: 1.x.x
  - name: clusterpedia-core
    repository: https://clusterpedia-io.github.io/clusterpedia-helm/
    version: 0.1.x
```

We need to override the storage layer related settings in clusterpedia-core, which provides both **values.yaml** and **dynamic templates** to set up the storage plugin and storage layer information

We override the static settings of the storage layer in values.yaml, such as **plugin image** and **storage layer name**
```yaml
# values.yaml
clusterpedia-core:
  storage:
    name: "sample-storage-layer"
    image:
      registry: "ghcr.io"
      repository: "clusterpedia-io/clusterpedia/sample-storage-layer"
      tag: "v0.0.0-v0.6.0"
```

The *config.yaml* and some environment variables of the custom storage layer generally need to refer to ConfigMap and Secret, and the names of these resources will change dynamically according to the Chart release name, so we need to use the **dynamic templates** way to set

clusterpedia-core provides three overriding naming templates
```yaml
# clusterpedia-core/templates/_storage_override.yaml
{{- define "clusterpedia.storage.override.initContainers" -}}
{{- end -}}

{{- define "clusterpedia.storage.override.configmap.name" -}}
{{- end -}}

{{- define "clusterpedia.storage.override.componentEnv" -}}
{{- end -}}
```
Each of them can be set as follows:
* `apiserver` and `clustersynchro manager` init containers before running
* ConfigMap name to store the config.yaml configuration that the plugin needs to read
* Environment variables to be used by the storage plugin

Let's take [clusterpedia-mysql](https://github.com/clusterpedia-io/clusterpedia-helm/blob/main/charts/clusterpedia-mysql/templates/_storage_override.yaml) as an example and see how it is set
```yaml
# _storage_override.yaml
{{- define "clusterpedia.storage.override.initContainers" -}}
- name: ensure-database
  image: docker.io/bitnami/mysql:8.0.28-debian-10-r23
  command:
  - /bin/sh
  - -ec
  - |
    if [ ${CREARE_DATABASE} = "ture" ]; then
      until mysql -u${STORAGE_USER} -p${DB_PASSWORD} --host=${STORAGE_HOST} --port=${STORAGE_PORT} -e 'CREATE DATABASE IF NOT EXISTS ${STORAGE_DATABASE}'; do
      echo waiting for database check && sleep 1;
      done;
      echo 'DataBase OK ✓'
    else
      until mysqladmin status -u${STORAGE_USER} -p${DB_PASSWORD} --host=${STORAGE_HOST} --port=${STORAGE_PORT}; do sleep 1; done
    fi
  env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: {{ include "clusterpedia.mysql.storage.fullname" . }}
        key: password
  envFrom:
  - configMapRef:
      name: {{ include "clusterpedia.mysql.storage.initContainer.env.name" . }}
{{- end -}}
```

clusterpedia-mysql defines the environment variables needed to init containers in the [storage-initcontainer-env-configmap.yaml](https://github.com/clusterpedia-io/clusterpedia-helm/blob/main/charts/clusterpedia-mysql/templates/storage-initcontainer-env-configmap.yaml)
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "clusterpedia.mysql.storage.initContainer.env.name" . }}
  namespace: {{ .Release.Namespace }}
  labels: {{ include "common.labels.standard" . | nindent 4 }}
data:
  STORAGE_HOST: {{ include "clusterpedia.mysql.storage.host" . | quote }}
  STORAGE_PORT: {{ include "clusterpedia.mysql.storage.port" . | quote }}
  STORAGE_USER: {{ include "clusterpedia.mysql.storage.user" . | quote }}
  STORAGE_DATABASE: {{ include "clusterpedia.mysql.storage.database" . | quote }}
  CREARE_DATABASE: {{ .Values.externalStorage.createDatabase | quote }}
```

The init container dynamically set by the `clusterpedia.storage.override.initContainers` naming template will be rendered to Deployment
```yaml
# helm template clusterpedia -n clusterpedia-system --set persistenceMatchNode=None .
...
    spec:
      initContainers:
      - name: ensure-database
        image: docker.io/bitnami/mysql:8.0.28-debian-10-r23
        command:
        - /bin/sh
        - -ec
        - |
          if [ ${CREARE_DATABASE} = "ture" ]; then
            until mysql -u${STORAGE_USER} -p${DB_PASSWORD} --host=${STORAGE_HOST} --port=${STORAGE_PORT} -e 'CREATE DATABASE IF NOT EXISTS ${STORAGE_DATABASE}'; do
            echo waiting for database check && sleep 1;
            done;
            echo 'DataBase OK ✓'
          else
            until mysqladmin status -u${STORAGE_USER} -p${DB_PASSWORD} --host=${STORAGE_HOST} --port=${STORAGE_PORT}; do sleep 1; done
          fi
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: clusterpedia-mysql-storage
              key: password
        envFrom:
        - configMapRef:
            name: clusterpedia-mysql-storage-initcontainer-env
```

The ConfigMap and environment variables of the storage plugin runtime configuration *config.yaml* are also dynamically configured in clusterpedia-mysql
```yaml
# _storage_override.yaml

{{- define "clusterpedia.storage.override.configmap.name" -}}
{{- printf "%s-mysql-storage-config" .Release.Name -}}
{{- end -}}

{{- define "clusterpedia.storage.override.componentEnv" -}}
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: {{ include "clusterpedia.mysql.storage.fullname" . }}
      key: password
{{- end -}}
```

The storage layer configuration is set to the APIServer and ClusterSynchro Manager Deployment through static override of values and dynamic setting of naming templates

**Advanced Chart** like *clusterpedia-mysql* allows you to mask the use of underlying storage plugins for users out of the box
