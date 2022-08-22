---
title: K8s 学习笔记（七）安全认证和 DashBoard
date: 2022-08-22T16:00:00+08:00
lastmod: 2022-08-22T017:38:00+08:00
categories:
  - 学习笔记
  - K8s
tags:
  - K8s
  - Kubernetes
  - auth
  - dashboard
---

## 1. 安全认证

### 1.1 访问控制

所谓的安全性其实就是保证对Kubernetes的各种**客户端**进行**认证和鉴权**操作。

**客户端**

在Kubernetes集群中，客户端通常有两类：

- **User Account**：一般是独立于kubernetes之外的其他服务管理的用户账号。
- **Service Account**：kubernetes管理的账号，用于为Pod中的服务进程在访问Kubernetes时提供身份标识。

**认证、授权与准入控制**

ApiServer是访问及管理资源对象的唯一入口。任何一个请求访问ApiServer，都要经过下面三个流程：

- Authentication（认证）：身份鉴别，只有正确的账号才能够通过认证
- Authorization（授权）：  判断用户是否有权限对访问的资源执行特定的动作
- Admission Control（准入控制）：用于补充授权机制以实现更加精细的访问控制功能。

### 1.2 认证管理

**K8s 的 3 种客户端身份认证方式**：

- HTTP Base认证：通过用户名+密码的方式认证
- HTTP Token认证：通过一个Token来识别合法用户
- HTTPS证书认证：基于CA根证书签名的双向数字证书认证方式

### 1.3 授权管理

每个发送到ApiServer的请求都带上了用户和资源的信息：比如发送请求的用户、请求的路径、请求的动作等，授权就是根据这些信息和授权策略进行比较，如果符合策略，则认为授权通过，否则会返回错误。

API Server目前支持以下几种授权策略：

- AlwaysDeny：表示拒绝所有请求，一般用于测试
- AlwaysAllow：允许接收所有请求，相当于集群不需要授权流程（Kubernetes默认的策略）
- ABAC：基于属性的访问控制，表示使用用户配置的授权规则对用户请求进行匹配和控制
- Webhook：通过调用外部REST服务对用户进行授权
- Node：是一种专用模式，用于对kubelet发出的请求进行访问控制
- RBAC：基于角色的访问控制（kubeadm安装方式下的默认选项）

RBAC引入了4个顶级资源对象：

- Role、ClusterRole：角色，用于指定一组权限
- RoleBinding、ClusterRoleBinding：角色绑定，用于将角色（权限）赋予给对象

**Role、ClusterRole**

一个角色就是一组权限的集合，这里的权限都是许可形式的（白名单）。

**Role 和 ClusterRole 资源清单**

```yaml
# Role只能对命名空间内的资源进行授权，需要指定nameapce
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: dev
  name: authorization-role
rules:
  - apiGroups: [ "" ] # 支持的API组列表,"" 空字符串，表示核心API群
    resources: [ "pods" ] # 支持的资源对象列表
    verbs: [ "get", "watch", "list" ] # 允许的对资源对象的操作方法列表
```

```yaml
# ClusterRole可以对集群范围内资源、跨namespaces的范围资源、非资源类型进行授权
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: authorization-clusterrole
rules:
  - apiGroups: [ "" ]
    resources: [ "pods" ]
    verbs: [ "get", "watch", "list" ]
```
**RoleBinding、ClusterRoleBinding**

角色绑定用来把一个角色绑定到一个目标对象上，绑定目标可以是User、Group或者ServiceAccount。

**RoleBinding 和 ClusterRoleBinding 资源清单**

```yaml
# RoleBinding可以将同一namespace中的subject绑定到某个Role下，则此subject即具有该Role定义的权限
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: authorization-role-binding
  namespace: dev
subjects:
  - kind: User
    name: heima
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: authorization-role
  apiGroup: rbac.authorization.k8s.io
```

**RoleBinding引用ClusterRole进行授权**

RoleBinding可以引用ClusterRole，对属于同一命名空间内ClusterRole定义的资源主体进行授权。

> 一种很常用的做法就是，集群管理员为集群范围预定义好一组角色（ClusterRole），然后在多个命名空间中重复使用这些ClusterRole。这样可以大幅提高授权管理工作效率，也使得各个命名空间下的基础性授权规则与使用体验保持一致。

**资源清单**

```yaml
# 虽然authorization-clusterrole是一个集群角色，但是因为使用了RoleBinding
# 所以heima只能读取dev命名空间中的资源
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: authorization-role-binding-ns
  namespace: dev
subjects:
  - kind: User
    name: heima
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: authorization-clusterrole
  apiGroup: rbac.authorization.k8s.io
```

### 1.4 准入控制

通过了前面的认证和授权之后，还需要经过准入控制处理通过之后，apiserver才会处理这个请求。

## 2. DashBoard

[kubernetes/dashboard](https://github.com/kubernetes/dashboard)
