# Helm Chart for ClickHouse Operator

## Installation

### Add Helm Repository

```
$ helm repo add ck https://radondb.github.io/radondb-clickhouse-kubernetes/
$ helm repo update

```

### Install to Kubernetes

```
$ helm install --generate-name ck/clickhouse-operator

```

## License

This helm chart is published under the Apache License, Version 2.0. See LICENSE.md for more information.

Copyright (c) by [TCeason](https://tceason.github.io/).
