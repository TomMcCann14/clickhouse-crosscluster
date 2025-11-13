# ClickHouse Cross‑Cluster Replication (Writer + Reader) on Kubernetes

This repo contains Kubernetes manifests and a step‑by‑step guide for running
ClickHouse with **cross‑cluster replication**:

- One **Writer** cluster (accepts inserts, reads)
- One **Reader** cluster (read‑only, replicates from writer)
- A shared **ClickHouse Keeper** cluster for metadata / replication
- S3 / MinIO for data storage

> The IPs in the examples (e.g. `79.99.47.121`, `79.99.47.53`) are taken from a
> working environment. **You must change them** to match your own LoadBalancer
> IPs when you deploy this elsewhere.

---

## 0. Layout & Files

```text
k8s/
  infra/
    01-namespace-clickhouse-infra.yaml
    02-keeper-configmap.yaml
    03-keeper-statefulset.yaml
    04-keeper-service-headless.yaml
    05-keeper-service-lb-0.yaml
  writer/
    01-namespace-clickhouse.yaml
    02-service-interserver-writer.yaml
    03-chi-writer.yaml
  reader/
    01-namespace-clickhouse.yaml
    02-service-interserver-reader.yaml
    03-chi-reader.yaml
```

You deploy:

- `k8s/infra/*` once in the **Keeper cluster**
- `k8s/writer/*` in the **Writer** Kubernetes cluster
- `k8s/reader/*` in the **Reader** Kubernetes cluster

---

## 1. Deploy Keeper Cluster (shared ZK) — `k8s/infra`

### 1.1 Create namespace

```bash
kubectl apply -f k8s/infra/01-namespace-clickhouse-infra.yaml
```

### 1.2 Deploy Keeper ConfigMap, StatefulSet & Services

```bash
kubectl apply -f k8s/infra/02-keeper-configmap.yaml
kubectl apply -f k8s/infra/03-keeper-statefulset.yaml
kubectl apply -f k8s/infra/04-keeper-service-headless.yaml
kubectl apply -f k8s/infra/05-keeper-service-lb-0.yaml
```

What this gives you:

- 3‑node `clickhouse-keeper` cluster (statefulset `ch-keeper`)
- Headless service `ch-keeper` for **in‑cluster** access
- `ch-keeper-0-lb` (type `LoadBalancer`) exposing pod `ch-keeper-0` on port 2181

How you use it:

- **Writer** ClickHouse cluster (same Kubernetes cluster as Keeper) points
  at Keeper via **cluster DNS**:

  ```yaml
  zookeeper:
    nodes:
      - host: ch-keeper-0.ch-keeper.clickhouse-infra.svc.cluster.local
        port: 2181
      - host: ch-keeper-1.ch-keeper.clickhouse-infra.svc.cluster.local
        port: 2181
      - host: ch-keeper-2.ch-keeper.clickhouse-infra.svc.cluster.local
        port: 2181
  ```

- **Reader** ClickHouse cluster (different Kubernetes cluster) points at
  Keeper via **public IPs of the Keeper pods / LBs**. You will need to create
  LBs for each Keeper pod (we provide `05-keeper-service-lb-0.yaml` for
  pod `ch-keeper-0`; you can duplicate for `-1` and `-2` if needed).

Once Keeper is up, move on.

---

## 2. Install ClickHouse Operator in Writer & Reader clusters

Do this **once in each** of the Writer and Reader Kubernetes clusters.

```bash
kubectl create namespace clickhouse || true

kubectl apply -f https://raw.githubusercontent.com/Altinity/clickhouse-operator/master/deploy/operator/clickhouse-operator-install-bundle.yaml

kubectl -n kube-system rollout status deploy/clickhouse-operator
```

> The operator runs in `kube-system` and watches the `clickhouse` namespace
> (and others); our CHI manifests live in `clickhouse`.

---

## 3. Writer Cluster (Production)

All commands here are run against the **Writer** cluster context.

### 3.1 Create namespace

```bash
kubectl apply -f k8s/writer/01-namespace-clickhouse.yaml
```

### 3.2 Deploy interserver LoadBalancer

This service exposes **port 9009** and is used for cross‑cluster replication.

```bash
kubectl apply -f k8s/writer/02-service-interserver-writer.yaml
kubectl -n clickhouse get svc ch-interserver
```

Wait until it has an `EXTERNAL-IP`. In our working setup this was:

- `79.99.47.121` → we set this as `interserver_http_host` in the writer CHI.

### 3.3 Edit Writer CHI for your environment

Open `k8s/writer/03-chi-writer.yaml` and adjust:

- **MinIO / S3 settings** inside `config.d/storage.xml`:

  ```xml
  <endpoint>https://s3-api.illapa.cloud/opexx/</endpoint>
  <access_key_id>YOUR_MINIO_ACCESS_KEY</access_key_id>
  <secret_access_key>YOUR_MINIO_SECRET_KEY</secret_access_key>
  ```

- **Interserver host** (must be the external IP or resolvable DNS of the
  `ch-interserver` service in this writer cluster):

  ```yaml
  settings:
    listen_host: 0.0.0.0
    interserver_http_host: 79.99.47.121   # CHANGE FOR YOUR ENV
    interserver_http_port: 9009
  ```

The CHI also:

- Defines `config.d/99-interserver.xml` (explicit override)
- Defines `config.d/storage.xml` (MinIO disk and storage policy `external`)
- Adds `users.d/99-access-override.xml` so the `default` user can manage users
  via SQL (`access_management=1` etc.), which we used to create the `opexx` user.

### 3.4 Apply Writer CHI

```bash
kubectl apply -f k8s/writer/03-chi-writer.yaml
kubectl -n clickhouse get pods -l clickhouse.altinity.com/chi=ch-aa
```

Wait until both pods (e.g. `chi-ch-aa-s1-0-0-0` and `chi-ch-aa-s1-0-1-0`)
are `Running` and `Ready`.

### 3.5 Create SQL user for external access (optional but common)

Exec into one writer pod:

```bash
kubectl -n clickhouse exec -it chi-ch-aa-s1-0-0-0 -- bash
```

Inside the pod:

```bash
clickhouse-client -u default

CREATE USER IF NOT EXISTS opexx
  IDENTIFIED WITH plaintext_password BY 'REPLACE_ME'
  HOST ANY;

GRANT ALL ON *.* TO opexx WITH GRANT OPTION;
```

Use this user for DBeaver / HTTP client connections later.

---

## 4. Reader Cluster (Replica)

All commands here are run against the **Reader** cluster context.

### 4.1 Create namespace

```bash
kubectl apply -f k8s/reader/01-namespace-clickhouse.yaml
```

### 4.2 Deploy interserver LoadBalancer (Reader side)

```bash
kubectl apply -f k8s/reader/02-service-interserver-reader.yaml
kubectl -n clickhouse get svc ch-interserver
```

Wait for the `EXTERNAL-IP`. In our working setup this was:

- `79.99.47.53`   → used as `interserver_http_host` on the reader.

### 4.3 Edit Reader CHI

Open `k8s/reader/03-chi-reader.yaml` and adjust:

- `zookeeper.nodes` to point to the shared Keeper cluster. If your reader
  cannot resolve the Keeper’s internal DNS names, use **public IPs** of
  the Keeper pods / services (e.g. `79.99.47.126`, `109.228.46.240`,
  `109.228.46.241` in our environment).

- MinIO / S3 settings in `config.d/storage.xml` (same bucket/policy as writer).

- Interserver host & port to the **reader** interserver LB external IP:

  ```yaml
  settings:
    listen_host: 0.0.0.0
    interserver_http_host: 79.99.47.53   # CHANGE FOR YOUR ENV
    interserver_http_port: 9009
  ```

The reader CHI also includes `users.d/99-access-override.xml` so you can
manage users, just like on the writer.

### 4.4 Apply Reader CHI

```bash
kubectl apply -f k8s/reader/03-chi-reader.yaml
kubectl -n clickhouse get pods -l clickhouse.altinity.com/chi=ch-aa
```

Wait until both reader pods are `Running` and `Ready`.

---

## 5. Wire Up Replicated Tables (Test)

Pick any simple test database / table.

### 5.1 On the Writer

```bash
kubectl -n clickhouse exec -it chi-ch-aa-s1-0-0-0 -- bash
```

Inside pod:

```bash
clickhouse-client -q "CREATE DATABASE IF NOT EXISTS testdb ENGINE = Atomic;"

clickhouse-client -q "
  CREATE TABLE IF NOT EXISTS testdb.demo
  (
    id  UInt64,
    msg String
  )
  ENGINE = ReplicatedMergeTree('/clickhouse/tables/0/testdb/demo', '{replica}')
  ORDER BY id;
"

clickhouse-client -q "INSERT INTO testdb.demo VALUES (1, 'hello from writer');"
```

Check ZooKeeper on writer:

```bash
clickhouse-client -q "
  SELECT name
  FROM system.zookeeper
  WHERE path = '/clickhouse/tables/0/testdb/demo/replicas';
"
# Expect: chi-ch-aa-s1-0-0 and chi-ch-aa-a3-s1-0-0 once reader is up
```

> Note: The `{replica}` macro and the shared zookeeper path
> `/clickhouse/tables/0/testdb/demo` are what make both clusters share the
> same replication metadata.

### 5.2 On the Reader

```bash
kubectl -n clickhouse exec -it chi-ch-aa-a3-s1-0-0-0 -- bash
```

Inside pod:

```bash
clickhouse-client -q "CREATE DATABASE IF NOT EXISTS testdb ENGINE = Atomic;"

clickhouse-client -q "
  CREATE TABLE IF NOT EXISTS testdb.demo
  (
    id  UInt64,
    msg String
  )
  ENGINE = ReplicatedMergeTree('/clickhouse/tables/0/testdb/demo', '{replica}')
  ORDER BY id;
"

clickhouse-client -q "SYSTEM SYNC REPLICA testdb.demo;"

clickhouse-client -q "SELECT * FROM testdb.demo;"
# Expect: 1	row with 'hello from writer'
```

If you see `last_exception` in `system.replicas` mentioning DNS for a replica
(host not resolvable), double‑check:

- `interserver_http_host` on **both** clusters
- That **writer interserver IP/port** is reachable from the **reader** pods

---

## 6. Quick DBeaver / HTTP Checks

### 6.1 DBeaver (HTTP)

- Driver: ClickHouse
- URL: `http://<WRITER_INTERSERVER_IP>:8123`
- User: `opexx`
- Password: your password

The error `Magic is not correct - expect [-126] but got [...]` usually indicates
that something other than ClickHouse is answering on that port. In our setup
the fix was to ensure:

- Service `clickhouse-external` (or similar) points to **ClickHouse pods**
- Only HTTP traffic is sent to 8123
- DBeaver uses HTTP (not native 9000)

### 6.2 Direct curl test

```bash
curl -u 'opexx:REPLACE_ME' "http://<WRITER_INTERSERVER_IP>:8123/?query=SELECT%201"
```

A correct setup returns:

```text
1
```

---

## 7. Summary of Gotchas We Hit

This repo bakes in the fixes we had to apply:

1. **Default user lacks `access_management`**  
   → we inject `users.d/99-access-override.xml` so `default` can create / grant
   users via SQL.

2. **S3 / MinIO disk not visible**  
   → we inject `config.d/storage.xml` with `<storage_configuration>` and a
   `minio` disk and `external` policy.

3. **`interserver_http_host` wrong / default**  
   → we add `config.d/99-interserver.xml` and a matching `settings` block
   so the interserver host is a real, routable LB IP and not
   `example.clickhouse.com`.

4. **Cross‑cluster replication stuck on DNS**  
   → we ensured both writer and reader point at the **same Keeper** and that
   the `host` stored in ZooKeeper for each replica is reachable from the
   other cluster (via the interserver LB IPs).

Use this repo as a baseline; for a new environment you mainly change:

- MinIO endpoint / credentials
- Keeper endpoints
- Interserver LB IPs
- CHI name / namespace if you want something different
