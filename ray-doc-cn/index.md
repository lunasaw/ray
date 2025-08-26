# 生产指南

```{toctree}
:hidden:

config
kubernetes
docker
fault-tolerance
handling-dependencies
best-practices
```

在生产环境中运行Ray Serve的推荐方式是在Kubernetes上使用[KubeRay](kuberay-quickstart) [RayService](kuberay-rayservice-quickstart)自定义资源。
RayService自定义资源自动处理重要的生产需求，如健康检查、状态报告、故障恢复和升级。
如果您不在Kubernetes上运行，也可以使用Serve CLI直接在Ray集群上运行Ray Serve。

本节将指导您快速了解如何生成Serve配置文件并使用Serve CLI部署它。
有关更多详细信息，您可以查看生产指南中的其他页面：
- 了解[Serve配置文件格式](serve-in-production-config-file)。
- 了解如何[使用KubeRay在Kubernetes上部署](serve-in-production-kubernetes)。
- 了解如何[监控运行中的Serve应用程序](serve-monitoring)。

有关在VM而不是Kubernetes上部署的信息，请参阅[在VM上部署](serve-in-production-deploying)。

## 工作示例：文本摘要和翻译应用程序

在整个生产指南中，我们将使用以下Serve应用程序作为工作示例。
该应用程序接收英文文本字符串，然后将其摘要并翻译成法语（默认）、德语或罗马尼亚语。

```{literalinclude} ../doc_code/production_guide/text_ml.py
:language: python
:start-after: __example_start__
:end-before: __example_end__
```

将此代码本地保存为`text_ml.py`。
在开发中，我们可能会使用`serve run`命令来迭代运行、开发和重复（有关更多信息，请参阅[开发工作流程](serve-dev-workflow)）。
当我们准备投入生产时，我们将生成一个结构化的[配置文件](serve-in-production-config-file)，作为应用程序的单一真实来源。

可以使用`serve build`生成此配置文件：
```
$ serve build text_ml:app -o serve_config.yaml
```

生成的文件版本包含应用程序中每个部署的`import_path`、`runtime_env`和配置选项。
应用程序需要`torch`和`transformers`包，因此修改生成配置的`runtime_env`字段以包含这两个pip包。将此配置本地保存为`serve_config.yaml`。

```yaml
proxy_location: EveryNode

http_options:
  host: 0.0.0.0
  port: 8000

applications:
- name: default
  route_prefix: /
  import_path: text_ml:app
  runtime_env:
    pip:
      - torch
      - transformers
  deployments:
  - name: Translator
    num_replicas: 1
    user_config:
      language: french
  - name: Summarizer
    num_replicas: 1
```

您可以使用`serve deploy`将应用程序部署到本地Ray集群，并使用`serve status`获取运行时状态：

```console
# 启动本地Ray集群。
ray start --head

# 将Text ML应用程序部署到本地Ray集群。
serve deploy serve_config.yaml
2022-08-16 12:51:22,043 SUCC scripts.py:180 --
部署请求发送成功！
 * 使用 `serve status` 检查部署状态。
 * 使用 `serve config` 查看运行中的应用程序配置。

$ serve status
proxies:
  cef533a072b0f03bf92a6b98cb4eb9153b7b7c7b7f15954feb2f38ec: HEALTHY
applications:
  default:
    status: RUNNING
    message: ''
    last_deployed_time_s: 1694041157.2211847
    deployments:
      Translator:
        status: HEALTHY
        replica_states:
          RUNNING: 1
        message: ''
      Summarizer:
        status: HEALTHY
        replica_states:
          RUNNING: 1
        message: ''
```

使用Python `requests`测试应用程序：

```{literalinclude} ../doc_code/production_guide/text_ml.py
:language: python
:start-after: __start_client__
:end-before: __end_client__
```

要更新应用程序，请修改配置文件并再次使用`serve deploy`。

## 下一步

要深入了解如何部署、更新和监控Serve应用程序，请参阅以下页面：
- 了解[Serve配置文件格式](serve-in-production-config-file)的详细信息。
- 了解如何[使用KubeRay在Kubernetes上部署](serve-in-production-kubernetes)。
- 了解如何[构建自定义Docker镜像](serve-custom-docker-images)以与KubeRay一起使用。
- 了解如何[监控运行中的Serve应用程序](serve-monitoring)。

[KubeRay]: kuberay-index
[RayService]: kuberay-rayservice-quickstart
