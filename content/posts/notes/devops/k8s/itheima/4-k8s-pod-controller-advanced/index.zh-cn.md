---
title: K8s 学习笔记（四）Pod 控制器详解
date: 2022-08-20T01:00:00+08:00
lastmod: 2022-08-21T016:13:00+08:00
categories:
  - 学习笔记
  - K8s
tags:
  - K8s
  - Kubernetes
  - pod
  - controller
---

## 1. Pod控制器介绍

> Pod控制器是管理pod的中间层，使用Pod控制器之后，只需要告诉Pod控制器，想要多少个什么样的Pod就可以了，它会创建出满足条件的Pod并确保每一个Pod资源处于用户期望的目标状态。如果Pod资源在运行中出现故障，它会基于指定策略重新编排Pod。

在kubernetes中，有很多类型的pod控制器，每种都有自己的适合的场景，常见的有下面这些：

- ReplicationController：比较原始的pod控制器，已经被废弃，由ReplicaSet替代
- ReplicaSet：保证副本数量一直维持在期望值，并支持pod数量扩缩容，镜像版本升级
- **Deployment**：通过控制ReplicaSet来控制Pod，并支持滚动升级、回退版本
- Horizontal Pod Autoscaler：可以根据集群负载自动水平调整Pod的数量，实现削峰填谷
- DaemonSet：在集群中的指定Node上运行且仅运行一个副本，一般用于守护进程类的任务
- Job：它创建出来的pod只要完成任务就立即退出，不需要重启或重建，用于执行一次性任务
- Cronjob：它创建的Pod负责周期性任务控制，不需要持续后台运行
- StatefulSet：管理有状态应用

## 2. ReplicaSet(RS)

ReplicaSet的主要作用是**保证一定数量的pod正常运行**，它会持续监听这些Pod的运行状态，一旦Pod发生故障，就会重启或重建。同时它还支持对pod数量的**扩缩容**和**镜像版本的升降级**。

**ReplicaSet资源清单文件**

```yaml
apiVersion: apps/v1 # 版本号
kind: ReplicaSet # 类型       
metadata: # 元数据
  name: pc-replicaset # rs名称 
  namespace: dev # 所属命名空间 
  labels: #标签
    controller: rs
spec: # 详情描述
  replicas: 3 # 副本数量
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本，就是 pod 的配置
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80
```

**扩缩容**

```bash
# 编辑rs的副本数量，修改spec:replicas: 6即可
$ kubectl edit rs pc-replicaset -n dev
replicaset.apps/pc-replicaset edited

# 当然也可以直接使用命令实现
# 使用scale命令实现扩缩容， 后面--replicas=n直接指定目标数量即可
$ kubectl scale rs pc-replicaset --replicas=2 -n dev
replicaset.apps/pc-replicaset scaled
```

**升降级**

```bash
# 编辑rs的容器镜像 - image: nginx:1.17.2
$ kubectl edit rs pc-replicaset -n dev
replicaset.apps/pc-replicaset edited

# 同样的道理，也可以使用命令完成这个工作
# kubectl set image rs rs名称 容器=镜像版本 -n namespace
$ kubectl set image rs pc-replicaset nginx=nginx:1.17.1  -n dev
replicaset.apps/pc-replicaset image updated
```

**删除 ReplicaSet**

```bash
# 使用kubectl delete命令会删除此RS以及它管理的Pod
# 在kubernetes删除RS前，会将RS的replicasclear调整为0，等待所有的Pod被删除后，在执行RS对象的删除
$ kubectl delete rs pc-replicaset -n dev
replicaset.apps "pc-replicaset" deleted

# 如果希望仅仅删除RS对象（保留Pod），可以使用kubectl delete命令时添加--cascade=false选项（不推荐）。
$ kubectl delete rs pc-replicaset -n dev --cascade=false
replicaset.apps "pc-replicaset" deleted

# 也可以使用yaml直接删除(推荐)
$ kubectl delete -f pc-replicaset.yaml
replicaset.apps "pc-replicaset" deleted
```

## 3. Deployment(Deploy)

为了更好的解决服务编排的问题，kubernetes在V1.2版本开始，引入了Deployment控制器。值得一提的是，这种控制器并不直接管理pod，而是通过管理ReplicaSet来简介管理Pod，即：Deployment管理ReplicaSet，ReplicaSet管理Pod。所以Deployment比ReplicaSet功能更加强大。

![Deployment](images/image-20200612005524778.png)

Deployment主要功能有下面几个：

- 支持ReplicaSet的所有功能
- 支持发布的停止、继续
- 支持滚动升级和回滚版本

> 版本回滚通过保留 rs 来实现，升级后，原 rs 依旧存在，只是 pod 数量变为 0

**Deployment 资源清单文件**

```yaml
apiVersion: apps/v1 # 版本号
kind: Deployment # 类型
metadata: # 元数据
  name: pc-deployment # rs名称
  namespace: dev # 所属命名空间
  labels: #标签
    controller: deploy
spec: # 详情描述
  replicas: 3 # 副本数量
  revisionHistoryLimit: 3 # 保留历史版本，默认 10，通过保留 rs 来实现
  paused: false # 暂停部署，默认是false
  progressDeadlineSeconds: 600 # 部署超时时间（s），默认是600
  strategy: # 策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxSurge: 30% # 最大额外可以存在的副本数，可以为百分比，也可以为整数
      maxUnavailable: 30% # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels: # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - { key: app, operator: In, values: [ nginx-pod ] }
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
        - name: nginx
          image: nginx:1.17.1
          ports:
            - containerPort: 80
```

**扩缩容**

同 ReplicaSet 类似

**镜像更新**

eployment支持两种更新策略:`重建更新`和`滚动更新`,可以通过`strategy`指定策略类型,支持两个属性:

```
strategy：指定新的Pod替换旧的Pod的策略， 支持两个属性：
  type：指定策略类型，支持以下两种策略
    (Recreate：在创建出新的Pod之前会先杀掉所有已存在的Pod
    RollingUpdate：滚动更新，就是杀死一部分，就启动一部分，在更新过程中，存在两个版本Pod)
  rollingUpdate：当type为RollingUpdate时生效，用于为RollingUpdate设置参数，支持两个属性：
    maxUnavailable：用来指定在升级过程中不可用Pod的最大数量，默认为25%。
    maxSurge： 用来指定在升级过程中可以超过期望的Pod的最大数量，默认为25%。
```

**版本回退**

deployment支持版本升级过程中的暂停、继续功能以及版本回退等诸多功能。

`kubectl rollout`：版本升级相关功能，支持下面的选项：

- status       显示当前升级状态
- history     显示升级历史记录
- pause       暂停版本升级过程
- resume    继续已经暂停的版本升级过程
- restart      重启版本升级过程
- undo        回滚到上一级版本（可以使用--to-revision回滚到指定版本）

**金丝雀发布**

比如有一批新的Pod资源创建完成后立即暂停更新过程，此时，仅存在一部分新版本的应用，主体部分还是旧的版本。然后，再筛选一小部分的用户请求路由到新版本的Pod应用，继续观察能否稳定地按期望的方式运行。确定没问题之后再继续完成余下的Pod资源滚动更新，否则立即回滚更新操作。这就是所谓的**金丝雀发布**。

```bash
# 更新deployment的版本，并配置暂停deployment
$  kubectl set image deploy pc-deployment nginx=nginx:1.17.4 -n dev && kubectl rollout pause deployment pc-deployment  -n dev
deployment.apps/pc-deployment image updated
deployment.apps/pc-deployment paused

# 确保更新的pod没问题了，继续更新
$ kubectl rollout resume deploy pc-deployment -n dev
deployment.apps/pc-deployment resumed
```
## 4. Horizontal Pod Autoscalar (HPA) 

- HPA可以获取每个Pod利用率，然后和HPA中定义的指标进行对比，同时计算出需要伸缩的具体值，最后实现Pod的数量的调整。
- 实现原理：HPA与之前的Deployment一样，也属于一种Kubernetes资源对象，它通过追踪分析RC控制的所有目标Pod的负载变化情况，来确定是否需要针对性地调整目标Pod的副本数。

![Horizontal Pod Autoscalar](images/image-20200608155858271.png)


**HPA 资源清单文件**

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: pc-hpa
  namespace: dev
spec:
  minReplicas: 1  #最小pod数量
  maxReplicas: 10 #最大pod数量
  targetCPUUtilizationPercentage: 3 # CPU使用率指标，百分比
  scaleTargetRef:   # 指定要控制的 deployment 信息
    apiVersion: apps/v1
    kind: Deployment
    name: nginx  
```

**流程**

1. 创建 deploy
2. 创建 HPA，绑定到 deploy
3. 负载上升 pod 水平扩展
4. 负载下架 pod 等待一段时间后再慢慢收缩

## 5. DaemonSet (DS)

DaemonSet类型的控制器可以保证在集群中的每一台（或指定）节点上都运行一个副本。一般适用于日志收集、节点监控等场景。

如果一个Pod提供的功能是节点级别的（每个节点都需要且只需要一个），那么这类Pod就适合使用DaemonSet类型的控制器创建。

![DaemonSet](images/image-20200612010223537.png)

DaemonSet控制器的特点：

- 每当向集群中添加一个节点时，指定的 Pod 副本也将添加到该节点上
- 当节点从集群中移除时，Pod 也就被垃圾回收了

**DS 资源清单文件**

```yaml
apiVersion: apps/v1 # 版本号
kind: DaemonSet # 类型
metadata: # 元数据
  name:  pc-daemonset # rs名称
  namespace: dev # 所属命名空间
  labels: #标签
    controller: daemonset
spec: # 详情描述
  revisionHistoryLimit: 3 # 保留历史版本
  updateStrategy: # 更新策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxUnavailable: 1 # 最大不可用状态的 Pod 的最大值，可以为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [nginx-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
        - name: nginx
          image: nginx:1.17.1
          ports:
            - containerPort: 80
```

## 6. Job

Job，主要用于负责**批量处理**(一次要处理指定数量任务)短暂的**一次性**(每个任务仅运行一次就结束)任务。Job特点如下：

- 当Job创建的pod执行成功结束时，Job将记录成功结束的pod数量
- 当成功结束的pod达到指定的数量时，Job将完成执行

**Job 资源清单文件**

```yaml
apiVersion: batch/v1 # 版本号
kind: Job # 类型       
metadata: # 元数据
  name: pc-job # rs名称 
  namespace: dev # 所属命名空间 
  labels: #标签
    controller: job
spec: # 详情描述
  completions: 1 # 指定job需要成功运行Pods的次数。默认值: 1
  parallelism: 1 # 指定job在任一时刻应该并发运行Pods的数量。默认值: 1
  activeDeadlineSeconds: 30 # 指定job可运行的时间期限，超过时间还未结束，系统将会尝试进行终止。
  backoffLimit: 6 # 指定job失败后进行重试的次数。默认是6
  manualSelector: true # 是否可以使用selector选择器选择pod，默认是false
  selector: # 选择器，通过它指定该控制器管理哪些pod
    matchLabels:      # Labels匹配规则
      app: counter-pod
    matchExpressions: # Expressions匹配规则
      - {key: app, operator: In, values: [counter-pod]}
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本
    metadata:
      labels:
        app: counter-pod
    spec:
      restartPolicy: Never # 重启策略只能设置为Never或者OnFailure
      containers:
        - name: counter
          image: busybox:1.30
          command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 2;done"]
```

## 7. CronJob (CJ)

CronJob控制器以Job控制器资源为其管控对象，并借助它管理pod资源对象

![CronJob](images/image-20200618213149531.png)

**CJ 资源清单文件**

```yaml
apiVersion: batch/v1 # 版本号
kind: CronJob # 类型
metadata: # 元数据
  name: pc-cronjob # rs名称
  namespace: dev # 所属命名空间
  labels: #标签
    controller: cronjob
spec: # 详情描述
  schedule: "*/1 * * * *" # cron格式的作业调度运行时间点,用于控制任务在什么时间执行
  concurrencyPolicy: Allow # 并发执行策略，用于定义前一次作业运行尚未完成时是否以及如何运行后一次的作业，Allow, Forbid, Replace
  failedJobsHistoryLimit: 1 # 为失败的任务执行保留的历史记录数，默认为1
  successfulJobsHistoryLimit: 3 # 为成功的任务执行保留的历史记录数，默认为3
  startingDeadlineSeconds: 600 # 启动作业错误的超时时长
  jobTemplate: # job控制器模板，用于为cronjob控制器生成job对象;下面其实就是job的定义
    metadata: {}
    spec:
      completions: 1
      parallelism: 1
      activeDeadlineSeconds: 30
      backoffLimit: 6
      manualSelector: true
      selector:
        matchLabels:
          app: counter-pod
        matchExpressions: # 规则
          - {key: app, operator: In, values: [counter-pod]}
      template:
        metadata:
          labels:
            app: counter-pod
        spec:
          restartPolicy: Never
          containers:
            - name: counter
              image: busybox:1.30
              command: ["bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1; do echo $i;sleep 20;done"]
```