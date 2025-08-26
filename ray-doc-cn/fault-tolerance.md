# 添加端到端容错

本节帮助您：

* 为您的Serve应用程序提供额外的容错能力
* 了解Serve的恢复程序
* 在您的Serve应用程序中模拟系统错误

:::{admonition} 相关指南
:class: seealso
本节讨论来自以下内容的概念：
* Serve的[架构指南](serve-architecture)
* Serve的[Kubernetes生产指南](serve-in-production-kubernetes)
:::

## 指南：为您的Serve应用程序提供端到端容错

Serve提供了一些开箱即用的[容错](serve-ft-detail)功能。获得端到端容错的两个选项如下：
* 调整这些功能并在[KubeRay]上运行Serve
* 使用[Anyscale平台](https://docs.anyscale.com/platform/services/head-node-ft?utm_source=ray_docs&utm_medium=docs&utm_campaign=tolerance)，一个托管的Ray平台

### 副本健康检查

默认情况下，Serve控制器定期对每个Serve部署副本进行健康检查，并在失败时重启它。

您可以定义自定义应用程序级健康检查并调整其频率和超时时间。
要定义自定义健康检查，请在部署类中添加`check_health`方法。
此方法不应接受任何参数且不返回结果，如果Ray Serve认为副本不健康，它应该引发异常。
如果健康检查失败，Serve控制器会记录异常，杀死不健康的副本，并重启它们。
您还可以使用部署选项来自定义Serve运行健康检查的频率以及Serve将副本标记为不健康后的超时时间。

```{literalinclude} ../doc_code/fault_tolerance/replica_health_check.py
:start-after: __health_check_start__
:end-before: __health_check_end__
:language: python
```

在此示例中，如果与外部数据库的连接丢失，`check_health`会引发错误。Serve控制器定期在部署的每个副本上调用此方法。如果方法为副本引发异常，Serve会将该副本标记为不健康并重启它。健康检查是按副本配置和执行的。

:::{note}
您不应该通过部署句柄直接调用`check_health`（例如，`await deployment_handle.check_health.remote()`）。这将在单个任意副本上调用健康检查。`check_health`方法设计为Serve控制器的接口，而不是用于直接用户调用。
:::

:::{note}
在可组合的部署图中，每个部署负责自己的健康，独立于它绑定的其他部署。例如，在由`app = ParentDeployment.bind(ChildDeployment.bind())`定义的应用程序中，如果`ChildDeployment`副本的健康检查失败，`ParentDeployment`不会重启。当`ChildDeployment`副本恢复时，`ParentDeployment`中的句柄会自动更新以将请求路由到健康的副本。
:::

### 工作者节点恢复

:::{admonition} 需要KubeRay
:class: caution, dropdown
您**必须**使用[KubeRay]部署Serve应用程序才能使用此功能。

请参阅Serve的[Kubernetes生产指南](serve-in-production-kubernetes)了解如何使用KubeRay部署应用程序。
:::

默认情况下，Serve可以从某些故障中恢复，例如不健康的actor。当[Serve在Kubernetes上运行](serve-in-production-kubernetes)并使用[KubeRay]时，它还可以从一些集群级故障中恢复，例如死亡的工作者或头节点。

当工作者节点失败时，在其上运行的actor也会失败。Serve检测到actor已失败，并尝试在剩余的健康节点上重新生成actor。同时，KubeRay检测到节点本身已失败，因此它尝试在另一个运行节点上重启工作者pod，并启动一个新的健康节点来替换它。一旦节点启动，如果pod仍然待处理，可以在该节点上重启它。类似地，Serve也可以在该节点上重新生成任何待处理的actor。在健康节点上运行的部署副本可以在整个恢复期间继续处理流量。

### 头节点恢复：Ray GCS容错

:::{admonition} 需要KubeRay
:class: caution, dropdown
您**必须**使用[KubeRay]部署Serve应用程序才能使用此功能。

请参阅Serve的[Kubernetes生产指南](serve-in-production-kubernetes)了解如何使用KubeRay部署应用程序。
:::

在本节中，您将学习如何为Ray的全局控制存储（GCS）添加容错，这允许您的Serve应用程序即使在头节点崩溃时也能处理流量。

默认情况下，Ray头节点是单点故障：如果它崩溃，整个Ray集群就会崩溃，您必须重启它。在Kubernetes上运行时，`RayService`控制器会健康检查Ray集群，如果发生这种情况会重启它，但这会引入一些停机时间。

从Ray 2.0+开始，KubeRay支持[全局控制存储（GCS）容错](kuberay-gcs-ft)，防止Ray集群在头节点宕机时崩溃。
当头节点恢复时，Serve应用程序仍然可以使用工作者节点处理流量，但您无法更新或从其他故障（如Actor或工作者节点崩溃）中恢复。
一旦GCS恢复，集群就会恢复正常行为。

您可以通过添加外部Redis服务器并修改`RayService` Kubernetes对象来在KubeRay上启用GCS容错，步骤如下：

#### 步骤1：添加外部Redis服务器

GCS容错需要外部Redis数据库。您可以选择托管自己的Redis数据库，或通过第三方供应商使用一个。使用高可用的Redis数据库以提高弹性。

**出于开发目的**，您也可以在Ray集群所在的同一Kubernetes集群上托管一个小型Redis数据库。例如，您可以通过在Kubernetes YAML前面添加这三个Redis对象来添加1节点Redis集群：

```YAML
kind: ConfigMap
apiVersion: v1
metadata:
  name: redis-config
  labels:
    app: redis
data:
  redis.conf: |-
    port 6379
    bind 0.0.0.0
    protected-mode no
    requirepass 5241590000000000
---
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: redis
spec:
  type: ClusterIP
  ports:
    - name: redis
      port: 6379
  selector:
    app: redis
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    app: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
        - name: redis
          image: redis:5.0.8
          command:
            - "sh"
            - "-c"
            - "redis-server /usr/local/etc/redis/redis.conf"
          ports:
            - containerPort: 6379
          volumeMounts:
            - name: config
              mountPath: /usr/local/etc/redis/redis.conf
              subPath: redis.conf
      volumes:
        - name: config
          configMap:
            name: redis-config
---
```

**此配置不适合生产环境**，但对开发和测试很有用。当您转向生产环境时，强烈建议您用高可用的Redis集群替换这个1节点Redis集群。

#### 步骤2：将Redis信息添加到RayService

添加Redis对象后，您还需要修改`RayService`配置。

首先，您需要更新`RayService`元数据的注释：

::::{tab-set}

:::{tab-item} 普通配置
```yaml
...
apiVersion: ray.io/v1alpha1
kind: RayService
metadata:
  name: rayservice-sample
spec:
...
```
:::

:::{tab-item} 容错配置
:selected:
```yaml
...
apiVersion: ray.io/v1alpha1
kind: RayService
metadata:
  name: rayservice-sample
  annotations:
    ray.io/ft-enabled: "true"
    ray.io/external-storage-namespace: "my-raycluster-storage-namespace"
spec:
...
```
:::

::::

注释包括：
* `ray.io/ft-enabled` 必需：当为true时启用GCS容错
* `ray.io/external-storage-namespace` 可选：设置[外部存储命名空间]

接下来，您需要在`headGroupSpec`中添加`RAY_REDIS_ADDRESS`环境变量：

::::{tab-set}

:::{tab-item} 普通配置

```yaml
apiVersion: ray.io/v1alpha1
kind: RayService
metadata:
    ...
spec:
    ...
    rayClusterConfig:
        headGroupSpec:
            ...
            template:
                ...
                spec:
                    ...
                    env:
                        ...
```

:::

:::{tab-item} 容错配置
:selected:

```yaml
apiVersion: ray.io/v1alpha1
kind: RayService
metadata:
    ...
spec:
    ...
    rayClusterConfig:
        headGroupSpec:
            ...
            template:
                ...
                spec:
                    ...
                    env:
                        ...
                        - name: RAY_REDIS_ADDRESS
                          value: redis:6379
```
:::

::::

`RAY_REDIS_ADDRESS`的值应该是您的Redis数据库的`redis://`地址。它应该包含您的Redis数据库的主机和端口。[示例Redis地址](https://www.iana.org/assignments/uri-schemes/prov/rediss)是`redis://user:secret@localhost:6379/0?foo=bar&qux=baz`。

在上面的示例中，Redis部署名称（`redis`）是Kubernetes集群内的主机，Redis端口是`6379`。该示例与前一节的[示例配置](one-node-redis-example)兼容。

在您应用Redis对象以及更新的`RayService`后，您的Ray集群可以从头节点崩溃中恢复，而无需重启所有工作者！

:::{seealso}
查看KubeRay关于[GCS容错](kuberay-gcs-ft)的指南，了解Serve如何利用外部Redis集群提供头节点容错的更多信息。
:::

### 在节点间分散副本

提高Serve应用程序可用性的一种方法是将部署副本分散到多个节点上，这样即使在特定数量的节点故障后，您仍然有足够的运行副本来处理流量。

默认情况下，Serve软分散所有部署副本，但它有一些限制：

* 分散是软的，尽最大努力，不保证完全均匀。

* Serve尝试在可能的情况下在现有节点间分散副本，而不是启动新节点。
例如，如果您有一个足够大的单节点集群，Serve会在该单节点上调度所有副本，假设它有足够的资源。但是，该节点成为单点故障。

您可以使用`max_replicas_per_node`[部署选项](../../serve/api/doc/ray.serve.deployment_decorator.rst)来更改部署的分散行为，它硬限制给定部署的副本数量可以在单个节点上运行。
如果您将其设置为1，那么您实际上是在严格分散部署副本。如果您不设置它，则没有硬分散约束，Serve使用前面段落中提到的默认软分散。`max_replicas_per_node`选项是按部署的，只影响部署内副本的分散。不同部署的副本之间没有分散。

以下代码示例显示如何设置`max_replicas_per_node`部署选项：

```{testcode}
import ray
from ray import serve

@serve.deployment(max_replicas_per_node=1)
class Deployment1:
  def __call__(self, request):
    return "hello"

@serve.deployment(max_replicas_per_node=2)
class Deployment2:
  def __call__(self, request):
    return "world"
```

此示例有两个具有不同`max_replicas_per_node`的Serve部署：`Deployment1`在每个节点上最多可以有一个副本，`Deployment2`在每个节点上最多可以有两个副本。如果您调度`Deployment1`的两个副本和`Deployment2`的两个副本，Serve运行一个至少有两个节点的集群，每个节点运行一个`Deployment1`副本。`Deployment2`的两个副本可能在单个节点上运行或跨两个节点运行，因为两者都满足`max_replicas_per_node`约束。

## Serve的恢复程序

本节解释Serve如何从系统故障中恢复。它使用以下Serve应用程序和配置作为工作示例。

::::{tab-set}

:::{tab-item} Python代码
```{literalinclude} ../doc_code/fault_tolerance/sleepy_pid.py
:start-after: __start__
:end-before: __end__
:language: python
```
:::

:::{tab-item} Kubernetes配置
```{literalinclude} ../doc_code/fault_tolerance/k8s_config.yaml
:language: yaml
```
:::

::::

按照[KubeRay快速入门指南](kuberay-quickstart)：
* 安装`kubectl`和`Helm`
* 准备Kubernetes集群
* 部署KubeRay操作符

然后，[部署Serve应用程序](serve-deploy-app-on-kuberay)：

```console
$ kubectl apply -f config.yaml
```

### 工作者节点故障

您可以在工作示例中模拟工作者节点故障。首先，查看Kubernetes集群中运行的节点和pod：

```console
$ kubectl get nodes

NAME                                        STATUS   ROLES    AGE     VERSION
gke-serve-demo-default-pool-ed597cce-nvm2   Ready    <none>   3d22h   v1.22.12-gke.1200
gke-serve-demo-default-pool-ed597cce-m888   Ready    <none>   3d22h   v1.22.12-gke.1200
gke-serve-demo-default-pool-ed597cce-pu2q   Ready    <none>   3d22h   v1.22.12-gke.1200

$ kubectl get pods -o wide

NAME                                                      READY   STATUS    RESTARTS        AGE    IP           NODE                                        NOMINATED NODE   READINESS GATES
ervice-sample-raycluster-thwmr-worker-small-group-bdv6q   1/1     Running   0               3m3s   10.68.2.62   gke-serve-demo-default-pool-ed597cce-nvm2   <none>           <none>
ervice-sample-raycluster-thwmr-worker-small-group-pztzk   1/1     Running   0               3m3s   10.68.2.61   gke-serve-demo-default-pool-ed597cce-m888   <none>           <none>
rayservice-sample-raycluster-thwmr-head-28mdh             1/1     Running   1 (2m55s ago)   3m3s   10.68.0.45   gke-serve-demo-default-pool-ed597cce-pu2q   <none>           <none>
redis-75c8b8b65d-4qgfz                                    1/1     Running   0               3m3s   10.68.2.60   gke-serve-demo-default-pool-ed597cce-nvm2   <none>           <none>
```

打开一个单独的终端窗口并端口转发到其中一个工作者节点：

```console
$ kubectl port-forward ervice-sample-raycluster-thwmr-worker-small-group-bdv6q 8000
Forwarding from 127.0.0.1:8000 -> 8000
Forwarding from [::1]:8000 -> 8000
```

当`port-forward`运行时，您可以在另一个终端窗口中查询应用程序：

```console
$ curl localhost:8000
418
```

输出是处理请求的部署副本的进程ID。应用程序启动6个部署副本，所以如果您多次运行查询，您应该看到不同的进程ID：

```console
$ curl localhost:8000
418
$ curl localhost:8000
256
$ curl localhost:8000
385
```

现在您可以模拟工作者故障。您有两个选项：杀死工作者pod或杀死工作者节点。让我们从工作者pod开始。确保杀死您**没有**端口转发的pod，这样您可以在另一个重启时继续查询活着的工作者。

```console
$ kubectl delete pod ervice-sample-raycluster-thwmr-worker-small-group-pztzk
pod "ervice-sample-raycluster-thwmr-worker-small-group-pztzk" deleted

$ curl localhost:8000
6318
```

当pod崩溃并恢复时，活着的pod可以继续处理流量！

:::{tip}
杀死节点并等待它恢复通常比杀死pod并等待它恢复需要更长时间。对于这种类型的调试，通过杀死pod级别而不是节点级别来模拟故障更快。
:::

您可以类似地杀死工作者节点并看到其他节点可以继续处理流量：

```console
$ kubectl get pods -o wide

NAME                                                      READY   STATUS    RESTARTS      AGE     IP           NODE                                        NOMINATED NODE   READINESS GATES
ervice-sample-raycluster-thwmr-worker-small-group-bdv6q   1/1     Running   0             65m     10.68.2.62   gke-serve-demo-default-pool-ed597cce-nvm2   <none>           <none>
ervice-sample-raycluster-thwmr-worker-small-group-mznwq   1/1     Running   0             5m46s   10.68.1.3    gke-serve-demo-default-pool-ed597cce-m888   <none>           <none>

$ kubectl delete node gke-serve-demo-default-pool-ed597cce-m888
node "gke-serve-demo-default-pool-ed597cce-m888" deleted

$ curl localhost:8000
385
```

### 头节点故障

您可以通过杀死头pod或头节点来模拟头节点故障。首先，查看集群中运行的pod：

```console
$ kubectl get pods -o wide

NAME                                                      READY   STATUS    RESTARTS      AGE     IP           NODE                                        NOMINATED NODE   READINESS GATES
ervice-sample-raycluster-thwmr-worker-small-group-6f2pk   1/1     Running   0             6m59s   10.68.2.64   gke-serve-demo-default-pool-ed597cce-nvm2   <none>           <none>
ervice-sample-raycluster-thwmr-worker-small-group-bdv6q   1/1     Running   0             79m     10.68.2.62   gke-serve-demo-default-pool-ed597cce-nvm2   <none>           <none>
rayservice-sample-raycluster-thwmr-head-28mdh             1/1     Running   1 (79m ago)   79m     10.68.0.45   gke-serve-demo-default-pool-ed597cce-pu2q   <none>           <none>
redis-75c8b8b65d-4qgfz                                    1/1     Running   0             79m     10.68.2.60   gke-serve-demo-default-pool-ed597cce-nvm2   <none>           <none>
```

端口转发到您的工作者pod之一。确保此pod与头节点在单独的节点上，这样您可以在不崩溃工作者的情况下杀死头节点：

```console
$ kubectl port-forward ervice-sample-raycluster-thwmr-worker-small-group-bdv6q
Forwarding from 127.0.0.1:8000 -> 8000
Forwarding from [::1]:8000 -> 8000
```

在单独的终端中，您可以向Serve应用程序发出请求：

```console
$ curl localhost:8000
418
```

您可以杀死头pod来模拟杀死Ray头节点：

```console
$ kubectl delete pod rayservice-sample-raycluster-thwmr-head-28mdh
pod "rayservice-sample-raycluster-thwmr-head-28mdh" deleted

$ curl localhost:8000
```

如果您在集群上配置了[GCS容错](serve-e2e-ft-guide-gcs)，当头pod崩溃并恢复时，您的工作者pod可以继续处理流量而无需重启。没有GCS容错，当头pod崩溃时KubeRay会重启所有工作者pod，所以您需要等待工作者重启和部署重新初始化，然后才能端口转发并发送更多请求。

### Serve控制器故障

您可以通过手动杀死Serve actor来模拟Serve控制器故障。

如果您运行KubeRay，`exec`到您的一个pod中：

```console
$ kubectl get pods

NAME                                                      READY   STATUS    RESTARTS   AGE
ervice-sample-raycluster-mx5x6-worker-small-group-hfhnw   1/1     Running   0          118m
ervice-sample-raycluster-mx5x6-worker-small-group-nwcpb   1/1     Running   0          118m
rayservice-sample-raycluster-mx5x6-head-bqjhw             1/1     Running   0          118m
redis-75c8b8b65d-4qgfz                                    1/1     Running   0          3h36m

$ kubectl exec -it rayservice-sample-raycluster-mx5x6-head-bqjhw -- bash
ray@rayservice-sample-raycluster-mx5x6-head-bqjhw:~$
```

您可以使用[Ray状态API](state-api-cli-ref)来检查您的Serve应用程序：

```console
$ ray summary actors

======== Actors Summary: 2022-10-04 21:06:33.678706 ========
Stats:
------------------------------------
total_actors: 10


Table (group by class):
------------------------------------
    CLASS_NAME              STATE_COUNTS
0   ProxyActor          ALIVE: 3
1   ServeReplica:SleepyPid  ALIVE: 6
2   ServeController         ALIVE: 1

$ ray list actors --filter "class_name=ServeController"

======== List: 2022-10-04 21:09:14.915881 ========
Stats:
------------------------------
Total: 1

Table:
------------------------------
    ACTOR_ID                          CLASS_NAME       STATE    NAME                      PID
 0  70a718c973c2ce9471d318f701000000  ServeController  ALIVE    SERVE_CONTROLLER_ACTOR  48570
```

然后您可以通过Python解释器杀死Serve控制器。请注意，您需要使用`ray list actor`输出中的`NAME`来获取Serve控制器的句柄。

```console
$ python

>>> import ray
>>> controller_handle = ray.get_actor("SERVE_CONTROLLER_ACTOR", namespace="serve")
>>> ray.kill(controller_handle, no_restart=True)
>>> exit()
```

您可以使用Ray状态API检查控制器的状态：

```console
$ ray list actors --filter "class_name=ServeController"

======== List: 2022-10-04 21:36:37.157754 ========
Stats:
------------------------------
Total: 2

Table:
------------------------------
    ACTOR_ID                          CLASS_NAME       STATE    NAME                      PID
 0  3281133ee86534e3b707190b01000000  ServeController  ALIVE    SERVE_CONTROLLER_ACTOR  49914
 1  70a718c973c2ce9471d318f701000000  ServeController  DEAD     SERVE_CONTROLLER_ACTOR  48570
```

当控制器恢复时，您应该仍然能够查询您的部署：

```
# 如果您运行KubeRay，您
# 可以从pod内部执行此操作：

$ python

>>> import requests
>>> requests.get("http://localhost:8000").json()
347
```

:::{note}
当控制器死亡时，副本健康检查和部署自动扩展将不工作。一旦控制器恢复，它们将继续工作。
:::

### 部署副本故障

您可以通过手动杀死部署副本来模拟副本故障。如果您运行KubeRay，在运行这些命令之前确保`exec`到Ray pod中。

```console
$ ray summary actors

======== Actors Summary: 2022-10-04 21:40:36.454488 ========
Stats:
------------------------------------
total_actors: 11


Table (group by class):
------------------------------------
    CLASS_NAME              STATE_COUNTS
0   ProxyActor          ALIVE: 3
1   ServeController         ALIVE: 1
2   ServeReplica:SleepyPid  ALIVE: 6

$ ray list actors --filter "class_name=ServeReplica:SleepyPid"

======== List: 2022-10-04 21:41:32.151864 ========
Stats:
------------------------------
Total: 6

Table:
------------------------------
    ACTOR_ID                          CLASS_NAME              STATE    NAME                               PID
 0  39e08b172e66a5d22b2b4cf401000000  ServeReplica:SleepyPid  ALIVE    SERVE_REPLICA::SleepyPid#RlRptP    203
 1  55d59bcb791a1f9353cd34e301000000  ServeReplica:SleepyPid  ALIVE    SERVE_REPLICA::SleepyPid#BnoOtj    348
 2  8c34e675edf7b6695461d13501000000  ServeReplica:SleepyPid  ALIVE    SERVE_REPLICA::SleepyPid#SakmRM    283
 3  a95405318047c5528b7483e701000000  ServeReplica:SleepyPid  ALIVE    SERVE_REPLICA::SleepyPid#rUigUh    347
 4  c531188fede3ebfc868b73a001000000  ServeReplica:SleepyPid  ALIVE    SERVE_REPLICA::SleepyPid#gbpoFe    383
 5  de8dfa16839443f940fe725f01000000  ServeReplica:SleepyPid  ALIVE    SERVE_REPLICA::SleepyPid#PHvdJW    176
```

您可以使用`ray list actor`输出中的`NAME`来获取其中一个副本的句柄：

```console
$ python

>>> import ray
>>> replica_handle = ray.get_actor("SERVE_REPLICA::SleepyPid#RlRptP", namespace="serve")
>>> ray.kill(replica_handle, no_restart=True)
>>> exit()
```

当副本重启时，其他副本可以继续处理请求。最终副本重启并继续处理请求：

```console
$ python

>>> import requests
>>> requests.get("http://localhost:8000").json()
383
```

### 代理故障

您可以通过手动杀死`ProxyActor` actor来模拟代理故障。如果您运行KubeRay，在运行这些命令之前确保`exec`到Ray pod中。

```console
$ ray summary actors

======== Actors Summary: 2022-10-04 21:51:55.903800 ========
Stats:
------------------------------------
total_actors: 12


Table (group by class):
------------------------------------
    CLASS_NAME              STATE_COUNTS
0   ProxyActor          ALIVE: 3
1   ServeController         ALIVE: 1
2   ServeReplica:SleepyPid  ALIVE: 6

$ ray list actors --filter "class_name=ProxyActor"

======== List: 2022-10-04 21:52:39.853758 ========
Stats:
------------------------------
Total: 3

Table:
------------------------------
    ACTOR_ID                          CLASS_NAME      STATE    NAME                                                                                                 PID
 0  283fc11beebb6149deb608eb01000000  ProxyActor  ALIVE    SERVE_CONTROLLER_ACTOR:SERVE_PROXY_ACTOR-91f9a685e662313a0075efcb7fd894249a5bdae7ee88837bea7985a0    101
 1  2b010ce28baeff5cb6cb161e01000000  ProxyActor  ALIVE    SERVE_CONTROLLER_ACTOR:SERVE_PROXY_ACTOR-cc262f3dba544a49ea617d5611789b5613f8fe8c86018ef23c0131eb    133
 2  7abce9dd241b089c1172e9ca01000000  ProxyActor  ALIVE    SERVE_CONTROLLER_ACTOR:SERVE_PROXY_ACTOR-7589773fc62e08c2679847aee9416805bbbf260bee25331fa3389c4f    267
```

您可以使用`ray list actor`输出中的`NAME`来获取其中一个副本的句柄：

```console
$ python

>>> import ray
>>> proxy_handle = ray.get_actor("SERVE_CONTROLLER_ACTOR:SERVE_PROXY_ACTOR-91f9a685e662313a0075efcb7fd894249a5bdae7ee88837bea7985a0", namespace="serve")
>>> ray.kill(proxy_handle, no_restart=False)
>>> exit()
```

当代理重启时，其他代理可以继续接受请求。最终代理重启并继续接受请求。您可以使用`ray list actor`命令查看代理何时重启：

```console
$ ray list actors --filter "class_name=ProxyActor"

======== List: 2022-10-04 21:58:41.193966 ========
Stats:
------------------------------
Total: 3

Table:
------------------------------
    ACTOR_ID                          CLASS_NAME      STATE    NAME                                                                                                 PID
 0  283fc11beebb6149deb608eb01000000  ProxyActor  ALIVE     SERVE_CONTROLLER_ACTOR:SERVE_PROXY_ACTOR-91f9a685e662313a0075efcb7fd894249a5bdae7ee88837bea7985a0  57317
 1  2b010ce28baeff5cb6cb161e01000000  ProxyActor  ALIVE    SERVE_CONTROLLER_ACTOR:SERVE_PROXY_ACTOR-cc262f3dba544a49ea617d5611789b5613f8fe8c86018ef23c0131eb    133
 2  7abce9dd241b089c1172e9ca01000000  ProxyActor  ALIVE    SERVE_CONTROLLER_ACTOR:SERVE_PROXY_ACTOR-7589773fc62e08c2679847aee9416805bbbf260bee25331fa3389c4f    267
```

请注意，第一个ProxyActor的PID已经改变，表明它已重启。

[KubeRay]: kuberay-index
[external storage namespace]: kuberay-external-storage-namespace
