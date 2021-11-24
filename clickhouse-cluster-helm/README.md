# Deploy Radondb ClickHouse On kubernetes

## Introduction

RadonDB ClickHouse is an open-source, cloud-native, highly availability cluster solutions based on [ClickHouse](https://clickhouse.tech/). It provides features such as high availability, PB storage, real-time analytical, architectural stability and scalability.

This tutorial demonstrates how to deploy RadonDB ClickHouse on Kubernetes.

## Prerequisites

- You have created a Kubernetes Cluster.

## Procedure

### Step 1 : Add Helm Repository

Add and update helm repositor.

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

### Step 2 : Install to Kubernetes

> Zookeeper store ClickHouse's metadata. You can install Zookeeper and ClickHouse Cluster at the same time.

```bash
$ helm install --generate-name <repoName>/clickhouse-cluster -n <nameSpace> --version v1.0
```

- For more configurable options and variables, see [values.yaml](values.yaml).
- If you need to customize cluster parameters, you can modify `values.yaml` file. For details, see [Custom Configuration](#custom-configuration).

**Expected output:**

```bash
$ helm install clickhouse ck/clickhouse-cluster -n test --version v1.0
NAME: clickhouse
LAST DEPLOYED: Thur June 17 07:55:42 2021
NAMESPACE: test
STATUS: deployed
REVISION: 1
TEST SUITE: None

$ helm list -n test
NAME      	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART             APP VERSION
clickhouse	test     	1       	2021-06-07 07:55:42.860240764 +0000 UTC	deployed	clickhouse-v1.0	  21.1
```

### Step 3 : Verification

#### Check the Pod

```bash
kubectl get all --selector app.kubernetes.io/instance=<ClickHouseClusterName> -n <nameSpace>
```

**Expected output:**

```bash
$ kubectl get all --selector app.kubernetes.io/instance=clickhouse -n test
NAME                     READY   STATUS    RESTARTS   AGE
pod/clickhouse-s0-r0-0   1/1     Running   0          72s
pod/clickhouse-s0-r1-0   1/1     Running   0          72s

NAME                       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
service/clickhouse         ClusterIP   10.96.230.92    <none>        9000/TCP,8123/TCP   72s
service/clickhouse-s0-r0   ClusterIP   10.96.83.41     <none>        9000/TCP,8123/TCP   72s
service/clickhouse-s0-r1   ClusterIP   10.96.240.111   <none>        9000/TCP,8123/TCP   72s

NAME                                READY   AGE
statefulset.apps/clickhouse-s0-r0   1/1     72s
statefulset.apps/clickhouse-s0-r1   1/1     72s
```

#### Check the Status of Pod

You should wait a while，then check the output in `Reason` line. When the output persistently return `Started`, indicate that RadonDB ClickHouse is up and running.

```bash
kubectl describe pod <pod name> -n <nameSpace>
```

**Expected output:**

```bash
$ kubectl describe pod clickhouse-s0-r0-0 -n test
...
Events:
  Type     Reason                  Age                    From                     Message
  ----     ------                  ----                   ----                     -------
  Warning  FailedScheduling        7m30s (x3 over 7m42s)  default-scheduler        error while running "VolumeBinding" filter plugin for pod "clickhouse-s0-r0-0": pod has unbound immediate PersistentVolumeClaims
  Normal   Scheduled               7m28s                  default-scheduler        Successfully assigned default/clickhouse-s0-r0-0 to worker-p004
  Normal   SuccessfulAttachVolume  7m6s                   attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-21c5de1f-c396-4743-a31b-2b094ecaf79b"
  Warning  Unhealthy               5m4s (x3 over 6m4s)    kubelet, worker-p004     Liveness probe failed: Code: 210. DB::NetException: Connection refused (localhost:9000)
  Normal   Killing                 5m4s                   kubelet, worker-p004     Container clickhouse failed liveness probe, will be restarted
  Normal   Pulled                  4m34s (x2 over 6m50s)  kubelet, worker-p004     Container image "tceason/clickhouse-server:v21.1.3.32-stable" already present on machine
  Normal   Created                 4m34s (x2 over 6m50s)  kubelet, worker-p004     Created container clickhouse
  Normal   Started                 4m33s (x2 over 6m48s)  kubelet, worker-p004     Started container clickhouse
```

## Access RadonDB ClickHouse

### Use Pod

You can directly connect to ClickHouse Pod with `kubectl`.

```bash
$ kubectl exec -it <podName> -n <nameSpace> -- clickhouse-client --user=<userName> --password=<userPassword>
```

**Expected output**

```bash
$ kubectl get pods -n test | grep clickhouse
clickhouse-s0-r0-0   1/1     Running   0          8m50s
clickhouse-s0-r1-0   1/1     Running   0          8m50s

$ kubectl exec -it clickhouse-s0-r0-0 -n test -- clickhouse-client -u default --password=C1ickh0use --query='select hostName()'
clickhouse-s0-r0-0
```

### Use Service

```bash
$ echo <query> | curl 'http://<username>:<password>@<svcIP>:<HTTPPort>/' --data-binary @-
```

**Expected output**

```bash
$ kubectl get service -n test | grep clickhouse
clickhouse            ClusterIP   10.96.71.193   <none>        9000/TCP,8123/TCP   12m
clickhouse-s0-r0      ClusterIP   10.96.40.207   <none>        9000/TCP,8123/TCP   12m
clickhouse-s0-r1      ClusterIP   10.96.63.179   <none>        9000/TCP,8123/TCP   12m

$ echo 'select hostname()' | curl 'http://default:C1ickh0use@10.96.71.193:8123/' --data-binary @-
clickhouse-s0-r1-0
$ echo 'select hostname()' | curl 'http://default:C1ickh0use@10.96.71.193:8123/' --data-binary @-
clickhouse-s0-r0-0
```

## Persistence

You can configure a Pod to use a PersistentVolumeClaim(PVC) for storage.
In default, PVC mount on the `/var/lib/clickhouse` directory.

1. You should create a Pod that uses the above PVC for storage.

2. You should create a PVC that is automatically bound to a suitable PersistentVolume(PV).

> **Note**
> PVC can use different PV, so using the different PV show the different performance.

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
