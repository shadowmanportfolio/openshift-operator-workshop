# Operator SDK with Helm

在之前的学习模块中，我们介绍了如何使用Operator SDK轻松创建以下类型的Operator:

- **Go**:

理想的传统软件开发团队，希望得到一个全自动驾驶operator。它使您能够利用上游项目在底层使用的相同Kubernetes库。

- **Ansible**:

对于那些在Ansible模块上有投资，但希望以kubernetes原生方式使用它们的基础设施团队非常有用。使用Ansible配置集群外对象(如硬件负载均衡器)也很不错。

现在我们将专注于开始开发operator的最简单的方法:

- **Helm**:

它不需要手动调用Helm来重新配置应用程序。

## 从Helm图创建一个CockroachDB operator

在本教程中，我们将从现有的 [CockroachDB Helm Chart](https://github.com/helm/charts/tree/master/stable/cockroachdb) 创建一个cockachdb operator。

[CockroachDB](https://www.cockroachlabs.com/)是建立在事务性和强一致性的键-值存储上的分布式SQL数据库。它可以:

- 横向扩展
- 在磁盘、机器、机架甚至数据中心故障中幸存下来，延迟中断最小，无需人工干预。
- 支持强一致性的ACID事务，并为结构化、操作和查询数据提供了熟悉的SQL API

让我们开始！

### 初始化项目

让我们从连接到OpenShift开始，可从web console中获得具体的登录链接。例如:

```shell
oc login --token=7DMj4CRxuUCzXXXXXXXXXX --server=https://XXXXX.com:30596
```

Create a new project called `myproject`:
创建一个名为 `myproject` 的新项目:

```
oc new-project myproject
```

现在让我们为我们的项目创建一个新目录:

```
mkdir -p $HOME/projects/cockroachdb-operator
```

导航到目录:

```
cd $HOME/projects/cockroachdb-operator
```

为CockroachDB Operator创建一个新的基于Helm的Operator SDK项目:

```
operator-sdk init --plugins=helm --domain example.com
```

自动获取Cockroachdb Helm Chart并生成CRD/API:

```
operator-sdk create api --helm-chart=cockroachdb --helm-chart-repo=https://charts.helm.sh/stable --helm-chart-version=3.0.7
```

### 项目脚手架布局

在创建一个新的operator项目后，目录中有许多生成的文件夹和文件。下表描述了每个生成的文件/目录的基本纲要。

| File/Folders | 目的                                                                                    |
| ------------ | ------------------------------------------------------------------------------------- |
| config       | 定制在集群上启动控制器所需的YAML定义。它是保存CustomResourceDefinitions、RBAC配置和WebhookConfigurations的目标目录。 |
| Dockerfile   | 用于构建Ansible Operator容器镜像的容器构建文件。                                                      |
| helm-charts  | 指定的helm-charts的位置。                                                                    |
| Makefile     | 为构建和部署控制器制定目标。                                                                        |
| PROJECT      | 用于搭建新组件的Kubebuilder元数据。                                                               |
| watches.yaml | 包含分组、版本、类型和所需chart。                                                                   |

### 更新Watches文件

 `watches.yaml` 文件将组、版本和类型映射到特定的Helm Chart。观察 `watches.yaml` 的内容:

```
cd /root/projects/cockroachdb-operator/ && \
  cat watches.yaml
```

### 应用CockroachDB自定义资源定义

将CockroachDB自定义资源定义应用到集群:

```
oc apply -f config/crd/bases/charts.example.com_cockroachdbs.yaml
```

一旦CRD被注册，有两种方式来运行Operator:

- 作为Kubernetes集群中的Pod。
- 作为一个使用Operator-SDK的集群外的Go程序。这对于你的operator的本地开发是很好的。

为了本教程的目的，我们将使用Operator-SDK和我们的 `kubeconfig` 凭证在集群外运行Operator作为Go程序

该命令一旦运行，将阻塞当前会话。您可以使用另一个终端选项卡继续与OpenShift集群交互。您可以通过按 `CTRL + C` 退出会话。

```
WATCH_NAMESPACE=myproject make run
```

### 应用CockroachDB自定义资源

从导航到 `cockroachdb-operator` 顶层目录:

```
cd projects/cockroachdb-operator
```

在应用CockroachDB自定义资源之前，请观察CockroachDB Helm Chart `values.yaml`:

[CockroachDB Helm Chart Values.yaml 文件](https://github.com/helm/charts/blob/master/stable/cockroachdb/values.yaml)

更新CockroachDB自定义资源 `config/samples/charts_v1alpha1_cockroachdb.yaml` 的以下配置值:

- `spec.statefulset.replicas: 1`
- `spec.storage.persistentVolume.size: 1Gi`
- `spec.storage.persistentVolume.storageClass: gp2`

```
apiVersion: charts.example.com/v1alpha1
kind: Cockroachdb
metadata:
  name: cockroachdb-sample
spec:
  statefulset:
    replicas: 1
  storage:
    persistentVolume:
      size: 1Gi
      storageClass: gp2
```

在用我们想要的规范更新CockroachDB Custom Resource之后，将其应用到集群中。确保你当前的作用域是 `myproject` 命名空间:

```
oc project myproject
```

```
oc apply -f config/samples/charts_v1alpha1_cockroachdb.yaml
```

确认已创建自定义资源:

```
oc get cockroachdb
```

环境下拉CockroachDB容器镜像可能需要一些时间。确认Stateful Set已经创建:

```
oc get statefulset
```

确认状态集的pod当前正在运行:

```
oc get pods -l app.kubernetes.io/component=cockroachdb
```

确认创建了CockroachDB "internal" 和 "public" ClusterIP服务:

```
oc get services
```

### 进入CockroachDB Web UI

通过将CockroachDB Service公开为公开可访问的OpenShift Route，验证您可以访问cockachdb Web UI:

```
COCKROACHDB_PUBLIC_SERVICE=`oc get svc -o jsonpath={$.items[1].metadata.name}`
oc expose --port=http svc $COCKROACHDB_PUBLIC_SERVICE
```

获取OpenShift Route URL并复制/粘贴到您的浏览器:

```
COCKROACHDB_UI_URL=`oc get route -o jsonpath={$.items[0].spec.host}`
echo $COCKROACHDB_UI_URL
```

### 连接到CockroachDB集群

让我们打通从集群内到服务的连接，与CockroachDB集群进行通信。CockroachDB是PostgreSQL有线协议兼容的，因此有各种各样的支持的客户端。为了举例说明，我们将使用CockroachDB的内置shell打开一个SQL shell，并对其进行一些操作。

```
oc run -it --rm cockroach-client --image=cockroachdb/cockroach --restart=Never --command -- ./cockroach sql --insecure --host $COCKROACHDB_PUBLIC_SERVICE
```

看到SQL提示符后，运行以下命令显示默认数据库:

```
SHOW DATABASES;
```

创建一个名为 `bank` 的新数据库，用随机值填充表accounts:

```
CREATE DATABASE bank;
CREATE TABLE bank.accounts (id INT PRIMARY KEY, balance DECIMAL);
INSERT INTO bank.accounts VALUES (1234, 10000.50);
```

验证表和值已成功创建:

```
SELECT * FROM bank.accounts;
```

Exit the SQL prompt:

```
\q
```

### 更新CockroachDB自定义资源

现在让我们更新CockroachDB `example` 自定义资源，并将所需的副本数量增加到 `3`:

```
oc patch cockroachdb cockroachdb-sample --type='json' -p '[{"op": "replace", "path": "/spec/statefulset/replicas", "value":3}]'
```

验证CockroachDB Stateful Set正在创建两个额外的pod:

```
oc get pods -l app.kubernetes.io/component=cockroachdb
```

CockroachDB UI现在也应该反映这些额外的节点。

### 测试CockroachDB集群故障切换

如果任何CockroachDB成员失败，它将被重启或由Kubernetes基础设施自动重新创建，并在集群恢复时自动重新加入集群。你可以通过杀死任何一个pod来测试这个场景:

```
oc delete pods -l app.kubernetes.io/component=cockroachdb
```

观看pod重生:

```
oc get pods -l app.kubernetes.io/component=cockroachdb
```

通过连接到数据库集群，确认数据库的内容仍然存在:

```
COCKROACHDB_PUBLIC_SERVICE=`oc get svc -o jsonpath={$.items[1].metadata.name}`
oc run -it --rm cockroach-client --image=cockroachdb/cockroach --restart=Never --command -- ./cockroach sql --insecure --host $COCKROACHDB_PUBLIC_SERVICE
```

看到SQL提示符后，执行以下命令确认数据库内容仍然存在:

```
SELECT * FROM bank.accounts;
```

退出SQL提示符:

```
\q
```

### 清理

通过删除 `example` 自定义资源，删除CockroachDB集群和所有相关资源:

```
oc delete cockroachdb cockroachdb-sample
```

验证Stateful Set、pod和服务是否被移除:

```
oc get statefulset
oc get pods
oc get services
```

## 从Helm Chart创建Memcached operator

我们将从现有的[Memcached Helm Chart](https://github.com/helm/charts/blob/master/stable/memcached/Chart.yaml)创建一个Memcached operator。

[Memcached](https://memcached.org/)是一个免费、开源、高性能的分布式内存对象缓存系统，本质上是通用的，但旨在通过减轻数据库负载来加速动态web应用程序。

让我们开始！

### 初始化项目

Memcached是一个内存中的键-值存储，用于存储来自数据库调用、API调用或页面呈现结果的任意数据块(字符串、对象)。

现在让我们为我们的项目创建一个新目录:

```
mkdir -p $HOME/projects/memcached-operator
```

导航到目录:

```
cd $HOME/projects/memcached-operator
```

为Memcached Operator创建一个新的基于helm的Operator SDK项目:

```
operator-sdk init --plugins helm --domain example.com
```

对于基于helm的项目， `operator-sdk` 初始化也会在 `config/rbac/role.yaml` 中生成RBAC规则。这是基于chart默认清单将部署的资源。一定要仔细检查在 `config/rbac/role.yaml` 中生成的规则，满足运营商的权限要求。

要了解更多关于项目目录结构的信息，请参见[项目布局](https://sdk.operatorframework.io/docs/overview/project-layout)文档。

### 使用现有 chart

您也可以使用 `--helm-chart`、 `--helm-chart-repo`和 `--helm-chart-version` 来使用本地文件系统或远程chart存储库中的现有chart，而不是使用Helm chart的样板文件创建项目。

自动获取Memcached Helm Chart并生成CRD/API:

```
operator-sdk create api --helm-chart memcached --helm-chart-repo=https://charts.helm.sh/stable
```

### 项目脚手架布局

在创建一个新的operator项目后，目录中有许多生成的文件夹和文件。下表描述了每个生成的文件/目录的基本纲要。

| File/Folders | 作用                                                                                    |
| ------------ | ------------------------------------------------------------------------------------- |
| config       | 定制在集群上启动控制器所需的YAML定义。它是保存CustomResourceDefinitions、RBAC配置和WebhookConfigurations的目标目录。 |
| Dockerfile   | 用于构建Ansible Operator容器映像的容器构建文件。                                                      |
| helm-charts  | 指定的helm-chart的位置。                                                                     |
| Makefile     | 为构建和部署控制器制定目标。                                                                        |
| PROJECT      | 用于搭建新组件的Kubebuilder元数据。                                                               |
| watches.yaml | 包含分组、版本、类型和所需chart。                                                                   |

### 更新Watches文件

 `watches.yaml` 文件将组、版本和类型映射到特定的Helm Chart。观察 `watches.yaml` 的内容:

```
cd /root/projects/memcached-operator && \
  cat watches.yaml
```

### 应用Memcached自定义资源定义

应用Memcached自定义资源定义到集群:

```
cd /root/projects/memcached-operator && \
  oc apply -f config/crd/bases/charts.example.com_memcacheds.yaml
```

一旦CRD被注册，有两种方式来运行Operator:

- 作为Kubernetes集群中的Pod。
- 作为一个使用Operator-SDK的集群外的Go程序。这对于你的operator的本地开发是很好的。

为了本教程的目的，我们将使用Operator-SDK和我们的 `kubeconfig` 凭证在集群外运行Operator作为Go程序。

```
WATCH_NAMESPACE=myproject make run
```

### 应用Memcached自定义资源

打开 *Terminal 1* 选项卡，导航到 `memcached-operator` 顶级目录:

```
cd projects/memcached-operator
```

在应用Memcached自定义资源之前，观察Memcached Helm Chart `values.yaml`:

[Memcached Helm Chart Values.yaml 文件](https://github.com/helm/charts/blob/master/stable/memcached/values.yaml)

更新Memcached自定义资源 `config/samples/charts_v1alpha1_memcached.yaml` ，修改以下配置值:

- `spec.replicaCount: 1`

```
apiVersion: charts.example.com/v1alpha1
kind: Memcached
metadata:
  name: memcached-sample
spec:
  # Default values copied from <project_dir>/helm-charts/memcached/values.yaml
  AntiAffinity: soft
  affinity: {}
  extraContainers: ""
  extraVolumes: ""
  image: memcached:1.5.20
  kind: StatefulSet
  memcached:
    extendedOptions: modern
    extraArgs: []
    maxItemMemory: 64
    verbosity: v
  metrics:
    enabled: false
    image: quay.io/prometheus/memcached-exporter:v0.6.0
    resources: {}
    serviceMonitor:
      enabled: false
      interval: 15s
  nodeSelector: {}
  pdbMinAvailable: 2
  podAnnotations: {}
  replicaCount: 1
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
  securityContext:
    enabled: false
    fsGroup: 1001
    runAsUser: 1001
  serviceAnnotations: {}
  tolerations: {}
  updateStrategy:
    type: RollingUpdate
```

在用我们想要的规范更新Memcached Custom Resource之后，将其应用到集群。确保你当前的作用域是 `myproject` 命名空间:

```
oc project myproject
```

```
oc apply -f config/samples/charts_v1alpha1_memcached.yaml
```

确认已创建自定义资源:

```
oc get memcached
```

环境下拉Memcached容器镜像可能需要一些时间。确认状态集已经创建:

```
oc get statefulset
```

确认Stateful Set的pod当前正在运行:

```
oc get pods
```

确认创建了Memcached的 "internal" 和 "public" ClusterIP服务:

```
oc get services
```

### 更新Memcached自定义资源

现在让我们更新Memcached `example` Custom Resource，并将期望的副本数量增加到 `5`:

```
oc patch memcached memcached-sample -p '{"spec":{"replicaCount": 3}}' --type=merge
```

验证Memcached Stateful Set正在创建两个额外的pod:

```
oc get pods
```

### 测试Memcached集群故障切换

如果任何一个Memcached成员失败，它会被重启或由Kubernetes基础设施自动重新创建，并在集群恢复时自动重新加入集群。你可以通过杀死任何一个pod来测试这个场景:

```
oc delete pods -l app.kubernetes.io/name=memcached
```

观看pod重生:

```
oc get pods -l app.kubernetes.io/name=memcached
```

### 清理

通过删除 `example` 自定义资源，删除Memcached集群和所有相关资源:

```
oc delete memcached memcached-sample
```

验证状态集、pod和服务是否被移除:

```
oc get statefulset
oc get pods
oc get services
```
