# K8s 学习笔记（三）Pod 详解


## 1. Pod 介绍

### 1.1 Pod 结构

![Pod 结构](images/image-20200407121501907.png)

每个Pod中都可以包含一个或者多个容器，这些容器可以分为两类：

- 用户程序所在的容器，数量可多可少

- Pause容器，这是每个Pod都会有的一个**根容器**，它的作用有两个：
    - 可以以它为依据，评估整个Pod的健康状态
    - 可以在根容器上设置Ip地址，其它容器都此Ip（Pod IP），以实现Pod**内部**的网路通信。*这里是Pod内部的通讯，Pod的之间的通讯采用虚拟二层网络技术来实现，我们当前环境用的是Flannel*

### 1.2 Pod 定义

**可通过`kubectl explain`命令来查看每种资源的可配置项**

- `kubectl explain 资源类型`：查看某种资源可以配置的一级属性
- `kubectl explain 资源类型.属性[.属性...]`：查看属性的子属性

Pod 资源清单(YAML 配置):

```yaml
apiVersion: v1     #必选，版本号，例如v1
kind: Pod       　 #必选，资源类型，例如 Pod
metadata: #必选，元数据
  name: string     #必选，Pod名称
  namespace: string  #Pod所属的命名空间,默认为"default"
  labels: 　　  #自定义标签列表
    - name: string      　
spec: #必选，Pod中容器的详细定义
  containers: #必选，Pod中容器列表
    - name: string   #必选，容器名称
      image: string  #必选，容器的镜像名称
      imagePullPolicy: [ Always|Never|IfNotPresent ]  #获取镜像的策略 
      command: [ string ]   #容器的启动命令列表，如不指定，使用打包时使用的启动命令
      args: [ string ]      #容器的启动命令参数列表
      workingDir: string  #容器的工作目录
      volumeMounts: #挂载到容器内部的存储卷配置
        - name: string      #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
          mountPath: string #存储卷在容器内mount的绝对路径，应少于512字符
          readOnly: boolean #是否为只读模式
      ports: #需要暴露的端口库号列表
        - name: string        #端口的名称
          containerPort: int  #容器需要监听的端口号
          hostPort: int       #容器所在主机需要监听的端口号，默认与Container相同
          protocol: string    #端口协议，支持TCP和UDP，默认TCP
      env: #容器运行前需设置的环境变量列表
        - name: string  #环境变量名称
          value: string #环境变量的值
      resources: #资源限制和请求的设置
        limits: #资源限制的设置
          cpu: string     #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
          memory: string  #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
        requests: #资源请求的设置
          cpu: string    #Cpu请求，容器启动的初始可用数量
          memory: string #内存请求,容器启动的初始可用数量
      lifecycle: #生命周期钩子
  postStart: #容器启动后立即执行此钩子,如果执行失败,会根据重启策略进行重启
  preStop: #容器终止前执行此钩子,无论结果如何,容器都会终止
    livenessProbe: #对Pod内各容器健康检查的设置，当探测无响应几次后将自动重启该容器
      exec: #对Pod容器内检查方式设置为exec方式
        command: [ string ]  #exec方式需要制定的命令或脚本
      httpGet: #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
          - name: string
            value: string
      tcpSocket: #对Pod内个容器健康检查方式设置为tcpSocket方式
        port: number
      initialDelaySeconds: 0       #容器启动完成后首次探测的时间，单位为秒
      timeoutSeconds: 0    　　    #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
      periodSeconds: 0     　　    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
      successThreshold: 0
      failureThreshold: 0
      securityContext:
        privileged: false
  restartPolicy: [ Always | Never | OnFailure ]  #Pod的重启策略
  nodeName: <string> #设置NodeName表示将该Pod调度到指定到名称的node节点上
  nodeSelector: obeject #设置NodeSelector表示将该Pod调度到包含这个label的node上
  imagePullSecrets: #Pull镜像时使用的secret名称，以key：secretkey格式指定
    - name: string
  hostNetwork: false   #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
  volumes: #在该pod上定义共享存储卷列表
    - name: string    #共享存储卷名称 （volumes类型有很多种）
      emptyDir: { }       #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string      　　        #Pod所在宿主机的目录，将被用于同期中mount的目录
      secret: #类型为secret的存储卷，挂载集群与定义的secret对象到容器内部
        secretName: string
        items:
          - key: string
            path: string
      configMap: #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
          - key: string
            path: string
```

在kubernetes中基本所有资源的一级属性都是一样的，主要包含5部分：

- `apiVersion <string>`：版本，由kubernetes内部定义，版本号必须可以用 `kubectl api-versions` 查询到
- `kind <string>`：类型，由kubernetes内部定义，版本号必须可以用 `kubectl api-resources` 查询到
- `metadata <Object>`：元数据，主要是资源标识和说明，常用的有name、namespace、labels等
- `spec <Object>`：描述，这是配置中**最重要**的一部分，里面是对各种资源配置的详细描述
- `status <Object>`：状态信息，里面的内容不需要定义，由kubernetes自动生成

在上面的属性中，spec是接下来研究的重点，继续看下它的常见子属性:

- `containers <[]Object>`：容器列表，用于定义容器的详细信息
- `nodeName <String>`：根据nodeName的值将pod调度到指定的Node节点上
- `nodeSelector  <map[`]>：根据NodeSelector中定义的信息选择将该Pod调度到包含这些label的Node 上
- `hostNetwork <boolean>`：是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
- `volumes <[]Object>`：存储卷，用于定义Pod上面挂在的存储信息
- `restartPolicy <string>`：重启策略，表示Pod在遇到故障的时候的处理策略

## 2. Pod 配置

```bash
[root@master ~]# kubectl explain pod.spec.containers
KIND:     Pod
VERSION:  v1
RESOURCE: containers <[]Object>   # 数组，代表可以有多个容器
FIELDS:
   name  <string>     # 容器名称
   image <string>     # 容器需要的镜像地址
   imagePullPolicy  <string> # 镜像拉取策略 
   command  <[]string> # 容器的启动命令列表，如不指定，使用打包时使用的启动命令
   args     <[]string> # 容器的启动命令需要的参数列表
   env      <[]Object> # 容器环境变量的配置
   ports    <[]Object>     # 容器需要暴露的端口号列表
   resources <Object>      # 资源限制和资源请求的设置
```

### 2.1 基本配置

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-base
  namespace: dev
  labels:
    user: heima
spec:
  containers:
    - name: nginx
      image: nginx
    - name: busybox
      image: busybox
```

### 2.2 镜像拉取(imagePullPolicy)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-base
  namespace: dev
  labels:
    user: heima
spec:
  containers:
    - name: nginx
      image: nginx
      imagePullPolicy: Always  # 镜像拉取策略
    - name: busybox
      image: busybox
```

**imagePullPolicy 三种拉取策略**：

- Always：总是从远程仓库拉取镜像（一直远程下载），tag 为 `latest` 时**默认**
- IfNotPresent：本地有则使用本地镜像，本地没有则从远程仓库拉取镜像（本地有就本地  本地没远程下载），tag 为具体版本时**默认**。
- Never：只使用本地镜像，从不去远程仓库拉取，本地没有就报错 （一直使用本地）

### 2.3 启动命令(command 和 args)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-command
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx
    - name: busybox
      image: busybox
      # command 指定容器运行后执行的命令
      command: ["/bin/sh", "-c", "touch /tmp/hello.txt;while true;do /bin/echo $(date +%T) >> /tmp/hello.txt; sleep 3; done;"]
      # args 指定参数
      args: []
```

- command 和 args 对应 Dockerfile 中的 ENTRYPOINT 
- 指定 command 会覆盖 ENTRYPOINT；指定 args 会执行 ENTRYPOINT 并带上 args 参数


**进入容器**
```bash
cubectl exec <pod-name> [-n <namespace>] -it -c <contaienr name> <command>
```

### 2.4 环境变量(env)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-env
  namespace: dev
spec:
  containers:
    - name: busybox
      image: busybox
      command: ["/bin/sh", "-c", "while true;do /bin/echo $(date +%T); sleep 60; done;"]
      env:
        - name: "username"
          value: "admin"
        - name: "password"
          value: "123456"
```

这种方式不是很推荐，推荐将这些配置单独存储在**配置文件**中。

### 2.5 端口设置

支持的选项：
```bash
[root@master ~]# kubectl explain pod.spec.containers.ports
KIND:     Pod
VERSION:  v1
RESOURCE: ports <[]Object>
FIELDS:
   name         <string>  # 端口名称，如果指定，必须保证name在pod中是唯一的		
   containerPort<integer> # 容器要监听的端口(0<x<65536)
   hostPort     <integer> # 容器要在主机上公开的端口，如果设置，主机上只能运行容器的一个副本(一般省略) 
   hostIP       <string>  # 要将外部端口绑定到的主机IP(一般省略)
   protocol     <string>  # 端口协议。必须是UDP、TCP或SCTP。默认为“TCP”。
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-ports
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx
      ports:  # 暴露的端口列表
        - name: nginx-port
          containerPort: 80  # 端口号
```

访问容器中的程序使用`podIp:containerPort`

### 2.6 资源配额

如果不对某个容器的资源做限制，那么它就可能吃掉大量资源，导致其它容器无法运行。

通过resources选项实现，他有两个子选项：

- limits：**上限**，用于限制运行时容器的最大占用资源，当容器占用资源超过limits时会被终止，并进行重启
- requests ：**下限**，用于设置容器需要的最小资源，如果环境资源不够，容器将无法启动

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-resources
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx
      resources:
        limits:
          cpu: "2"
          memory: "10Gi"
        requests:
          cpu: "1"
          memory: "10Mi"
```

> 1Gi = 1.024 * 1G

## 3. Pod 生命周期

**Pod 生命周期**：

![Pod 生命周期](images/image-20200412111402706.png)

- pod创建过程
- 运行初始化容器（init container）过程
- 运行主容器（main container）
  - 容器启动后钩子（post start）、容器终止前钩子（pre stop）
  - 容器的存活性探测（liveness probe）、就绪性探测（readiness probe）
- pod终止过程

在整个生命周期中，Pod会出现5种**状态**（**相位**），分别如下：

- 挂起（Pending）：apiserver已经创建了pod资源对象，但它尚未被调度完成或者仍处于下载镜像的过程中
- 运行中（Running）：pod已经被调度至某节点，并且所有容器都已经被kubelet创建完成
- 成功（Succeeded）：pod中的所有容器都已经成功终止并且不会被重启
- 失败（Failed）：所有容器都已经终止，但至少有一个容器终止失败，即容器返回了非0值的退出状态
- 未知（Unknown）：apiserver无法正常获取到pod对象的状态信息，通常由网络通信失败所导致

### 3.1 创建和终止

**pod的创建过程**

1. 用户通过kubectl或其他api客户端提交需要创建的pod信息给apiServer
2. apiServer开始生成pod对象的信息，并将信息存入etcd，然后返回确认信息至客户端
3. apiServer开始反映etcd中的pod对象的变化，其它组件使用**watch机制**来跟踪检查apiServer上的变动
4. scheduler发现有新的pod对象要创建，开始为Pod分配主机并将结果信息更新至apiServer
5. node节点上的kubelet发现有pod调度过来，尝试调用docker启动容器，并将结果回送至apiServer
6. apiServer将接收到的pod状态信息存入etcd中

**pod的终止过程**

1. 用户向apiServer发送删除pod对象的命令
2. apiServcer中的pod对象信息会随着时间的推移而更新，在宽限期内（默认30s），pod被视为dead
3. 将pod标记为terminating状态
4. kubelet在监控到pod对象转为terminating状态的同时启动pod关闭过程
5. 端点控制器监控到pod对象的关闭行为时将其从所有匹配到此端点的service资源的端点列表中移除
6. 如果当前pod对象定义了preStop钩子处理器，则在其标记为terminating后即会以同步的方式启动执行
7. pod对象中的容器进程收到停止信号
8. 宽限期结束后，若pod中还存在仍在运行的进程，那么pod对象会收到立即终止的信号
9. kubelet请求apiServer将此pod资源的宽限期设置为0从而完成删除操作，此时pod对于用户已不可见

### 3.2 运行初始化容器

初始化容器是在pod的主容器启动之前要运行的容器，主要是做一些主容器的前置工作，它具有两大特征：

1. 初始化容器必须运行完成直至结束，若某初始化容器运行失败，那么kubernetes需要重启它直到成功完成
2. 初始化容器必须按照定义的顺序执行，当且仅当前一个成功之后，后面的一个才能运行

初始化容器的应用场景：

- 提供主容器镜像中不具备的工具程序或自定义代码
- 初始化容器要先于应用容器串行启动并运行完成，因此可用于延后应用容器的启动直至其**依赖**的条件得到满足

**Pod 配置文件案例**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-initcontainer
  namespace: dev
spec:
  containers:
    - name: main-container
      image: nginx
      ports:
        - containerPort: 80
  initContainers:
    - name: test-mysql
      image: busybox
      command: ["sh", "-c", "until ping 192.168.31.201 -c 1; do echo waiting for mysql...; sleep 2; done;"]
    - name: test-redis
      image: busybox
      command: ["sh", "-c", "until ping 192.168.31.202 -c 1; do echo waiting for mysql...; sleep 2; done;"]
```

### 3.3 钩子函数

kubernetes在主容器的启动之后和停止之前提供了两个钩子函数：

- post start：容器创建之后执行，如果失败了会重启容器
- pre stop  ：容器终止之前执行，执行完成之后容器将成功终止，在其完成之前会阻塞删除容器的操作

钩子处理器支持使用下面三种方式定义动作：

- Exec命令：在容器内执行一次命令

```yaml
lifecycle:
  postStart: 
    exec:
      command:
        - cat
        - /tmp/healthy
```

- TCPSocket：在当前容器尝试访问指定的socket

```yaml
lifecycle:
  postStart:
    tcpSocket:
      port: 8080
```

- HTTPGet：在当前容器中向某url发起http请求

```yaml
lifecycle:
  postStart:
    httpGet:
      path: / #URI地址
      port: 80 #端口号
      host: 192.168.109.100 #主机地址
      scheme: HTTP #支持的协议，http或者https
```

**Pod配置文件举例**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-hook-exec
  namespace: dev
spec:
  containers:
    - name: main-container
      image: nginx
      ports:
        - containerPort: 80
      lifecycle:
        postStart:
          exec:
            command: ["/bin/sh", "-c", "echo postStart... > /usr/share/nginx/html/index.html"]
        preStop:
          exec:
            command: ["/usr/sbin/nginx", "-s", "quit"]
```

### 3.4 容器探测

容器探测用于检测容器中的应用实例是否正常工作，是保障业务可用性的一种传统机制。如果经过探测，实例的状态不符合预期，那么kubernetes就会把该问题实例" 摘除 "，不承担业务流量。kubernetes提供了两种探针来实现容器探测，分别是：

- liveness probes：存活性探针，用于检测应用实例当前是否处于正常运行状态，如果不是，k8s会重启容器
- readiness probes：就绪性探针，用于检测应用实例当前是否可以接收请求，如果不能，k8s不会转发流量

> livenessProbe 决定是否重启容器，readinessProbe 决定是否将请求转发给容器。

上面两种探针同样支持[三种探测方式](#33-钩子函数)：

- Exec 命令
- TCPSocket
- HTTPGet

**Pod配置文件举例**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-liveness-exec
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
      livenessProbe:
        exec:
          command:
            - /bin/cat
            - /tmp/hello.txt
```

livenessProbe 还有其他配置：

```bash
[root@master ~]# kubectl explain pod.spec.containers.livenessProbe
  FIELDS:
    exec <Object>
    tcpSocket    <Object>
    httpGet      <Object>
    initialDelaySeconds  <integer>  # 容器启动后等待多少秒执行第一次探测
    timeoutSeconds       <integer>  # 探测超时时间。默认1秒，最小1秒
    periodSeconds        <integer>  # 执行探测的频率。默认是10秒，最小1秒
    failureThreshold     <integer>  # 连续探测失败多少次才被认定为失败。默认是3。最小值是1
    successThreshold     <integer>  # 连续探测成功多少次才被认定为成功。默认是1
```

### 3.5 重启策略

pod的重启策略有 3 种，分别如下：

- Always ：容器失效时，自动重启该容器，这也是默认值。
- OnFailure ： 容器终止运行且退出码不为0时重启
- Never ： 不论状态为何，都不重启该容器

重启延迟：

- 首次立即重启
- 接下来重启延迟为 10s、20s、40s、80s、160s和300s
- 最大延迟为 300s

**Pod配置文件举例**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-restartpolicy
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 80
      livenessProbe:
        httpGet:
          port: 80
          path: /hello
          scheme: HTTP
  restartPolicy: Never  # 注意！是 pod 的属性，不是 containers 的
```

## 4. Pod 调度

在默认情况下，一个Pod在哪个Node节点上运行，是由Scheduler组件采用相应的算法计算出来的，这个过程是不受人工控制的。

但是在实际使用中，这并不满足的需求，因为很多情况下，我们想控制某些Pod到达某些节点上，那么应该怎么做呢？这就要求了解kubernetes对Pod的调度规则，kubernetes提供了四大类调度方式：

- 自动调度：运行在哪个节点上完全由Scheduler经过一系列的算法计算得出
- 定向调度：NodeName、NodeSelector
- 亲和性调度：NodeAffinity、PodAffinity、PodAntiAffinity
- 污点（容忍）调度：Taints、Toleration

### 4.1 定向调度

利用在pod上声明nodeName或者nodeSelector，以此将Pod调度到期望的node节点上。

**注意**，这里的调度是**强制的**，这就意味着即使要调度的目标Node不存在，也会向上面进行调度，只不过pod运行**失败**而已。

#### 4.1.1 NodeName

NodeName用于**强制约束**将Pod调度到指定的Name的Node节点上。这种方式，其实是直接跳过Scheduler的调度逻辑，直接将Pod调度到指定名称的节点。

**Pod配置文件举例**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodename
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx
  nodeName: u-2  # 节点名称，通过 kubectl get nodes 查看
```

#### 4.1.2 NodeSelector

NodeSelector用于将pod调度到添加了指定标签的node节点上。也是**强制约束**

**Pod配置文件举例**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeselector
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx
  nodeSelector:
    nodeenv: pro  # 指定调度到具有 nodeenv=pro 标签的节点上
```

### 4.2 亲和性调度

定向调度有一定的**问题**，那就是如果没有满足条件的Node，那么Pod将不会被运行，即使在集群中还有可用Node列表也不行，这就限制了它的使用场景。

亲和性调度（Affinity）：它在NodeSelector的基础之上的进行了扩展，可以通过配置的形式，实现优先选择满足条件的Node进行调度，如果没有，也可以调度到不满足条件的节点上，使调度更加灵活。

Affinity主要分为三类：

- nodeAffinity(node亲和性）: 以node为目标，解决pod可以调度到哪些node的问题
- podAffinity(pod亲和性) :  以pod为目标，解决pod可以和哪些已存在的pod部署在同一个拓扑域中的问题
- podAntiAffinity(pod反亲和性) :  以pod为目标，解决pod不能和哪些已存在pod部署在同一个拓扑域中的问题

> 亲和性(反亲和性)使用场景的说明：
> - **亲和性**：如果两个应用**频繁交互**，那就有必要利用亲和性让两个应用的尽可能的靠近，这样可以减少因网络通信而带来的性能损耗。
> - **反亲和性**：当应用的采用多副本部署时，有必要采用反亲和性让各个应用实例打散分布在各个node上，这样可以提高服务的**高可用性**。

#### 4.2.1 NodeAffinity

**主要字段**

```bash
$ kubectl explain pod.spec.affinity.nodeAffinity
  requiredDuringSchedulingIgnoredDuringExecution  # Node节点必须满足指定的所有规则才可以，相当于硬限制
    nodeSelectorTerms  # 节点选择列表
      matchFields   # 按节点字段列出的节点选择器要求列表
      matchExpressions   #按节点标签列出的节点选择器要求列表(推荐)
        key    # 键
        values # 值
        operator # 关系符 支持Exists, DoesNotExist, In, NotIn, Gt, Lt
  preferredDuringSchedulingIgnoredDuringExecution # 优先调度到满足指定的规则的Node，相当于软限制 (倾向)
    preference   # 一个节点选择器项，与相应的权重相关联
      matchFields   # 按节点字段列出的节点选择器要求列表
      matchExpressions   # 按节点标签列出的节点选择器要求列表(推荐)
        key    # 键
        values # 值
        operator # 关系符 支持In, NotIn, Exists, DoesNotExist, Gt, Lt
	weight # 倾向权重，在范围1-100。
```

**Pod配置文件举例**

硬限制：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeaffinity-required
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx
  affinity:  # 亲和性设置
    nodeAffinity:  # 设置 node 亲和性
      requiredDuringSchedulingIgnoredDuringExecution:  # 硬限制
        nodeSelectorTerms:  # Selector 列表
          - matchExpressions:  # 匹配 nodeenv 值在 ['xxx', 'yyy'] 中的标签
              - key: nodeenv
                operator: In
                values:
                  - xxx
                  - yyy
```

软限制：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-nodeaffinity-preferred
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx
  affinity: # 亲和性设置
    nodeAffinity: # 设置 node 亲和性
      preferredDuringSchedulingIgnoredDuringExecution: # 软限制
        - weight: 1
          preference:
            matchExpressions: # 匹配 nodeenv 值在 ['xxx', 'yyy'] 中的标签
              - key: nodeenv
                operator: In
                values:
                  - xxx
                  - yyy
```

#### 4.2.1 PodAffinity

PodAffinity主要实现以运行的Pod为参照，实现让新创建的Pod跟参照pod在同一个区域的功能。

同样有硬限制和软限制

**配置项**：

```bash
$ kubectl explain pod.spec.affinity.podAffinity
  requiredDuringSchedulingIgnoredDuringExecution  # 硬限制
    namespaces      # 指定参照pod的namespace
    topologyKey     # 指定调度作用域
    labelSelector   # 标签选择器
      matchExpressions # 按节点标签列出的节点选择器要求列表(推荐)
        key   # 键
        values # 值
        operator # 关系符 支持In, NotIn, Exists, DoesNotExist.
      matchLabels   # 指多个matchExpressions映射的内容
  preferredDuringSchedulingIgnoredDuringExecution # 软限制
    podAffinityTerm # 选项
      namespaces      
      topologyKey
      labelSelector
        matchExpressions  
          key   # 键
          values # 值
          operator
        matchLabels 
    weight # 倾向权重，在范围1-100
```

> topologyKey：指定调度作用域
> - kubernetes.io/hostname，那就是以Node节点为区分范围
> - beta.kubernetes.io/os,则以Node节点的操作系统类型来区分

**Pod配置文件举例**

硬限制：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-podaffinity-required
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx
  affinity: # 亲和性设置
    podAffinity: # 设置 pod 亲和性
      requiredDuringSchedulingIgnoredDuringExecution: # 硬限制
        - labelSelector:
            matchExpressions:
              - key: podenv
                operator: In
                values:
                  - xxx
                  - yyy
          topologyKey: kubernetes.io/hostname
```

#### 4.2.1 PodAntiAffinity

PodAntiAffinity主要实现以运行的Pod为参照，让新创建的Pod跟参照pod不在一个区域中的功能。

它的配置方式和选项跟PodAffinity是一样的


### 4.3 污点和容忍

#### 4.3.1 污点

我们也可以站在Node的角度上，通过在Node上添加**污点**属性，来决定是否允许Pod调度过来。

Node被设置上污点之后就和Pod之间存在了一种相斥的关系，进而拒绝Pod调度进来，甚至可以将已经存在的Pod驱逐出去。

污点的格式为：`key=value:effect`, key和value是污点的标签，effect描述污点的作用，支持如下三个选项：

- PreferNoSchedule：kubernetes将**尽量**避免把Pod调度到具有该污点的Node上，除非没有其他节点可调度
- NoSchedule：kubernetes将不会把Pod调度到具有该污点的Node上，但**不会影响**当前Node上**已存在的**Pod
- NoExecute：kubernetes将不会把Pod调度到具有该污点的Node上，同时也会将Node上已存在的Pod**驱离**

> - 使用kubeadm搭建的集群，默认就会给master节点添加一个污点标记(node-role.kubernetes.io/master:NoSchedule),所以pod就不会调度到master节点上.
> - 离线的 node 会添加两个污点： node.kubernetes.io/unreachable:NoExecute、node.kubernetes.io/unreachable:NoSchedule


**命令**：

```bash
# 设置污点
kubectl taint nodes <node-name> <key=value:effect>

# 去除污点
kubectl taint nodes <node-name> <key:effect->

# 去除所有污点
kubectl taint nodes <node-name> <key->
```

#### 4.3.2 容忍

> 污点就是拒绝，容忍就是忽略，Node通过污点拒绝pod调度上去，Pod通过容忍忽略拒绝

**Pod配置举例**

添加容忍：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-toleration
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx
  tolerations:  # 添加容忍
    - key: tag  # 要容忍的污点的 key, 空意味着匹配所有的键
      operator: Equal  # key 和 value 间操作符，支持 Equal 和 Exists（默认）
      value: heima  # 容忍的污点的 value
      effect: NoExecute  # 容忍的规则，必须和标记的污点的规则相同，空意味着匹配所有影响
      # tolerationSeconds   # 容忍时间, 当effect为NoExecute时生效，表示pod在Node上的停留时间
```


---

> 作者: [黄波](https://boh5.com)  
> URL: https://boh5.com/posts/notes/devops/k8s/itheima/3-k8s-pod-advanced/  

