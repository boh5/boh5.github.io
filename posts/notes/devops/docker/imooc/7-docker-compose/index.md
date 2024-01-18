# Docker 学习笔记（七）docker compose


## 1. 简介

- docker-compose： 定义和运行多个 Docker 容器的应用
- compose v2： Docker 官方用 GO 语言 重写 了 Docker Compose，并将其作为了 docker cli 的子命令，称为 Compose V2。`docker-compose` 命令替换为 `docker compose`，即可使用 Compose v2

## 2. compose 文件结构和命令

基本语法结构：

```docker-compose
version: "3.8" # 新版 compose spec 废弃了 version 字段，不用指定 version

services: # 容器
  servicename: # 服务名字，这个名字也是内部 bridge网络可以使用的 DNS name
    image: # 可选，镜像的名字
    build: # 可选，如果同时指定了 image 会使用 image 字段作为镜像名称
    command: # 可选，如果设置，则会覆盖默认镜像里的 CMD命令
    environment: # 可选，相当于 docker run里的 --env
    volumes: # 可选，相当于docker run里的 -v
    networks: # 可选，相当于 docker run里的 --network
    ports: # 可选，相当于 docker run里的 -p
  servicename2:

volumes: # 可选，相当于 docker volume create

networks: # 可选，相当于 docker network create
```

命令：
- 创建并启动：`docker compose [-f composefile.yml] up [-d] [--build]`
- 停止并删除：`docker compose [-f composefile.yml] down`
- 启动、停止：`docker compose [-f composefile.yml] {start|stop}`
- 重启：`docker compose [-f composefile.yml] restart`
- build or rebuild：`docker compose [-f composefile.yml] build`

## 3. compose 网络

- 不指定网络，会创建一个默认网络
- 创建网络不指定 driver 为默认 bridge

## 4. compose 水平扩展

- 配置 compose file

```docker-compose
services:
  frontend:
    image: awesome/webapp
    deploy:
      mode: replicated
      replicas: 6 
```
- 或者 up 时指定 scale，会覆盖 compose file 中的配置：`docker compose up -d --scale frontend=3`
- 还会做简单的负载均衡，使用 service name 访问时，dns 会解析到不同的 replicas 中

## 5. compose 环境变量

- 在 compose 文件中使用 `${VAR_NAME}` 代表从外部读取变量值填入 compose 文件
- 传入变量的方式：创建 `.env` 文件，在文件中赋值会自动传入
- 验证 compose 文件：`docker compose config`
- 指定 env 文件名：`docker compose --env-file <filename> ...`

## 6. 服务依赖和健康检查

- `depends_on` 指定依赖，被依赖的服务启动后，才会启动该服务
- 只依赖启动顺序不靠谱，可能被依赖的服务启动了但是发生错误了，因此需要健康检查
- `Dockerfile` 也可以定义健康检查：`HEALTHCHECK` 指令，根据 `--interval` 间隔不断检查，`docker container ls` 可以在 `STATUS` 字段看到 health 状态(starting、healthy, unhealthy)
- 在 compose 文件中定义 health check：
```docker-compose
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost"]
  interval: 1m30s
  timeout: 10s
  retries: 3
  start_period: 40s 
```
- 只有 depends_on 是 [healthy|started] 才启动：
```docker-compose
depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started 
```


---

> 作者: [黄波](https://boh5.github.io)  
> URL: https://boh5.github.io/posts/notes/devops/docker/imooc/7-docker-compose/  

