# Ray Serve 生产指南中文文档

本目录包含Ray Serve生产指南的中文翻译版本，原始文档位于 `ray/doc/source/serve/production-guide/`。

## 文档列表

### 1. [index.md](index.md) - 生产指南索引
- 介绍Ray Serve在生产环境中的推荐部署方式
- 提供快速入门示例：文本摘要和翻译应用程序
- 指导如何使用Serve CLI部署应用程序

### 2. [config.md](config.md) - Serve配置文件
- 详细介绍Serve配置文件的格式和结构
- 解释各个配置字段的含义和用法
- 说明如何使用`serve build`自动生成配置文件
- 介绍`user_config`的动态参数更新功能

### 3. [handling-dependencies.md](handling-dependencies.md) - 处理依赖关系
- 介绍如何为Serve应用程序添加运行时环境
- 说明如何在每个部署中指定不同的Python依赖
- 提供延迟导入的最佳实践

### 4. [docker.md](docker.md) - 自定义Docker镜像
- 指导如何扩展官方Ray Docker镜像
- 说明如何在自定义Docker镜像中打包Serve应用程序
- 介绍在KubeRay中使用自定义Docker镜像的方法

### 5. [kubernetes.md](kubernetes.md) - 在Kubernetes上部署
- 详细介绍如何使用KubeRay在Kubernetes上部署Serve应用程序
- 说明RayService自定义资源的配置
- 提供应用程序部署、查询、状态监控和更新的完整流程
- 介绍自动扩展、负载均衡器和监控配置

### 6. [fault-tolerance.md](fault-tolerance.md) - 添加端到端容错
- 介绍Serve的容错功能和恢复程序
- 详细说明副本健康检查、工作者节点恢复、头节点恢复等机制
- 提供GCS容错的配置方法
- 包含各种故障场景的模拟和测试方法

### 7. [best-practices.md](best-practices.md) - 生产最佳实践
- 总结CLI使用的最佳实践
- 介绍生产环境中HTTP请求的配置方法
- 说明负载卸载的概念和配置
- 提供重试、指数退避和超时的最佳实践

## 使用说明

这些中文文档保持了原始文档的结构和链接，但所有内容都已翻译成中文。文档中的代码示例、配置文件和命令行示例都保持原样，确保可以直接使用。

## 相关链接

- 原始英文文档：`ray/doc/source/serve/production-guide/`
- Ray官方文档：https://docs.ray.io/
- KubeRay文档：https://docs.ray.io/en/latest/kuberay/

## 注意事项

1. 这些翻译文档基于Ray Serve的生产指南，适用于生产环境部署
2. 建议在使用前先阅读原始英文文档以确保理解最新更新
3. 代码示例和配置文件可以直接使用，无需修改
4. 如有疑问，请参考原始英文文档或Ray官方社区
