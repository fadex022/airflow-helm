# Apache Airflow on Kubernetes Deployment Guide

This guide outlines the configuration and deployment of Apache Airflow using CeleryExecutor, a standalone Redis broker, and Git-Sync for DAG management.

---

## Deployment Process

Execute the following commands in sequence to initialize the environment:

```bash
# 1. Create the dedicated namespace
kubectl create ns airflow

# 2. Deploy the standalone Redis instance
kubectl apply -f redis.yaml

# 3. Deploy or Upgrade Airflow via Helm
helm upgrade --install airflow apache-airflow/airflow \
  --namespace airflow \
  --values values.yaml --create-namespace

```

---

## Infrastructure Architecture

The setup is optimized for local or restricted Kubernetes environments with the following technical components:

* **Executor**: `CeleryExecutor` manages distributed task execution across worker pods.
* **Message Broker**: A manual Redis deployment is used to ensure stable connection strings and lifecycle management outside of the Airflow Helm chart.
* **Database**: PostgreSQL handles metadata storage.
* **DAG Synchronization**: A Git-Sync sidecar container automatically pulls DAG files from a remote GitHub repository every 60 seconds.

---

## Key Configuration Parameters (values.yaml)

To resolve common issues such as missing logs or task heartbeats, the following settings are implemented:

| Category | Parameter | Value | Purpose |
| --- | --- | --- | --- |
| **Networking** | `workers.workerService.enabled` | `true` | Enables internal DNS resolution so the Webserver can locate Workers for log streaming. |
| **Logging** | `config.logging.worker_log_server_port` | `8793` | Sets the internal port for the worker log server. |
| **Logging** | `config.celery.worker_hostname_dns_check` | `True` | Forces workers to report their full DNS hostname to the metadata database. |
| **Storage** | `logs.persistence.accessMode` | `ReadWriteOnce` | Ensures compatibility with standard local-path storage provisioners. |
| **Resources** | `workers.resources.limits.memory` | `2Gi` | Prevents task failures caused by Out-Of-Memory (OOM) kills during Python execution. |

---

## Operational Commands

### Accessing the Web Interface

Use port-forwarding to tunnel the internal service to your local machine:

```bash
kubectl port-forward svc/airflow-webserver 8080:8080 -n airflow

```

The UI will be accessible at `http://localhost:8080`.

### Forcing a DAG Refresh

To bypass the 60-second wait and force Git-Sync to pull the latest changes immediately, restart the worker pods:

```bash
kubectl delete pods -n airflow -l component=worker

```

### Resetting the Message Broker

If tasks remain in a persistent "Queued" state, clear the Redis message queue:

```bash
kubectl exec -it <redis-pod-name> -n airflow -- redis-cli flushall

```

---

### Next Step

Would you like me to provide a template for a **Kubernetes NetworkPolicy** to secure communications between the Airflow components and the Redis broker?