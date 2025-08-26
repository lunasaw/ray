# 在Kubernetes上部署

本节应该帮助您：

- 了解如何安装和使用[KubeRay]操作符。
- 了解如何使用[RayService]部署Ray Serve应用程序。
- 了解如何监控和更新您的应用程序。

在Kubernetes上部署Ray Serve提供了Ray Serve的可扩展计算能力和Kubernetes的操作优势。
这种组合还允许您与可能在Kubernetes上运行的现有应用程序集成。在Kubernetes上运行时，使用来自[KubeRay]的[RayService]控制器。

> 注意：[Anyscale](https://www.anyscale.com/get-started)是一个托管的Ray解决方案，提供高可用性、高性能自动扩展、多云集群、spot实例支持等开箱即用的功能。

[RayService] CR将多节点Ray集群和在其上运行的Serve应用程序封装到单个Kubernetes清单中。
可以使用标准的`kubectl`命令来部署、升级和获取应用程序的状态。
本节介绍如何在Kubernetes上部署、监控和升级[Text ML示例](serve-in-production-example)。

## 安装KubeRay操作符

按照[KubeRay快速入门指南](kuberay-quickstart)：
* 安装`kubectl`和`Helm`
* 准备Kubernetes集群
* 部署KubeRay操作符

## 设置RayService自定义资源（CR）
一旦KubeRay控制器运行，通过创建和更新`RayService` CR来管理您的Ray Serve应用程序（[示例](https://github.com/ray-project/kuberay/blob/5b1a5a11f5df76db2d66ed332ff0802dc3bbff76/ray-operator/config/samples/ray-service.text-ml.yaml)）。

在`RayService` CR的`spec`部分下，设置以下字段：

**`serveConfigV2`**：表示Ray Serve用于部署应用程序的配置。使用`serve build`打印Serve配置并直接复制粘贴到您的[Kubernetes配置](serve-in-production-kubernetes)和`RayService` CR中。

**`rayClusterConfig`**：用`RayCluster` CR YAML文件的`spec`字段内容填充此字段。有关更多详细信息，请参阅[KubeRay配置](kuberay-config)。

:::{tip}
为了增强应用程序的可靠性，特别是在处理可能需要大量时间下载的大型依赖关系时，请考虑在镜像的Dockerfile中包含依赖关系，以便在pod启动时依赖关系立即可用。
:::

## 部署Serve应用程序

当创建`RayService`时，`KubeRay`控制器首先使用提供的配置创建Ray集群。
然后，一旦集群运行，它使用[REST API](serve-in-production-deploying)将Serve应用程序部署到集群。
控制器还创建一个Kubernetes Service，可用于将流量路由到Serve应用程序。

要查看示例，请部署[Text ML示例](serve-in-production-example)。
示例的Serve配置嵌入到[这个示例`RayService` CR](https://github.com/ray-project/kuberay/blob/5b1a5a11f5df76db2d66ed332ff0802dc3bbff76/ray-operator/config/samples/ray-service.text-ml.yaml)中。
将此CR本地保存到名为`ray-service.text-ml.yaml`的文件中：

:::{note}
- 示例`RayService`使用非常低的`numCpus`值用于演示目的。在生产环境中，为Serve应用程序提供更多资源。
在[这里](kuberay-config)了解更多关于如何配置KubeRay集群的信息。
- 如果您有必须在部署期间安装的依赖关系，可以将它们添加到部署代码的`runtime_env`中。在[这里](serve-handling-dependencies)了解更多信息。
:::

```console
$ curl -o ray-service.text-ml.yaml https://raw.githubusercontent.com/ray-project/kuberay/2ba0dd7bea387ac9df3681666bab3d622e89846c/ray-operator/config/samples/ray-service.text-ml.yaml
```

要部署示例，我们只需`kubectl apply` CR。
这会创建底层Ray集群，包括头节点和工作者节点pod（有关Ray集群的更多详细信息，请参阅[Ray集群关键概念](../../cluster/key-concepts.rst)），以及可用于查询我们应用程序的服务：

```console
$ kubectl apply -f ray-service.text-ml.yaml

$ kubectl get rayservices
NAME                SERVICE STATUS   NUM SERVE ENDPOINTS
rayservice-sample   Running          1

$ kubectl get pods
NAME                                                          READY   STATUS    RESTARTS   AGE
rayservice-sample-raycluster-7wlx2-head-hr8mg                 1/1     Running   0          XXs
rayservice-sample-raycluster-7wlx2-small-group-worker-tb8nn   1/1     Running   0          XXs

$ kubectl get services
NAME                                          TYPE        CLUSTER-IP        EXTERNAL-IP   PORT(S)                                         AGE
rayservice-sample-head-svc                    ClusterIP   None              <none>        10001/TCP,8265/TCP,6379/TCP,8080/TCP,8000/TCP   XXs
rayservice-sample-raycluster-7wlx2-head-svc   ClusterIP   None              <none>        10001/TCP,8265/TCP,6379/TCP,8080/TCP,8000/TCP   XXs
rayservice-sample-serve-svc                   ClusterIP   192.168.145.219   <none>        8000/TCP                                        XXs
```

请注意，上面的`rayservice-sample-serve-svc`是可用于向Serve应用程序发送查询的服务——这将在下一节中使用。

## 查询应用程序

一旦`RayService`运行，我们可以使用KubeRay控制器创建的服务通过HTTP查询它。
可以直接从集群内部查询此服务，但要从笔记本电脑访问它，您需要配置[Kubernetes ingress](kuberay-networking)或使用如下端口转发：

```console
$ kubectl port-forward service/rayservice-sample-serve-svc 8000
$ curl -X POST -H "Content-Type: application/json" localhost:8000/summarize_translate -d '"It was the best of times, it was the worst of times, it was the age of wisdom, it was the age of foolishness, it was the epoch of belief"'
c'était le meilleur des temps, c'était le pire des temps .
```

## 获取应用程序状态

当`RayService`运行时，`KubeRay`控制器持续监控它并将相关状态更新写入CR。
您可以使用`kubectl describe`查看应用程序的状态。
这包括集群状态、健康检查失败或重启等事件，以及由[`serve status`](serve-in-production-inspecting)报告的应用程序级状态。

```console
$ kubectl get rayservices
NAME                AGE
rayservice-sample   7s

$ kubectl describe rayservice rayservice-sample
...
Status:
  Active Service Status:
    Ray Cluster Status:
      Available Worker Replicas:  1
      Desired CPU:                2500m
      Desired GPU:                0
      Desired Memory:             4Gi
      Desired TPU:                0
      Desired Worker Replicas:    1
      Endpoints:
        Client:     10001
        Dashboard:  8265
        Metrics:    8080
        Redis:      6379
        Serve:      8000
      Head:
        Pod IP:             10.48.99.153
        Pod Name:           rayservice-sample-raycluster-7wlx2-head-dqv7t
        Service IP:         10.48.99.153
        Service Name:       rayservice-sample-raycluster-7wlx2-head-svc
      Last Update Time:     2025-04-28T06:32:13Z
      Max Worker Replicas:  5
      Min Worker Replicas:  1
      Observed Generation:  1
  Observed Generation:      1
  Pending Service Status:
    Application Statuses:
      text_ml_app:
        Health Last Update Time:  2025-04-28T06:39:02Z
        Serve Deployment Statuses:
          Summarizer:
            Health Last Update Time:  2025-04-28T06:39:02Z
            Status:                   HEALTHY
          Translator:
            Health Last Update Time:  2025-04-28T06:39:02Z
            Status:                   HEALTHY
        Status:                       RUNNING
    Ray Cluster Name:                 rayservice-sample-raycluster-7wlx2
    Ray Cluster Status:
      Desired CPU:     0
      Desired GPU:     0
      Desired Memory:  0
      Desired TPU:     0
      Head:
  Service Status:  Running
Events:
  Type    Reason   Age                      From                   Message
  ----    ------   ----                     ----                   -------
  Normal  Running  2m15s (x29791 over 16h)  rayservice-controller  The Serve applicaton is now running and healthy.
```

## 更新应用程序

要更新`RayService`，请修改清单并使用`kubectl apply`应用它。
可能发生两种类型的更新：
- *应用程序级更新*：当仅更改Serve配置选项时，更新在同一个Ray集群上*就地*应用。这启用了[轻量级更新](serve-in-production-lightweight-update)，例如向上或向下扩展部署或修改自动扩展参数。
- *集群级更新*：当更改`RayCluster`配置选项时，例如更新集群的容器镜像，可能会导致集群级更新。在这种情况下，启动新集群，并将应用程序部署到其中。一旦新集群准备就绪，Kubernetes服务就会更新以指向新集群，并终止先前的集群。应用程序不应该有任何停机时间，但请注意，这需要Kubernetes集群足够大以调度两个Ray集群。

### 示例：Serve配置更新

在上面的Text ML示例中，将Serve配置中Translator的语言更改为德语：

```yaml
  - name: Translator
    num_replicas: 1
    user_config:
      language: german
```

现在要更新应用程序，我们应用修改后的清单：

```console
$ kubectl apply -f ray-service.text-ml.yaml

$ kubectl describe rayservice rayservice-sample
...
  Serve Deployment Statuses:
    text_ml_app_Translator:
      Health Last Update Time:  2023-09-07T18:21:36Z
      Last Update Time:         2023-09-07T18:21:36Z
      Status:                   UPDATING
...
```

查询应用程序以查看德语的不同翻译：

```console
$ curl -X POST -H "Content-Type: application/json" localhost:8000/summarize_translate -d '"It was the best of times, it was the worst of times, it was the age of wisdom, it was the age of foolishness, it was the epoch of belief"'
Es war die beste Zeit, es war die schlimmste Zeit .
```

### 更新RayCluster配置

更新RayCluster配置的过程与更新Serve配置相同。
例如，我们可以在清单中将工作者节点数更新为2：

```console
workerGroupSpecs:
  # 工作者组中的pod数量。
  - replicas: 2
```

```console
$ kubectl apply -f ray-service.text-ml.yaml

$ kubectl describe rayservice rayservice-sample
...
  pendingServiceStatus:
    appStatus: {}
    dashboardStatus:
      healthLastUpdateTime: "2022-07-18T21:54:53Z"
      lastUpdateTime: "2022-07-18T21:54:54Z"
    rayClusterName: rayservice-sample-raycluster-bshfr
    rayClusterStatus: {}
...
```

在状态中，您可以看到`RayService`正在准备一个待处理的集群。
待处理集群健康后，它成为活动集群，先前的集群被终止。

## 自动扩展
您可以通过在Serve配置中设置自动扩展字段来为Serve应用程序配置自动扩展。在[Serve自动扩展指南](serve-autoscaling)中了解有关配置选项的更多信息。

要在KubeRay集群中启用自动扩展，您需要将`enableInTreeAutoscaling`设置为True。此外，还有其他选项可用于配置自动扩展行为。有关更多详细信息，请参阅[这里](serve-autoscaling)的文档。

:::{note}
在大多数用例中，建议启用Kubernetes自动扩展以充分利用集群中的资源。如果您使用GKE，可以利用AutoPilot Kubernetes集群。有关说明，请参阅[创建AutoPilot集群](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-an-autopilot-cluster)。对于EKS，您可以通过利用集群自动扩展器来启用Kubernetes集群自动扩展。有关详细信息，请参阅[AWS上的集群自动扩展器](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/README.md)。要了解Kubernetes自动扩展和Ray自动扩展之间的关系，请参阅[使用Kubernetes集群自动扩展器的Ray自动扩展器](kuberay-autoscaler-with-ray-autoscaler)。
:::

## 负载均衡器
设置ingress以使用负载均衡器公开您的Serve应用程序。请参阅[此配置](https://github.com/ray-project/kuberay/blob/v1.0.0/ray-operator/config/samples/ray-service-alb-ingress.yaml)

:::{note}
- Ray Serve在每个节点上运行HTTP代理，允许您使用`/-/routes`作为节点健康检查的端点。
- Ray Serve使用端口8000作为默认HTTP代理流量端口。您可以通过在Serve配置中设置`http_options`来更改端口。在[这里](serve-multi-application)了解更多详细信息。
:::

## 监控
使用Ray Dashboard监控您的Serve应用程序。
- 在[这里](observability-configure-manage-dashboard)了解更多关于如何配置和管理Dashboard的信息。
- 在[这里](serve-monitoring)了解Ray Serve Dashboard。
- 了解如何为Dashboard设置[Prometheus](prometheus-setup)和[Grafana](grafana)。
- 了解[Ray Serve日志](serve-logging)以及如何在Kubernetes上[持久化日志](persist-kuberay-custom-resource-logs)。

:::{note}
- 要排查Serve中应用程序部署失败的问题，您可以通过运行`kubectl logs -f <kuberay-operator-pod-name>`（例如，`kubectl logs -f kuberay-operator-7447d85d58-lv7pf`）来检查KubeRay操作符日志。KubeRay操作符日志包含有关Serve应用程序部署事件和Serve应用程序健康检查的信息。
- 您还可以检查控制器日志和部署日志，它们位于头节点pod和工作者节点pod中的`/tmp/ray/session_latest/logs/serve/`下。这些日志包含有关特定部署失败原因和自动扩展事件的信息。
:::

## 下一步

请参阅[添加端到端容错](serve-e2e-ft)以了解更多关于Serve的故障条件和如何防范它们的信息。

[KubeRay]: kuberay-quickstart
[RayService]: kuberay-rayservice-quickstart
