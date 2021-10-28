Contents
=================
- [Contents](#contents)
- [在 Kubernetes 上部署 RadonDB ClickHouse](#在-kubernetes-上部署-radondb-clickhouse)
  - [简介](#简介)
  - [部署准备](#部署准备)
  - [部署步骤](#部署步骤)
    - [步骤 1 : 添加仓库](#步骤-1--添加仓库)
    - [步骤 2 : 部署 RadonDB ClickHouse Operator](#步骤-2--部署-radondb-clickhouse-operator)
    - [步骤 3 : 部署 RadonDB ClickHouse 集群](#步骤-3--部署-radondb-clickhouse-集群)
    - [步骤 4 : 部署校验](#步骤-4--部署校验)
      - [查看 Pod 运行状态](#查看-pod-运行状态)
      - [查询 SVC 运行状态](#查询-svc-运行状态)
  - [访问 RadonDB ClickHouse](#访问-radondb-clickhouse)
    - [通过 Pod](#通过-pod)
    - [通过 Service](#通过-service)
  - [持久化](#持久化)
  - [配置](#配置)

# 在 Kubernetes 上部署 RadonDB ClickHouse

## 简介

RadonDB ClickHouse 是基于 [ClickHouse](https://clickhouse.tech/) 的开源、高可用、云原生集群解决方案。具备高可用、PB 级数据存储、实时数据分析、架构稳定和可扩展等性能。

本教程演示如何使用命令行在 Kubernetes 上部署 RadonDB ClickHouse。

## 部署准备

- 已成功部署 Kubernetes 集群。
- 已安装 Helm 包管理工具。

## 部署步骤

### 步骤 1 : 添加仓库

添加并更新 Helm 仓库。

```bash
$ helm repo add <repoName> https://radondb.github.io/radondb-clickhouse-kubernetes/
$ helm repo update
```

**预期效果**

```shell
$ helm repo add ck https://radondb.github.io/radondb-clickhouse-kubernetes/
"ck" has been added to your repositories

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ck" chart repository
Update Complete. ⎈Happy Helming!⎈

```

### 步骤 2 : 部署 RadonDB ClickHouse Operator

```bash
$ helm install --generate-name -n <nameSpace> <repoName>/clickhouse-operator 
```

**预期效果**

```shell
$ helm install clickhouse-operator ck/clickhouse-operator -n kube-system
NAME: clickhouse-operator
LAST DEPLOYED: Wed Aug 17 14:43:44 2021
NAMESPACE: kube-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

> **注意**
> 
> 上述示例 ClickHouse Operator 将会被安装在 `kube-system` 命名空间下，因此一个 Kubernetes 集群只需要安装一次 ClickHouse Operator。

### 步骤 3 : 部署 RadonDB ClickHouse 集群

```bash
$ helm install --generate-name <repoName>/clickhouse-cluster -n <nameSpace>\
  --set <para_name>=<para_value>
```

更多参数说明，请参见 [配置](#配置)。

**预期效果**

```shell
$ helm install clickhouse ck/clickhouse-cluster -n test
NAME: clickhouse
LAST DEPLOYED: Wed Aug 17 14:48:12 2021
NAMESPACE: test
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

### 步骤 4 : 部署校验

#### 查看 Pod 运行状态

执行如下命令，查看创建的集群 Pod 运行状态。

```bash
$ kubectl get pods -n <nameSpace>
```

**预期结果**

```shell
$ kubectl get pods -n test
NAME                                READY   STATUS    RESTARTS   AGE
pod/chi-ClickHouse-replicas-0-0-0   2/2     Running   0          3m13s
pod/chi-ClickHouse-replicas-0-1-0   2/2     Running   0          2m51s
pod/zk-clickhouse-cluster-0         1/1     Running   0          3m13s
pod/zk-clickhouse-cluster-1         1/1     Running   0          3m13s
pod/zk-clickhouse-cluster-2         1/1     Running   0          3m13s
```

#### 查询 SVC 运行状态

执行如行命令，查看集群 SVC 运行状态。

```bash
$ kubectl get service -n <nameSpace>
```

**预期结果**

```shell
$ kubectl get service -n test
NAME                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
service/chi-ClickHouse-replicas-0-0   ClusterIP   None            <none>        8123/TCP,9000/TCP,9009/TCP      2m53s
service/chi-ClickHouse-replicas-0-1   ClusterIP   None            <none>        8123/TCP,9000/TCP,9009/TCP      2m36s
service/clickhouse-ClickHouse         ClusterIP   10.96.137.152   <none>        8123/TCP,9000/TCP               3m14s
service/zk-client-clickhouse-cluster  ClusterIP   10.107.33.51    <none>        2181/TCP,7000/TCP               3m13s
service/zk-server-clickhouse-cluster  ClusterIP   None            <none>        2888/TCP,3888/TCP               3m13s
```

## 访问 RadonDB ClickHouse

### 通过 Pod

通过 `kubectl` 工具直接访问 ClickHouse Pod。

```bash
$ kubectl exec -it <podName> -n <nameSpace> -- clickhouse-client --user=<userName> --password=<userPassword>
```

**预期效果**

```shell
$ kubectl get pods |grep clickhouse
chi-ClickHouse-replicas-0-0-0   1/1     Running   0          8m50s
chi-ClickHouse-replicas-0-1-0   1/1     Running   0          8m50s

$ kubectl exec -it chi-ClickHouse-replicas-0-0-0 -- clickhouse-client -u clickhouse --password=c1ickh0use0perator --query='select hostName()'
chi-ClickHouse-replicas-0-0-0
```

### 通过 Service

由于 Service 的`spec.type` 类型为 `ClusterIP`，故需通过客户端连接服务。

**预期效果**

```
$ kubectl get service |grep clickhouse
clickhouse-ClickHouse            ClusterIP   10.96.137.152   <none>        9000/TCP,8123/TCP   12m
chi-ClickHouse-replicas-0-0      ClusterIP   None            <none>        9000/TCP,8123/TCP   12m
chi-ClickHouse-replicas-0-1      ClusterIP   None            <none>        9000/TCP,8123/TCP   12m

$ kubectl exec -it clickhouse-ClickHouse -- clickhouse-client -u clickhouse --password=c1ickh0use0perator -h 10.96.137.152 --query='select hostName()'
chi-ClickHouse-replicas-0-1-0
$ kubectl exec -it clickhouse-ClickHouse -- clickhouse-client -u clickhouse --password=c1ickh0use0perator -h 10.96.137.152 --query='select hostName()'
chi-ClickHouse-replicas-0-0-0
```

## 持久化

配置 Pod 使用 PersistentVolumeClaim 作为存储，实现 ClickHouse 持久化。

默认情况下，每个 Pod 将创建一个 PVC ，并将其挂载到 `/var/lib/clickhouse` 目录。

1. 创建一个使用 PVC 作为存储的 Pod。
2. 创建一个 PVC 自动绑定到合适的 PersistentVolume。

> **注意**
> 
> 在 PersistentVolumeClaim 中，可以配置不同特性的 PersistentVolume。

## 配置

|参数 |  描述 |  默认值 |
|:----|:----|:----|
|   **ClickHouse**   |     |    |
|   `clickhouse.clusterName`   |  ClickHouse cluster name. | all-nodes  |
|   `clickhouse.shardscount`   |  Shards count. Once confirmed, it cannot be reduced.  |   1  |
|   `clickhouse.replicascount`   |  Replicas count. Once confirmed, it cannot be modified.  |   2  |
|   `clickhouse.image`   |  ClickHouse image name, it is not recommended to modify.  | radondb/clickhouse-server:v21.1.3.32-stable  |
|   `clickhouse.imagePullPolicy`   |  Image pull policy. The value can be Always/IfNotPresent/Never.  | IfNotPresent  |
|   `clickhouse.resources.memory`   |  K8s memory resources should be requested by a single Pod.  |  1Gi |
|   `clickhouse.resources.cpu`   |  K8s CPU resources should be requested by a single Pod.  |  0.5 |
|   `clickhouse.resources.storage`   |  K8s Storage resources should be requested by a single Pod.  |  10Gi  |
|   `clickhouse.user`   |  ClickHouse user array. Each user needs to contain a username, password and networks array.  | [{"username": "clickhouse", "password": "c1ickh0use0perator", "networks": ["127.0.0.1", "::/0"]}]  |
|   `clickhouse.port.tcp`   |  Port for the native interface.  |  9000  |
|   `clickhouse.port.http`   |  Port for HTTP/REST interface.  |  8123  |
|   `clickhouse.svc.type`   |  K8s service type. The value can be ClusterIP/NodePort/LoadBalancer.  |  ClusterIP  |
|   `clickhouse.svc.qceip`   |  If the value of type is LoadBalancer, You need to configure loadbalancer that provided by third-party platforms.     |  nil   |
|   **ZooKeeper**   |     |    |
|   `zookeeper.install`   |  Whether to create ZooKeeper by operator.  |  true  |
|   `zookeeper.port`   |  ZooKeeper service port.   |  2181  |
|   `zookeeper.replicas`   |  ZooKeeper cluster replicas count.  |  3  |
|   `zookeeper.image`   |  ZooKeeper image name, it is not recommended to modify.  |  Deprecated, if install = true  |
|   `zookeeper.imagePullPolicy`   |  Image pull policy. The value can be Always/IfNotPresent/Never.  |  Deprecated, if install = true  |
|   `zookeeper.resources.memory`   |  K8s memory resources should be requested by a single Pod.  | Deprecated, if install = true  |
|   `zookeeper.resources.cpu`   |  K8s CPU resources should be requested by a single Pod.  |  Deprecated, if install = true  |
|   `zookeeper.resources.storage`   |  K8s storage resources should be requested by a single Pod.  |  Deprecated, if install = true  |
