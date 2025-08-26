# 处理依赖关系

## 添加运行时环境

导入路径（例如，`text_ml:app`）必须在运行时可由Serve导入。
在本地运行时，此路径可能在您当前的工作目录中。
但是，在集群上运行时，您还需要确保路径是可导入的。
将代码构建到集群的容器镜像中（有关更多详细信息，请参阅[集群配置](kuberay-config)），或使用带有[远程URI](remote-uris)的`runtime_env`，该URI在远程存储中托管代码。

有关示例，请参阅GitHub上的[Text ML Models应用程序](https://github.com/ray-project/serve_config_examples/blob/master/text_ml.py)。您可以使用此配置文件将文本摘要和翻译应用程序部署到您自己的Ray集群，即使您在本地没有代码：

```yaml
import_path: text_ml:app

runtime_env:
    working_dir: "https://github.com/ray-project/serve_config_examples/archive/HEAD.zip"
    pip:
      - torch
      - transformers
```

:::{note}
您还可以将部署图打包成一个独立的Python包，您可以使用[PYTHONPATH](https://docs.python.org/3.10/using/cmdline.html#envvar-PYTHONPATH)导入该包，以在本地机器上提供位置独立性。但是，最佳实践是使用`runtime_env`，以确保集群中所有机器的一致性。
:::

## 每个部署的依赖关系

Ray Serve还支持为具有不同（可能冲突的）Python依赖关系的部署提供服务。
例如，您可以同时服务一个使用旧版Tensorflow 1的部署和另一个使用Tensorflow 2的部署。

这在Mac OS和Linux上使用Ray的{runtime-environments}`runtime-environments`功能得到支持。
与所有其他Ray actor选项一样，通过部署中的`ray_actor_options`传递运行时环境。
确保首先运行`pip install "ray[default]"`以确保安装了Runtime Environments功能。

示例：

```{literalinclude} ../doc_code/varying_deps.py
:language: python
```

:::{tip}
避免动态安装从源代码安装的包：这些包可能很慢，并在安装时耗尽所有资源，导致Ray集群出现问题。
考虑在私有仓库或Docker镜像中预编译此类包。
:::

部署中所需的依赖关系可能与驱动程序（运行Serve API调用的程序）中安装的依赖关系不同。
在这种情况下，您应该在类内使用延迟导入，以避免在驱动程序中导入不可用的包。
即使在不使用运行时环境时，这也适用。

示例：

```{literalinclude} ../doc_code/delayed_import.py
:language: python
```
