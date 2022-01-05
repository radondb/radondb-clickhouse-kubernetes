# Deploy Radondb ClickHouse On Kubernetes

## Introduction

RadonDB ClickHouse is an open-source, cloud-native, highly availability cluster solutions based on [ClickHouse](https://clickhouse.tech/). It provides features such as high availability, PB storage, real-time analytical, architectural stability and scalability.

This tutorial demonstrates how to deploy RadonDB ClickHouse on Kubernetes.

## Prerequisites

- You have created a Kubernetes cluster.
- You have installed [ClickHouse Operator](/clickhouse-operator/README.md).

## Procedure

### Step 1 : Add Helm Repository

Add and update helm repository.

```bash
$ helm repo add <repoName> https://radondb.github.io/radondb-clickhouse-kubernetes/
$ helm repo update
```

**Expected output**

```bash
$ helm repo add ck https://radondb.github.io/radondb-clickhouse-kubernetes/
"ck" has been added to your repositories

$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "ck" chart repository
Update Complete. ⎈Happy Helming!⎈
```

### Step 2 : Install RadonDB ClickHouse Cluster

```bash
$ helm install --generate-name <repoName>/clickhouse-cluster -n <nameSpace>\
  --set <para_name>=<para_value>
```

- For more information about cluter parameters, see [Configuration](#configuration).
- If you need to customize many parameters, you can modify `values.yaml` file. For details, see [Custom Configuration](#custom-configuration).

**Expected output**

```bash
$ helm install clickhouse ck/clickhouse-cluster -n test
NAME: clickhouse
LAST DEPLOYED: Wed Aug 17 14:48:12 2021
NAMESPACE: test
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

### Step 3 : Verification

#### Check the Status of Pod

```bash
$ kubectl get pods -n <nameSpace>
```

**Expected output**

```bash
$ kubectl get pods -n test
NAME                                READY   STATUS    RESTARTS   AGE
pod/chi-ClickHouse-replicas-0-0-0   2/2     Running   0          3m13s
pod/chi-ClickHouse-replicas-0-1-0   2/2     Running   0          2m51s
pod/zk-clickhouse-cluster-0         1/1     Running   0          3m13s
pod/zk-clickhouse-cluster-1         1/1     Running   0          3m13s
pod/zk-clickhouse-cluster-2         1/1     Running   0          3m13s
```

#### Check the Status of SVC

```bash
$ kubectl get service -n <nameSpace>
```

**Expected output**

```bash
$ kubectl get service -n test
NAME                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
service/chi-ClickHouse-replicas-0-0   ClusterIP   None            <none>        8123/TCP,9000/TCP,9009/TCP      2m53s
service/chi-ClickHouse-replicas-0-1   ClusterIP   None            <none>        8123/TCP,9000/TCP,9009/TCP      2m36s
service/clickhouse-ClickHouse         ClusterIP   10.96.137.152   <none>        8123/TCP,9000/TCP               3m14s
service/zk-client-clickhouse-cluster  ClusterIP   10.107.33.51    <none>        2181/TCP,7000/TCP               3m13s
service/zk-server-clickhouse-cluster  ClusterIP   None            <none>        2888/TCP,3888/TCP               3m13s
```

## Access RadonDB ClickHouse

### Use Pod

You can directly connect to ClickHouse Pod with `kubectl`.

```bash
$ kubectl exec -it <podName> -n <nameSpace> -- clickhouse-client --user=<userName> --password=<userPassword>
```

**Expected output**

```bash
$ kubectl get pods | grep clickhouse
chi-ClickHouse-replicas-0-0-0   1/1     Running   0          8m50s
chi-ClickHouse-replicas-0-1-0   1/1     Running   0          8m50s

$ kubectl exec -it chi-ClickHouse-replicas-0-0-0 -- clickhouse-client -u clickhouse --password=c1ickh0use0perator --query='select hostName()'
chi-ClickHouse-replicas-0-0-0
```

### Use Service

```bash
$ echo <query> | curl 'http://<username>:<password>@<svcIP>:<HTTPPort>/' --data-binary @-
```

**Expected output**

```bash
$ kubectl get service | grep clickhouse
clickhouse-ClickHouse            ClusterIP   10.96.137.152   <none>        9000/TCP,8123/TCP   12m
chi-ClickHouse-replicas-0-0      ClusterIP   None            <none>        9000/TCP,8123/TCP   12m
chi-ClickHouse-replicas-0-1      ClusterIP   None            <none>        9000/TCP,8123/TCP   12m

$ echo 'select hostname()' | curl 'http://clickhouse:c1ickh0use0perator@10.96.137.152:8123/' --data-binary @-
chi-ClickHouse-replicas-0-1-0
$ echo 'select hostname()' | curl 'http://clickhouse:c1ickh0use0perator@10.96.137.152:8123/' --data-binary @-
chi-ClickHouse-replicas-0-0-0
```

## Persistence

You can configure a Pod to use a PersistentVolumeClaim(PVC) for storage.
In default, PVC mount on the `/var/lib/clickhouse` directory.

1. You should create a Pod that uses the above PVC for storage.

2. You should create a PVC that is automatically bound to a suitable PersistentVolume(PV).

> **Notices**
>
> PVC can use different PV, so using the different PV show the different performance.

## Configuration

| Parameter |  Description  |  Default Value |
|:----|:----|:----|
|   **ClickHouse**   |     |    |
|   `clickhouse.clusterName`   |  ClickHouse cluster name. | all-nodes  |
|   `clickhouse.shardscount`   |  Shards count. Once confirmed, it cannot be reduced.  |   1  |
|   `clickhouse.replicascount`   |  Replicas count. Once confirmed, it cannot be modified.  |   2  |
|   `clickhouse.image`   |  ClickHouse image name, it is not recommended to modify.  | radondb/clickhouse-server:21.1.3.32  |
|   `clickhouse.imagePullPolicy`   |  Image pull policy. The value can be Always/IfNotPresent/Never.  | IfNotPresent  |
|   `clickhouse.resources.memory`   |  K8s memory resources should be requested by a single Pod.  |  1Gi |
|   `clickhouse.resources.cpu`   |  K8s CPU resources should be requested by a single Pod.  |  0.5 |
|   `clickhouse.resources.storage`   |  K8s Storage resources should be requested by a single Pod.  |  10Gi  |
|   `clickhouse.user`   |  ClickHouse user array. Each user needs to contain a username, password and networks array.  | [{"username": "clickhouse", "password": "c1ickh0use0perator", "networks": ["127.0.0.1", "::/0"]}]  |
|   `clickhouse.port.tcp`   |  Port for the native interface.  |  9000  |
|   `clickhouse.port.http`   |  Port for HTTP/REST interface.  |  8123  |
|   `clickhouse.svc.type`   |  K8s service type. The value can be ClusterIP/NodePort/LoadBalancer.  |  ClusterIP  |
|   **Backup**   |     |    |
|   `backup.on`   |  Whether to enable backup and restore.  |  true  |
|   `backup.image`   |  ClickHouse-Backup image name, it is not recommended to modify.  |  radondb/clickhouse-backup:latest  |
|   `backup.imagePullPolicy`   |  Image pull policy. The value can be Always/IfNotPresent/Never.  |  IfNotPresent  |
|   `backup.s3EndPoint`   |  Object storage endpoint required for backup.  |    |
|   `backup.s3Bucket`   |  Object storage bucket required for backup.  |    |
|   `backup.s3Path`   |  Object storage path required for backup.  |    |
|   `backup.s3AccessKey`   |  Object storage access key required for backup.  |    |
|   `backup.s3SecretKey`   |  Object storage sercret key required for backup.  |    |
|   `backup.backupKeep`   |  The number of backups to keep, beyond which stale backups will be deleted  |  "3"  |
|   **BusyBox**   |     |    |
|   `busybox.image`   |  BusyBox image name, it is not recommended to modify.  |  busybox  |
|   `busybox.imagePullPolicy`   |  Image pull policy. The value can be Always/IfNotPresent/Never.  |  IfNotPresent  |
|   **ZooKeeper**   |     |    |
|   `zookeeper.install`   |  Whether to create ZooKeeper by operator.  |  true  |
|   `zookeeper.port`   |  ZooKeeper service port.   |  2181  |
|   `zookeeper.replicas`   |  ZooKeeper cluster replicas count.  |  3  |
|   `zookeeper.image`   |  ZooKeeper image name, it is not recommended to modify.  |  radondb/zookeeper:3.6.1  |
|   `zookeeper.imagePullPolicy`   |  Image pull policy. The value can be Always/IfNotPresent/Never.  |  IfNotPresent  |
|   `zookeeper.resources.memory`   |  K8s memory resources should be requested by a single Pod.  | Deprecated, if install = true  |
|   `zookeeper.resources.cpu`   |  K8s CPU resources should be requested by a single Pod.  |  Deprecated, if install = true  |
|   `zookeeper.resources.storage`   |  K8s storage resources should be requested by a single Pod.  |  Deprecated, if install = true  |

## Custom Configuration

If you need to customize many parameters, you can modify [values.yaml](values.yaml).

1. Download the `values.yaml` file.
2. Modify the parameter values in the `values.yaml`.
3. Run the following command to deploy the cluster.

```bash
$ helm install --generate-name <repoName>/clickhouse-cluster -n <nameSpace>\
  -f /<path>/to/values.yaml
```

## License

This helm chart is published under the Apache License, Version 2.0.
See [LICENSE](../LICENSE) for more information.

Copyright (c) by [RadonDB](https://github.com/radondb).

### Attributions

* **ClickHouse**
    * Project URL: https://clickhouse.tech/
    * License: Apache License, Version 2.0

<p align="center">
<br/><br/>
Please submit any RadonDB ClickHouse bugs, issues, and feature requests to GitHub Issue.
<br/>
</p>
