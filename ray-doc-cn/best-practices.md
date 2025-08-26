# 生产最佳实践

本节帮助您：

* 了解在生产环境中操作Serve的最佳实践
* 了解更多关于使用Serve CLI管理Serve的信息
* 在查询Serve时配置HTTP请求

## CLI最佳实践

本节总结了使用Serve CLI部署到生产环境的最佳实践：

* 使用`serve run`在本地手动测试和改进您的Serve应用程序。
* 使用`serve build`为您的Serve应用程序创建Serve配置文件。
    * 对于开发，将您的Serve应用程序代码放在远程仓库中，并手动配置Serve配置文件`runtime_env`中的`working_dir`或`py_modules`字段以指向该仓库。
    * 对于生产，将您的Serve应用程序代码放在自定义Docker镜像中而不是`runtime_env`中。请参阅[本教程](serve-custom-docker-images)了解如何创建自定义Docker镜像并在KubeRay上部署它们。
* 使用`serve status`跟踪您的Serve应用程序的健康状况和部署进度。有关更多信息，请参阅[监控指南](serve-in-production-inspecting)。
* 使用`serve config`检查您的Serve应用程序收到的最新配置。这是其目标状态。有关更多信息，请参阅[监控指南](serve-in-production-inspecting)。
* 通过修改Serve配置文件并使用`serve deploy`重新部署来进行轻量级配置更新（例如，`num_replicas`或`user_config`更改）。

## 客户端HTTP请求

这些文档中的大多数示例使用Python的`requests`库进行简单的`get`或`post`请求，例如：

```{literalinclude} ../doc_code/requests_best_practices.py
:start-after: __prototype_code_start__
:end-before: __prototype_code_end__
:language: python
```

这种模式对原型设计很有用，但对生产环境不够。在生产环境中，HTTP请求应该使用：

* 重试：请求可能偶尔由于临时问题而失败（例如，网络缓慢、节点故障、停电、流量激增等）。重试失败的请求几次以应对这些问题。
* 指数退避：为了避免在临时错误期间用重试轰炸Serve应用程序，在失败时应用指数退避。每次重试应该在运行前比前一次等待指数级更长的时间。例如，第一次重试可能在失败后等待0.1秒，后续重试在失败后等待0.4秒（4 x 0.1）、1.6秒、6.4秒、25.6秒等。
* 超时：为每次重试添加超时以防止请求挂起。超时应该比应用程序的延迟更长，以给您的应用程序足够的时间处理请求。此外，在Serve应用程序中设置[端到端超时](serve-performance-e2e-timeout)，这样慢请求就不会阻塞副本。

```{literalinclude} ../doc_code/requests_best_practices.py
:start-after: __production_code_start__
:end-before: __production_code_end__
:language: python
```

## 负载卸载

当请求发送到集群时，它首先由Serve代理接收，然后使用{mod}`DeploymentHandle <ray.serve.handle.DeploymentHandle>`将其转发给副本进行处理。
副本可以同时处理最多可配置数量的请求。使用`max_ongoing_requests`选项配置数量。
如果所有副本都忙且无法接受更多请求，请求会在{mod}`DeploymentHandle <ray.serve.handle.DeploymentHandle>`中排队，直到一个可用。

在重负载下，{mod}`DeploymentHandle <ray.serve.handle.DeploymentHandle>`队列可能会增长并导致高尾部延迟和系统上的过度负载。
为了避免不稳定，通常最好故意拒绝一些请求以避免这些队列无限增长。
这种技术称为"负载卸载"，它允许系统优雅地处理过度负载，而不会导致尾部延迟激增或使组件过载到故障点。

您可以使用{mod}`@serve.deployment <ray.serve.deployment>`装饰器的`max_queued_requests`参数为Serve部署配置负载卸载。
这控制每个{mod}`DeploymentHandle <ray.serve.handle.DeploymentHandle>`（包括Serve代理）将排队的最大请求数。
一旦达到限制，入队任何新请求都会立即引发{mod}`BackPressureError <ray.serve.exceptions.BackPressureError>`。
HTTP请求将返回`503`状态码（服务不可用）。

以下示例定义了一个模拟慢请求处理的部署，并配置了`max_ongoing_requests`和`max_queued_requests`。

```{literalinclude} ../doc_code/load_shedding.py
:start-after: __example_deployment_start__
:end-before: __example_deployment_end__
:language: python
```

要测试行为，并行发送HTTP请求以模拟多个客户端。
Serve接受`max_ongoing_requests`和`max_queued_requests`请求，并以`503`或服务不可用状态拒绝进一步请求。

```{literalinclude} ../doc_code/load_shedding.py
:start-after: __client_test_start__
:end-before: __client_test_end__
:language: python
```

```bash
2024-02-28 11:12:22,287 INFO worker.py:1744 -- Started a local Ray instance. View the dashboard at http://127.0.0.1:8265
(ProxyActor pid=21011) INFO 2024-02-28 11:12:24,088 proxy 127.0.0.1 proxy.py:1140 - Proxy actor 15b7c620e64c8c69fb45559001000000 starting on node ebc04d744a722577f3a049da12c9f83d9ba6a4d100e888e5fcfa19d9.
(ProxyActor pid=21011) INFO 2024-02-28 11:12:24,089 proxy 127.0.0.1 proxy.py:1357 - Starting HTTP server on node: ebc04d744a722577f3a049da12c9f83d9ba6a4d100e888e5fcfa19d9 listening on port 8000
(ProxyActor pid=21011) INFO:     Started server process [21011]
(ServeController pid=21008) INFO 2024-02-28 11:12:24,199 controller 21008 deployment_state.py:1614 - Deploying new version of deployment SlowDeployment in application 'default'. Setting initial target number of replicas to 1.
(ServeController pid=21008) INFO 2024-02-28 11:12:24,300 controller 21008 deployment_state.py:1924 - Adding 1 replica to deployment SlowDeployment in application 'default'.
(ProxyActor pid=21011) WARNING 2024-02-28 11:12:27,141 proxy 127.0.0.1 544437ef-f53a-4991-bb37-0cda0b05cb6a / router.py:96 - Request dropped due to backpressure (num_queued_requests=2, max_queued_requests=2).
(ProxyActor pid=21011) WARNING 2024-02-28 11:12:27,142 proxy 127.0.0.1 44dcebdc-5c07-4a92-b948-7843443d19cc / router.py:96 - Request dropped due to backpressure (num_queued_requests=2, max_queued_requests=2).
(ProxyActor pid=21011) WARNING 2024-02-28 11:12:27,143 proxy 127.0.0.1 83b444ae-e9d6-4ac6-84b7-f127c48f6ba7 / router.py:96 - Request dropped due to backpressure (num_queued_requests=2, max_queued_requests=2).
(ProxyActor pid=21011) WARNING 2024-02-28 11:12:27,144 proxy 127.0.0.1 f92b47c2-6bff-4a0d-8e5b-126d948748ea / router.py:96 - Request dropped due to backpressure (num_queued_requests=2, max_queued_requests=2).
(ProxyActor pid=21011) WARNING 2024-02-28 11:12:27,145 proxy 127.0.0.1 cde44bcc-f3e7-4652-b487-f3f2077752aa / router.py:96 - Request dropped due to backpressure (num_queued_requests=2, max_queued_requests=2).
(ServeReplica:default:SlowDeployment pid=21013) INFO 2024-02-28 11:12:28,168 default_SlowDeployment 8ey9y40a e3b77013-7dc8-437b-bd52-b4839d215212 / replica.py:373 - __CALL__ OK 2007.7ms
(ServeReplica:default:SlowDeployment pid=21013) INFO 2024-02-28 11:12:30,175 default_SlowDeployment 8ey9y40a 601e7b0d-1cd3-426d-9318-43c2c4a57a53 / replica.py:373 - __CALL__ OK 4013.5ms
(ServeReplica:default:SlowDeployment pid=21013) INFO 2024-02-28 11:12:32,183 default_SlowDeployment 8ey9y40a 0655fa12-0b44-4196-8fc5-23d31ae6fcb9 / replica.py:373 - __CALL__ OK 3987.9ms
(ServeReplica:default:SlowDeployment pid=21013) INFO 2024-02-28 11:12:34,188 default_SlowDeployment 8ey9y40a c49dee09-8de1-4e7a-8c2f-8ce3f6d8ef34 / replica.py:373 - __CALL__ OK 3960.8ms
Request finished with status code 200.
Request finished with status code 200.
Request finished with status code 200.
Request finished with status code 200.
```
