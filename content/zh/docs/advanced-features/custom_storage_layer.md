---
title: 自定义存储层插件
weight: 15
---

Clusterpedia 可以通过存储层来使用不同的存储组件，例如 MySQL/PostgreSQL, Memory, Elasticsearch

当前，Clusterpedia 内置了两个存储层：
* 用于接入关系型数据库的 [internalstorage](https://clusterpedia.io/zh-cn/docs/installation/configurate/configurate-internalstorage/) 存储层
* 基于内存的 [memory](https://github.com/clusterpedia-io/clusterpedia/tree/main/pkg/storage/memorystorage) 存储层

尽管 Clusterpedia 已经默认支持了关系型数据库和内存，但是用户的需求往往是多变和复杂的，固定的存储层可能无法满足不同用户对于存储组件和性能的要求，于是 Clusterpedia 支持通过插件的方式来接入用户自己实现的存储层，我们称这种方式为 `自定义存储层插件`，简称为 `存储插件`。

通过`存储插件`，用户可以做到以下事情：
* **使用任意的存储组件**，例如 `Elasticsearch`，`RedisGraph`，`Etcd`，甚至是 `MessageQueue` 也没有问题
* 允许用户针对自己的业务来优化资源的存储格式和查询性能
* 针对存储组件的特性来实现更高级的检索功能

Clusterpedia 也维护了一些`存储插件`，用户可以根据需求选用：
* [Sample Storage](https://github.com/clusterpedia-io/sample-storage): 可以连接**关系型数据库**的存储插件示例
* [Elasticsearch Storage](https://github.com/clusterpedia-io/elasticsearch-storage): 用于连接 **Elasticsearch** 的存储插件

`存储插件`是通过 **Go Plugin** 的方式来让 Clusterpedia 组件加载，相比 RPC 或者其他方式，这种方式在没有任何损耗的前提下提供非常灵活的插件接入。
> 关于 **Go Plugin** 对性能的影响可以简单参考 https://github.com/uberswe/goplugins

众所周知，**Go Plugin** 在开发和使用上都比较繁琐，不过 Clusterpedia 通过一些机制来巧妙的优化了存储插件的使用和开发，并且提供了 [clusterpedia-io/sample-storage](https://github.com/clusterpedia-io/sample-storage) 插件作为参考。

下面我们以 [clusterpedia-io/sample-storage](https://github.com/clusterpedia-io/sample-storage) 为例，分别介绍：
* [如何使用 `自定义存储层插件`](#使用-自定义存储层插件)
* [如何实现 `自定义存储层插件`](#开发自定义存储层插件)

## 使用 `自定义存储层插件`
存储插件的使用可以大致分为三种方式：
1. 运行 Clusterpedia 组件二进制并加载 `存储插件`
2. 使用基础 Chart —— clusterpedia-core 来设置`存储插件镜像`和配置存储层
3. 使用 Clusterpedia **高级 Chart**，无需关心存储插件的设置

通过本地运行组件二进制，我们可以更加了解 Cluserpedia 组件是如何加载和运行 `存储插件`，

用户在真正使用时通常是利用已经构建好的`存储插件镜像`，或者是直接部署 Clusterpedia **高级 Chart**
### 本地运行
#### 插件构建
存储插件实际是一个 *.so* 后缀的动态链接库，Clusterpedia 组件在启动时可以加载`存储插件`，并根据指定的存储层名称来使用具体的`存储插件`。

我们以 [clusterpedia-io/sample-storage](https://github.com/clusterpedia-io/sample-storage) 为例，构建出一个`存储插件`二进制
```bash
$ git clone --recursive https://github.com/clusterpedia-io/sample-storage.git && cd sample-storage
$ make build-plugin
```

使用 **file** 命令查看`存储插件`信息
```
$ file ./plugins/sample-storage-layer.so
./plugins/sample-storage-layer.so: Mach-O 64-bit dynamically linked shared library x86_64
```

Clusterpedia 的 `ClusterSynchro Manager` 和 `APIServer` 组件可以通过环境变量和命令参数来加载和使用存储插件： 
* `STORAGE_PLUGINS=<plugins dir>` 环境变量设置插件所在目录，Clusterpedia 会将该目录下所有插件都加载到组件中
* `--storage-name=<storage name>` 插件命令参数，设置存储层名称
* `--storage-config=<storage config path>` 插件命令参数，设置存储层配置

#### 组件构建
本地运行时，为了保证依赖一致，需要在本地通过 `make build-components` 命令构建 clusterpedia 组件
```bash
$ # 在 sample-storage 目录
$ make build-components
$ ls -al ./bin
-rwxr-xr-x   1 icebergu  staff  90707488 11  7 11:15 apiserver
-rwxr-xr-x   1 icebergu  staff  91896016 11  7 11:16 binding-apiserver
-rwxr-xr-x   1 icebergu  staff  82769728 11  7 11:16 clustersynchro-manager
-rwxr-xr-x   1 icebergu  staff  45682000 11  7 11:17 controller-manager
```
> 关于存储插件与 Clusterpedia 组件的构建可以查看 [开发自定义存储层插件](#本地开发运行)

#### 存储插件运行时的配置文件
在运行 clusterpedia 前还需要准备`存储插件`在运行时的配置文件，*sample-storage* 提供了它的配置示例 [example-config.yaml](https://github.com/clusterpedia-io/sample-storage/blob/main/example-config.yaml)

在运行时 clusterpedia 组件时，通过 `--storage-config=./config.yaml` 来指定运行时配置文件
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
用户需要根据选择的存储层来配置运行时配置

#### 运行 clusterpedia clustersynchro manager
```bash
$ STORAGE_PLUGINS=./plugins ./bin/clustersynchro-manager --kubeconfig ~/.kube/config \
    --storage-name=sample-storage-layer \
    --storage-config ./config.yaml
```

#### 运行 clusterpedia apiserver
可以选择不使用自己生成的证书，这时需要在运行 apiserver 忽略掉 `--client-ca-file ca.crt` 参数
```bash
$ openssl req -nodes -new -x509 -keyout ca.key -out ca.crt
$ openssl req -out client.csr -new -newkey rsa:4096 -nodes -keyout client.key -subj "/CN=development/O=system:masters"
$ openssl x509 -req -days 365 -in client.csr -CA ca.crt -CAkey ca.key -set_serial 01 -sha256 -out client.crt
```

运行 apiserver
```bash
$ STORAGE_PLUGINS=./plugins ./bin/apiserver --client-ca-file ca.crt  --secure-port 8443 \
    --kubeconfig ~/.kube/config \
    --authentication-kubeconfig ~/.kube/config \
    --authorization-kubeconfig ~/.kube/config \
    --storage-name=sample-storage-layer \
    --storage-config ./config.yaml
```

### 存储插件镜像 + Helm Charts
Clusterpeida 已经提供了多个 Charts：
* [charts/clusterpedia](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia) 使用 internalstorage 存储层的 Chart，可以选择部署 MySQL 或者 PostgreSQL，但是不支持设置存储插件
* [charts/clusterpedia-core](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-core) 支持配置任意存储层的 Chart，通常作为子 Chart 来使用
* [charts/clusterpedia-mysql](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-mysql) 使用 MySQL 作为存储组件的**高级 Chart**，基于 clusterpedia-core 实现
* [charts/clusterpedia-postgresql](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-postgresql) 使用 PostgreSQL 作为存储组件的**高级 Chart**，基于 clusterpedia-core 实现
* [charts/clusterpedia-elasticsearch](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-elasticsearch) 使用 Elasticsearch 作为存储组件的**高级 Chart**，基于 clusterpedia-core 实现

如果用户没有`存储插件`需求，默认存储层和关系型数据库已经可以满足需求的话，可以直接选用 [charts/clusterpedia](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia)，开箱即用

[clusterpedia-mysql](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-mysql)，[clusterpedia-postgresql](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-postgresql) 和 [clusterpedia-elasticsearch](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-elasticsearch) 便是基于 [clusterpedia-core](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-core) 实现的**高级 Charts**，在这些 Charts 中通过默认配置 clusterpedia-core 的`存储插件镜像`和存储层配置来为用户屏蔽掉`存储插件`的复杂概念，开箱即用

尽管我们在使用中通常会直接使用**高级 Charts**，但是知道如何使用 [clusterpedia-core](https://github.com/clusterpedia-io/clusterpedia-helm/tree/main/charts/clusterpedia-core) 来设置`存储插件镜像`，可以让我们更好的理解`插件镜像`是如何工作的。

#### clusterpedia-core
我们以 [clusterpedia-io/sample-storage](https://github.com/clusterpedia-io/sample-storage) 为例，使用 [ghcr.io/clusterpedia-io/clusterpedia/sample-storage-layer](https://github.com/clusterpedia-io/sample-storage/pkgs/container/clusterpedia%2Fsample-storage-layer) 插件镜像来部署 Clusterpedia。

*clusterpedia-core* 不涉及任何存储组件的部署安装，所以用户需要根据已部署的存储组件来配置存储层
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
`storage.name` 配置存储镜像插件的存储层名称

clusterpedia-core 会将 `storage.image` 定义的插件镜像中的`存储插件`复制到组件的插件目录下
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

除了使用 `storage.config` 定义存储层的运行时配置 config.yaml 外，还可以使用已有的 configmap 以及 secret
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

*clusterpedia-core* 为了能够作为子 Chart 被其他高级 Charts 引用，在存储层配置上是非常灵活的，但是用户实际在使用中并不一定需要直接使用 clusterpedia-core，只需要使用针对具体存储组件部署的**高级 Charts** 即可，例如 *clusterpedia-mysql* 和 *clusterpedia-postgresql*

在下节我们也会介绍如何基于 *clusterpedia-core* 来针对具体存储组件实现**高级 Chart**。

## 开发`自定义存储层插件`
[clusterpedia-io/sample-storage](https://github.com/clusterpedia-io/sample-storage) 不仅仅是一个存储插件示例，也是一个模版仓库，其中项目结构和大部分构建工具都可以在其他存储插件项目中使用

我们首先 clone *sample-storage*，或者根据 *sample-storage* 生成一个新的存储插件仓库
```bash
$ git clone --recursive https://github.com/clusterpedia-io/sample-storage.git && cd sample-storage
```
注意拉取仓库时，需要指定 `--recursive` 拉取子仓库

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
整个项目结构分为三类
* `main.go`, `storage` 包：`自定义存储层插件`的核心逻辑
* `clusterpedia` 主库，用于本地开发和测试
* *Dockerfile* 和 *Makefile* 用于项目构建和镜像打包，可以适用于任何存储插件项目

### 核心逻辑
*main.go* 是存储插件的主文件，主要用来调用 storage 包中的注册函数 —— `RegisterStorageLayer`。
```go
package main

import (
	plugin "github.com/clusterpedia-io/sample-storage-layer/storage"
)

func init() {
	plugin.RegisterStorageLayer()
}
```

storage 包中的是存储插件的核心逻辑：
1. 实现 clusterpedia 存储层接口 [`storage.StorageFactory`](https://github.com/clusterpedia-io/clusterpedia/blob/4608c8d13101d82960525dfe39f51e4f64ed49b3/pkg/storage/storage.go#L13)
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

2. [NewStorageFactory](https://github.com/clusterpedia-io/sample-storage/blob/main/storage/register.go#L33) 函数来返回 `storage.StorageFactory` 实例
```go
func NewStorageFactory(configPath string) (storage.StorageFactory, error)
```

3. RegisterStorageLayer 函数将 NewStorageFactory 注册到 clusterpedia 中
```go
const StorageName = "sample-storage-layer"

func RegisterStorageLayer() {
	storage.RegisterStorageFactoryFunc(StorageName, NewStorageFactory)
}
```

当用户通过 `--storage-name` 指定该存储层时，会自动调用注册的 **NewStorageFactory** 来创建 `storage.StorageFactory` 实例。
```bash
./bin/apiserver --storage-name=sample-storage-layer <other flags>
```

### 本地开发运行
为了方便开发测试，我们在存储插件的仓库中添加了 clusterpedia 主库作为子库
```bash
$ git submodule status
+4608c8d13101d82960525dfe39f51e4f64ed49b3 clusterpedia (v0.6.0)
```

并且将 go.mod 中 clusterpedia 仓库 replace 为本地子库
```
# go.mod
replace (
        github.com/clusterpedia-io/api => ./clusterpedia/staging/src/github.com/clusterpedia-io/api
        github.com/clusterpedia-io/clusterpedia => ./clusterpedia
)
```
在构建存储层镜像时，不会使用本地 cluserpedia 子库

#### 存储插件构建
存储插件的构建分为两部分，分别是构建 clusterpedia 子库中组件和构建存储插件

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
`make build-components` 命令会在调用 clusterpedia 库中的 `make all`，并将结果输出到存储插件项目的 ./bin 目录下
> 如果 clusterpedia 子库未发生更改，那么只需要构建组件即可

构建存储插件
```bash
$ make build-plugin
CLUSTERPEDIA_REPO=/Users/icebergu/workspace/clusterpedia/sample-storage/clusterpedia \
                clusterpedia/hack/builder.sh plugins sample-storage-layer.so

$ ls -al ./plugins
-rw-r--r--   1 icebergu  staff  53354352 12 15 09:47 sample-storage-layer.so
```
本地构建存储插件也是需要使用 clusterpedia 库的 [builder.sh](https://github.com/clusterpedia-io/clusterpedia/blob/main/hack/builder.sh) 脚本来构建插件二进制

关于运行存储层插件可以参考 [本地运行存储插件](#本地运行)

### 存储插件镜像
上文提到，存储插件在真正的部署中，是通过镜像的方式将存储插件共享给 clusterpedia。

Makefile 中提供 `make image-plugin` 来构建镜像，`make push-images` 来发布镜像

#### 构建镜像
构建插件镜像时，我们需要使用 [clusterpedia/builder](https://github.com/clusterpedia-io/clusterpedia/pkgs/container/clusterpedia%2Fbuilder) 镜像作为基础镜像来构建插件，builder 镜像的版本需要和使用插件的 clusterpedia 组件的版本一致
```bash
$ BUILDER_IMAGE=ghcr.io/clusterpedia-io/clusterpedia/builder:v0.6.0 make image-plugin
```
clusterpedia 中维护了已发布版本的 builder 镜像，用户也可以使用自己本地构建的 builder 镜像

本地构建 builder 镜像
```bash
$ cd clusterpedia
$ make image-builder
docker buildx build \
                -t "ghcr.io/clusterpedia-io/clusterpedia"/builder-amd64:4608c8d13101d82960525dfe39f51e4f64ed49b3 \
                --platform=linux/amd64 \
                --load \
                -f builder.dockerfile . ; \
```

`存储插件镜像`的 tag 格式为 <storage-version>-<clusterpedia-version/commit>，例如：
*ghcr.io/clusterpedia-io/clusterpedia/sample-storage-layer:v0.0.0-v0.6.0*

存储插件镜像可以部署在 <clusterpedia-version/commit> 版本的 Clusterpedia 中

#### 镜像推送
`make image-plugin` 根据手动设置的 builder 镜像来构建存储插件镜像

而使用 `make push-images` 推送镜像时会自动为所有兼容版本和架构构建镜像
```makefile
# Makefile
CLUSTERPEDIA_VERSIONS = v0.6.0-beta.1 v0.6.0
RELEASE_ARCHS ?= amd64 arm64
```

镜像构建好后，可以[通过 cluserpedia-core 来使用`存储插件镜像`](#clusterpedia-core)

## 基于 clusterpeida-core 实现高级 Chart
我们在实现自己的存储插件后，为了更加方便的使用，还是需要基于 clusterpedia-core Chart 来封装出**高级 Chart**

**高级 Chart** 需要提供以下能力：
* 设置默认的`存储插件镜像`
* 设置存储层名称
* 支持动态设置存储层的运行时配置
* 提供对存储组件的配置和安装

创建一个使用 *sample-storage* 存储插件的新 Chart —— clusterpedia-sample-mysql，这个 Chart 会使用 mysql 作为存储组件。
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

我们需要覆盖 clusterpedia-core 中存储层相关的设置，clusterpedia-core 提供 **values.yaml** 和**动态模版**两种方式来设置存储插件和存储层信息

我们在 values.yaml 覆盖存储层的静态设置，例如**插件镜像**和**存储层名称**
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

自定义存储层的 *config.yaml* 和一些环境变量的设置，一般需要引用 ConfigMap 和 Secret，而这些资源的名称会根据 Chart 的 Release 名字动态变化，所以我们需要使用 **动态模版** 的方式来设置

clusterpedia-core 提供了三个可以覆盖的命名模版
```yaml
# clusterpedia-core/templates/_storage_override.yaml
{{- define "clusterpedia.storage.override.initContainers" -}}
{{- end -}}

{{- define "clusterpedia.storage.override.configmap.name" -}}
{{- end -}}

{{- define "clusterpedia.storage.override.componentEnv" -}}
{{- end -}}
```
可以分别设置：
* `apiserver`，`clustersynchro manager` 组件运行前的 init containers
* 保存存储插件需要读取的 config.yaml 配置的 ConfigMap 名称
* 存储插件需要使用的环境变量

我们以 [clusterpedia-mysql](https://github.com/clusterpedia-io/clusterpedia-helm/blob/main/charts/clusterpedia-mysql/templates/_storage_override.yaml) 为例，看看它的设置
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
clusterpedia-mysql 在 [storage-initcontainer-env-configmap.yaml](https://github.com/clusterpedia-io/clusterpedia-helm/blob/main/charts/clusterpedia-mysql/templates/storage-initcontainer-env-configmap.yaml) 中定义了 init container 需要的环境变量
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

通过 `clusterpedia.storage.override.initContainers` 命名模版动态设置的 init container 会被渲染到 Deployment 中：
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

保存`存储插件`运行时配置 *config.yaml* 的 ConfigMap 和环境变量也是在 clusterpedia-mysql 中动态配置
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

通过 values 静态覆盖和 命名模版的动态设置，存储层配置会被设置到 APIServer 和 ClusterSynchro Manager 的 Deployment 中

通过类似 *clusterpedia-mysql* 的**高级 Chart** 可以为用户屏蔽掉底层存储插件的使用，达到开箱即用。
