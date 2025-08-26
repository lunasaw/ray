# 自定义Docker镜像

本节帮助您：

* 使用您自己的依赖关系扩展官方Ray Docker镜像
* 在自定义Docker镜像中打包您的Serve应用程序，而不是使用`runtime_env`
* 在KubeRay中使用自定义Docker镜像

要遵循本教程，请确保安装[Docker Desktop](https://docs.docker.com/engine/install/)并创建一个[Dockerhub](https://hub.docker.com/)账户，您可以在其中托管自定义Docker镜像。

## 工作示例

创建一个名为`fake.py`的Python文件，并将以下Serve应用程序保存到其中：

```{literalinclude} ../doc_code/fake_email_creator.py
:start-after: __fake_start__
:end-before: __fake_end__
:language: python
```

此应用程序创建并返回一个假电子邮件地址。它依赖于[Faker包](https://github.com/joke2k/faker)来创建假电子邮件地址。在本地安装`Faker`包以运行它：

```console
% pip install Faker==18.13.0

...

% serve run fake:app

...

# 在另一个终端窗口中：
% curl localhost:8000
john24@example.org
```

本教程解释了如何将代码打包并在自定义Docker镜像中提供服务。

## 扩展Ray Docker镜像

[rayproject](https://hub.docker.com/u/rayproject)组织维护包含运行Ray所需依赖关系的Docker镜像。实际上，[rayproject/ray](https://hub.docker.com/r/rayproject/ray)仓库托管了本文档的Docker镜像。例如，[这个RayService配置](https://github.com/ray-project/kuberay/blob/release-1.1.0/ray-operator/config/samples/ray-service.sample.yaml)使用由`rayproject/ray`托管的[rayproject/ray:2.9.0](https://hub.docker.com/layers/rayproject/ray/2.9.0/images/sha256-e64546fb5c3233bb0f33608e186e285c52cdd7440cae1af18f7fcde1c04e49f2?context=explore)镜像。

您可以通过在Dockerfile中使用这些镜像作为基础层来扩展这些镜像并向其中添加您自己的依赖关系。例如，工作示例应用程序使用Ray 2.9.0和Faker 18.13.0。您可以创建一个Dockerfile，通过添加Faker包来扩展`rayproject/ray:2.9.0`：

```dockerfile
# 文件名：Dockerfile
FROM rayproject/ray:2.9.0

RUN pip install Faker==18.13.0
```

通常，`rayproject/ray`镜像仅包含导入Ray和Ray库所需的依赖关系。您可以从这些仓库中的任何一个扩展镜像来构建您的自定义镜像。

然后，您可以构建此镜像并将其推送到您的Dockerhub账户，以便将来可以拉取：

```console
% docker build . -t your_dockerhub_username/custom_image_name:latest

...

% docker image push your_dockerhub_username/custom_image_name:latest

...
```

确保将`your_dockerhub_username`替换为您的DockerHub用户名，将`custom_image_name`替换为您想要的镜像名称。`latest`是此镜像的版本。如果您在拉取镜像时不指定版本，Docker会自动拉取包的`latest`版本。如果您愿意，也可以用特定版本替换`latest`。

## 将Serve应用程序添加到Docker镜像

在开发过程中，将Serve应用程序打包成zip文件并使用`runtime_envs`将其拉入Ray集群是很有用的。在生产环境中，将Serve应用程序放在Docker镜像中而不是`runtime_env`中更稳定，因为新节点在运行之前不需要动态拉取和安装Serve应用程序代码。

在Dockerfile中使用[WORKDIR](https://docs.docker.com/engine/reference/builder/#workdir)和[COPY](https://docs.docker.com/engine/reference/builder/#copy)命令在镜像中安装示例Serve应用程序代码：

```dockerfile
# 文件名：Dockerfile
FROM rayproject/ray:2.9.0

RUN pip install Faker==18.13.0

# 将容器的工作目录设置为/serve_app
WORKDIR /serve_app

# 将本地`fake.py`文件复制到WORKDIR中
COPY fake.py /serve_app/fake.py
```

KubeRay在`WORKDIR`目录内使用`ray start`命令启动Ray。然后所有Ray Serve actor都能够导入目录中的任何依赖关系。通过将Serve文件`COPY`到`WORKDIR`中，Serve部署可以访问Serve代码，而无需`runtime_env`。

对于您的应用程序，您还可以将Serve应用程序所需的任何其他依赖关系添加到`WORKDIR`目录中。

构建此镜像并将其推送到Dockerhub。使用与之前相同的版本来覆盖存储在该版本的镜像。

## 在KubeRay中使用自定义Docker镜像

通过在RayService配置中添加这些自定义Docker镜像，在KubeRay中运行它们。进行以下更改：

1. 将`rayClusterConfig`中的`rayVersion`设置为自定义Docker镜像中使用的Ray版本。
2. 将`ray-head`容器的`image`设置为Dockerhub上自定义镜像的名称。
3. 将`ray-worker`容器的`image`设置为Dockerhub上自定义镜像的名称。
4. 更新`serveConfigV2`字段以移除容器中的任何`runtime_env`依赖关系。

此镜像的预构建版本可在[shrekrisanyscale/serve-fake-email-example](https://hub.docker.com/r/shrekrisanyscale/serve-fake-email-example)获得。通过运行此RayService配置来尝试：

```{literalinclude} ../doc_code/fake_email_creator.yaml
:start-after: __fake_config_start__
:end-before: __fake_config_end__
:language: yaml
```
