# Serve配置文件

本节应该帮助您：

- 了解Serve配置文件格式。
- 学习如何使用Serve配置在生产环境中部署和更新您的应用程序。
- 学习如何为Serve应用程序列表生成配置文件。

Serve配置是在生产环境中部署和更新应用程序的推荐方式。它允许您完全配置与Serve相关的所有内容，包括系统级组件（如代理）和应用程序级选项（如单个部署参数）（回想如何[配置Serve部署](serve-configure-deployment)）。一个主要好处是您可以通过修改Serve配置来动态更新单个部署参数，而无需重新部署或重启应用程序。

:::{tip}
如果您在VM上部署Serve，可以使用[serve deploy](serve-in-production-deploying) CLI命令使用Serve配置。如果您在Kubernetes上部署Serve，可以将Serve配置嵌入到Kubernetes中的[RayService](serve-in-production-kubernetes)自定义资源中。
:::

Serve配置是一个具有以下格式的YAML文件：

```yaml
proxy_location: ...

http_options:
  host: ...
  port: ...
  request_timeout_s: ...
  keep_alive_timeout_s: ...

grpc_options:
  port: ...
  grpc_servicer_functions: ...
  request_timeout_s: ...

logging_config:
  log_level: ...
  logs_dir: ...
  encoding: ...
  enable_access_log: ...

applications:
- name: ...
  route_prefix: ...
  import_path: ...
  runtime_env: ...
  deployments:
  - name: ...
    num_replicas: ...
    ...
  - name:
    ...
```

该文件包含`proxy_location`、`http_options`、`grpc_options`、`logging_config`和`applications`。

`proxy_location`字段配置在何处运行代理来处理集群流量。您可以将`proxy_location`设置为以下值：
- EveryNode（默认）：在集群中至少有一个副本actor的每个节点上运行代理。
- HeadOnly：仅在头节点上运行单个代理。
- Disabled：完全不运行代理。如果您仅使用部署句柄调用应用程序，请设置此值。

`http_options`如下。请注意，HTTP配置对您的Ray集群是全局的，您无法在运行时更新它。

- **`host`**：Serve HTTP代理的主机IP地址。这是可选的，可以省略。默认情况下，`host`设置为`0.0.0.0`以公开您的部署。如果您使用Kubernetes，必须将`host`设置为`0.0.0.0`以在集群外公开您的部署。
- **`port`**：Serve HTTP代理的端口。此参数是可选的，可以省略。默认情况下，端口设置为`8000`。
- **`request_timeout_s`**：允许您设置请求的端到端超时时间，在终止并在另一个副本上重试之前。默认情况下，没有请求超时。
- **`keep_alive_timeout_s`**：允许您设置HTTP代理的保持连接超时。有关更多详细信息，请参阅[这里](serve-http-guide-keep-alive-timeout)

`grpc_options`如下。请注意，gRPC配置对您的Ray集群是全局的，您无法在运行时更新它。
- **`port`**：gRPC代理监听的端口。这些是可选的设置，可以省略。默认情况下，端口设置为`9000`。
- **`grpc_servicer_functions`**：要添加到Serve gRPC代理的gRPC `add_servicer_to_server`函数的导入路径列表。servicer函数需要从Serve运行的上下文中可导入。这默认为空列表，这意味着gRPC服务器不会启动。
- **`request_timeout_s`**：允许您设置请求的端到端超时时间，在终止并在另一个副本上重试之前。默认情况下，没有请求超时。

`logging_config`是全局配置，您可以配置控制器、代理和副本日志。请注意，您还可以设置应用程序和部署级别的日志配置，这将优先于全局配置。有关更多详细信息，请参阅日志配置API[这里](../../serve/api/doc/ray.serve.schema.LoggingConfig.rst)。

以下是每个应用程序的字段：

- **`name`**：由`serve build`自动生成的每个应用程序的名称。每个应用程序的名称必须是唯一的。
- **`route_prefix`**：可以通过HTTP在指定的路由前缀调用应用程序。它默认为`/`。每个应用程序的路由前缀必须是唯一的。
- **`import_path`**：到您的顶级Serve部署的路径（或传递给`serve run`的相同路径）。最少的配置文件仅包含一个`import_path`。
- **`runtime_env`**：定义应用程序运行的环境。使用此参数打包应用程序依赖项，如`pip`包（有关支持的字段，请参阅{runtime-environments}`Runtime Environments <runtime-environments>`）。如果指定了`runtime_env`，`import_path`必须在`runtime_env`内可用。Serve配置的`runtime_env`只能在其`working_dir`和`py_modules`中使用[远程URI](remote-uris)；它不能使用本地zip文件或目录。[有关runtime env的更多详细信息](serve-runtime-env)。
- **`deployments (可选)`**：部署选项列表，允许您覆盖部署图代码中指定的`@serve.deployment`设置。此列表中的每个条目都必须包含部署`name`，该名称必须与代码中的一个匹配。如果省略此部分，Serve将使用代码中指定的参数启动图中的所有部署。请参阅如何[配置serve部署选项](serve-configure-deployment)。
- **`args`**：传递给[应用程序构建器](serve-app-builder-guide)的参数。

以下是遵循上述格式的[`Text ML Model`示例](serve-in-production-example)的配置：

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

该文件使用与`serve run`相同的`text_ml:app`导入路径，并在`deployments`列表中有两个条目用于翻译和摘要部署。两个条目都包含`name`设置和其他配置选项，如`num_replicas`。

:::{tip}
`deployments`列表中的每个单独条目都是可选的。在上面的示例配置文件中，您可以省略`Summarizer`，包括其`name`和`num_replicas`，文件仍然有效。当您部署文件时，`Summarizer`部署仍然会部署，使用应用程序代码中`@serve.deployment`装饰器设置的配置。
:::

## 使用`serve build`自动生成Serve配置

您可以使用实用工具从代码自动生成此配置文件。`serve build`命令接受应用程序的导入路径，并生成包含应用程序代码中所有部署及其参数的配置文件。调整这些参数以在生产环境中管理您的部署。

```console
$ ls
text_ml.py

$ serve build text_ml:app -o serve_config.yaml

$ ls
text_ml.py
serve_config.yaml
```

`serve_config.yaml`文件包含：

```yaml
proxy_location: EveryNode

http_options:
  host: 0.0.0.0
  port: 8000

grpc_options:
  port: 9000
  grpc_servicer_functions: []

logging_config:
  encoding: TEXT
  log_level: INFO
  logs_dir: null
  enable_access_log: true

applications:
- name: default
  route_prefix: /
  import_path: text_ml:app
  runtime_env: {}
  deployments:
  - name: Translator
    num_replicas: 1
    user_config:
      language: french
  - name: Summarizer
```

请注意，使用`serve build`时，`runtime_env`字段将始终为空，必须手动设置。在这种情况下，如果`torch`和`transformers`未全局安装，您应该在`runtime_env`中包含这两个pip包。

此外，`serve build`在其自动生成的文件中包含默认的HTTP和gRPC选项。您可以修改这些参数。

## 动态更改参数而不重启副本（`user_config`）

您可以使用`user_config`字段为部署提供结构化配置。您可以将任意JSON可序列化对象传递给YAML配置。Serve然后将其应用于所有运行中和未来的部署副本。用户配置的应用*不会*重启副本。这种部署连续性意味着您可以使用此字段动态地：
- 在不重启集群的情况下调整模型权重和版本。
- 调整模型组合图的流量分割百分比。
- 为部署配置任何功能标志、A/B测试和超参数。

要启用`user_config`功能，请实现一个`reconfigure`方法，该方法接受JSON可序列化对象（例如，字典、列表或字符串）作为其唯一参数：

```python
@serve.deployment
class Model:
    def reconfigure(self, config: Dict[str, Any]):
        self.threshold = config["threshold"]
```

如果您在创建部署时设置`user_config`（即在装饰器或Serve配置文件中），Ray Serve会在部署的`__init__`方法之后立即调用此`reconfigure`方法，并将`user_config`作为参数传递。您还可以通过使用新的`user_config`更新Serve配置文件并将其重新应用到Ray集群来触发`reconfigure`方法。有关更多信息，请参阅[就地更新](serve-inplace-updates)。

相应的YAML片段是：

```yaml
...
deployments:
    - name: Model
      user_config:
        threshold: 1.5
```
