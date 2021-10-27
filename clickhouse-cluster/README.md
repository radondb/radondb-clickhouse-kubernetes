# Helm Chart for ClickHouse Cluster

[ClickHouse](https://clickhouse.tech/) is an open source column-oriented database management system capable of real time generation of analytical data reports using
SQL queries.
This repository provides a helm chart for easily setting up a ClickHouse cluster in Kubernetes.

## Installation

Before install ClickHouse cluster, you should install [ClickHouse-Operator](../clickhouse-operator/README.md) first.

### Add Helm Repository

```
$ helm repo add ck https://radondb.github.io/radondb-clickhouse-kubernetes/
$ helm repo update

```

### Install to Kubernetes

```
helm install clickhouse ck/clickhouse-cluster
```

For a list of all configurable options and variables see [values.yaml](values.yaml).

## License

This helm chart is published under the Apache License, Version 2.0.
See [LICENSE.md](LICENSE.md) for more information.

Copyright (c) by [RadonDB](https://github.com/radondb).

### Attributions

* **ClickHouse**
    * Project URL: https://clickhouse.tech/
    * License: Apache License, Version 2.0

