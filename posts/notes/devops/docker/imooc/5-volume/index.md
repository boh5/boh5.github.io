# Docker 学习笔记（五）存储


Docker主要提供了两种方式做数据的持久化

- Data Volume, 由Docker管理，(/var/lib/docker/volumes/), 持久化数据的最好方式
- Bind Mount，由用户指定存储的数据具体mount在系统什么位置

## 1. Data Volume

- 仅在 Dockerfile 中定义 `VOLUME ['path']` 时，Docker会自动创建一个随机名字的volume，去存储我们在Dockerfile定义的volume
- `docker container run -d -v <volumn_name>:<dir_in_container> my-cron`：启动容器时指定 volume，可以指向容器内任意路径
- Volume 相关命令：`docker volume [ls|inspect|rm|prune...]`

## 2. Bind Mount

- 如果使用 `-v` 或 `-volume` 来绑定挂载 Docker 主机上还不存在的文件或目录，则 -v 将为您创建它。它总是作为目录创建的。
- 如果使用 `--mount` 绑定挂载 Docker 主机上还不存在的文件或目录，Docker 不会自动为您创建它，而是产生一个错误。

`--mount`：由多个键-值对组成，以逗号分隔，每个键-值对由一个 <key>=<value> 元组组成。`--mount` 语法比 `-v` 或 `--volume` 更冗长，但是键的顺序并不重要，标记的值也更容易理解。

- 挂载的类型（`type`），可以是 `bind`、`volume` 或者 `tmpfs`。
- 挂载的源（`source`），对于绑定挂载，这是 Docker 守护进程主机上的文件或目录的路径。可以用 source 或者 `src` 来指定。
- 目标（`destination`），将容器中文件或目录挂载的路径作为其值。可以用 destination、`dst` 或者 target 来指定。
- `readonly` 选项（如果存在），则会将绑定挂载以只读形式挂载到容器中。
- `bind-propagation` 选项（如果存在），则更改绑定传播。 可能的值是 rprivate、 private、 rshared、 shared、 rslave 或 slave 之一。



## 3. 使用 docker 做开发环境

- bind 项目路径到 docker container，然后可以在 container 中做操作
- 设置 pycharm 的 interpreter 为 docker，然后可以运行 celery 等在 linux 中才能跑的库

## 4. 多个机器的容器共享数据

- Docker的volume支持多种driver。默认创建的volume driver都是local
- sshfs的driver，如何让docker使用不在同一台机器上的文件系统做volume
- 安装 plugin: `docker plugin install --grant-all-permissions vieux/sshfs`
- 创建 volume，该 volume 实际上 ssh 远程机器上的文件系统：

```bash
docker volume create --driver vieux/sshfs \
                          -o sshcmd=vagrant@192.168.200.12:/home/vagrant \
                          -o password=vagrant \
                          sshvolume
```


---

> 作者: [黄波](https://dilless.github.io)  
> URL: https://dilless.github.io/posts/notes/devops/docker/imooc/5-volume/  

