Contents
=================

- [Deploy Radondb ClickHouse On Kubernetes](#deploy-radondb-clickhouse-on-kubernetes)
  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
  - [Procedure](#procedure)
    - [Step 1 : Add Helm Repository](#step-1--add-helm-repository)
    - [Step 2 :  Install RadonDB ClickHouse Operator](#step-2---install-radondb-clickhouse-operator)
    - [Step 3 :  Install RadonDB ClickHouse Cluster](#step-3---install-radondb-clickhouse-cluster)
    - [Step 4 :  Verification](#step-4---verification)
      - [Check the Status of Pod](#check-the-status-of-pod)
      - [Check the Status of SVC](#check-the-status-of-svc)
  - [Access RadonDB ClickHouse](#access-radondb-clickhouse)
    - [Use pod](#use-pod)
    - [Use Service](#use-service)
  - [Persistence](#persistence)

# Deploy Radondb ClickHouse On Kubernetes

## Introduction

RadonDB ClickHouse is an open-source, cloud-native, highly availability cluster solutions based on [ClickHouse](https://clickhouse.tech/).

This tutorial demonstrates how to deploy RadonDB ClickHouse on Kubernetes.

## Prerequisites

- You have created a Kubernetes Cluster.

## Procedure

### Step 1 : Add Helm Repository

Add and update helm repositor.

```bash
$ helm repo add ck https://radondb.github.io/radondb-clickhouse-kubernetes/
$ helm repo update
```

### Step 2 :  Install RadonDB ClickHouse Operator

```bash
$ helm install clickhouse-operator ck/clickhouse-operator
```

### Step 3 :  Install RadonDB ClickHouse Cluster

```bash
$ helm install clickhouse ck/clickhouse-cluster
```

### Step 4 :  Verification

#### Check the Status of Pod

```bash
kubectl get pods -n <Namespace>
```

**Expected output:**

```shell
$ kubectl get pods -n test
NAME                                READY   STATUS    RESTARTS   AGE
pod/chi-ClickHouse-replicas-0-0-0   2/2     Running   0          3m13s
pod/chi-ClickHouse-replicas-0-1-0   2/2     Running   0          2m51s
pod/chi-ClickHouse-replicas-1-0-0   2/2     Running   0          2m34s
pod/chi-ClickHouse-replicas-1-1-0   2/2     Running   0          2m17s
pod/chi-ClickHouse-replicas-2-0-0   2/2     Running   0          115s
pod/chi-ClickHouse-replicas-2-1-0   2/2     Running   0          48s
pod/zk-clickhouse-cluster-0         1/1     Running   0          3m13s
pod/zk-clickhouse-cluster-1         1/1     Running   0          3m13s
pod/zk-clickhouse-cluster-2         1/1     Running   0          3m13s
```

#### Check the Status of SVC

```bash
$ kubectl get service -n <Namespace>
```

**Expected output:**

```shell
$ kubectl get service -n test
NAME                                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
service/chi-ClickHouse-replicas-0-0   ClusterIP   None            <none>        8123/TCP,9000/TCP,9009/TCP      2m53s
service/chi-ClickHouse-replicas-0-1   ClusterIP   None            <none>        8123/TCP,9000/TCP,9009/TCP      2m36s
service/chi-ClickHouse-replicas-1-0   ClusterIP   None            <none>        8123/TCP,9000/TCP,9009/TCP      2m19s
service/chi-ClickHouse-replicas-1-1   ClusterIP   None            <none>        8123/TCP,9000/TCP,9009/TCP      117s
service/chi-ClickHouse-replicas-2-0   ClusterIP   None            <none>        8123/TCP,9000/TCP,9009/TCP      50s
service/chi-ClickHouse-replicas-2-1   ClusterIP   None            <none>        8123/TCP,9000/TCP,9009/TCP      13s
service/clickhouse-ClickHouse         ClusterIP   10.96.137.152   <none>        8123/TCP,9000/TCP               3m14s
service/zk-client-clickhouse-cluster  ClusterIP   10.107.33.51    <none>        2181/TCP,7000/TCP               3m13s
service/zk-server-clickhouse-cluster  ClusterIP   None            <none>        2888/TCP,3888/TCP               3m13s
```

## Access RadonDB ClickHouse

### Use pod

You can directly connect to ClickHouse Pod with `kubectl`.

```bash
kubectl exec -it <pod name> -n <project name> -- clickhouse-client --user=<user name> --password=<user password>
```

**Expected output:**

```shell
$ kubectl get pods |grep clickhouse
clickhouse-s0-r0-0   1/1     Running   0          8m50s
clickhouse-s0-r1-0   1/1     Running   0          8m50s

$ kubectl exec -it clickhouse-s0-r0-0 -- clickhouse client -u default --password=C1ickh0use --query='select hostName()'
clickhouse-s0-r0-
```

### Use Service

The Service `spec.type` is `ClusterIP`, so you need to create a client to connect the service.

**Expected output:**

```shell
$ kubectl get service |grep clickhouse
clickhouse            ClusterIP   10.96.71.193   <none>        9000/TCP,8123/TCP   12m
clickhouse-s0-r0      ClusterIP   10.96.40.207   <none>        9000/TCP,8123/TCP   12m
clickhouse-s0-r1      ClusterIP   10.96.63.179   <none>        9000/TCP,8123/TCP   12m

$ cat client.yaml
apiVersion: v1
kind: Pod
metadata:
name: clickhouse-client
labels:
app: clickhouse-client
spec:
containers:
- name: clickhouse-client
image: tceason/clickhouse-server:v21.1.3.32-stable
imagePullPolicy: Always

$ kubectl apply -f client.yaml
pod/clickhouse-client unchanged

$ kubectl exec -it clickhouse-client -- clickhouse client -u default --password=C1ickh0use -h 10.96.71.193 --query='select hostName()'
clickhouse-s0-r1-0
$ kubectl exec -it clickhouse-client -- clickhouse client -u default --password=C1ickh0use -h 10.96.71.193 --query='select hostName()'
clickhouse-s0-r0-0
```

## Persistence

You can configure a Pod to use a PersistentVolumeClaim(PVC) for storage. 
In default, PVC mount on the `/var/lib/clickhouse` directory.

1. You should create a Pod that uses the above PVC for storage.

2. You should create a PVC that is automatically bound to a suitable PersistentVolume(PV). 

> **Note** 
> PVC can use different PV, so using the different PV show the different performance.
