# Docker 学习笔记（十）docker 的多架构支持


## 1. 使用 buildx 构建多架构支持的 Docker 镜像

- Docker for Linux 不支持构建 arm 架构镜像，我们可以运行一个新的容器让其支持该特性，Docker 桌面版无需进行此项设置：` docker run --rm --privileged tonistiigi/binfmt:latest --install all`
- 由于 Docker 默认的 builder 实例不支持同时指定多个 --platform，我们必须首先创建一个新的 builder 实例：`docker buildx create --name mybuilder --use`
- 构建并 push 镜像：`docker buildx build --push --platform linux/arm,linux/arm64,linux/amd64 -t myusername/hello .`




---

> 作者: [黄波](https://dilless.github.io)  
> URL: https://dilless.github.io/posts/notes/devops/docker/imooc/10-docker-arch/  

