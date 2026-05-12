# Zalando Postgres Operator with MinIO S3 Backup Integration

This repository contains the configuration and deployment manifests for a Highly Available PostgreSQL cluster managed by the Zalando Postgres Operator, integrated with a local MinIO instance for automated WAL archiving and backups.

## Architecture Overview

The solution is designed with security and Kubernetes best practices in mind:
* **PostgreSQL Cluster:** Managed via Zalando Spilo, utilizing `WAL-G` for backup and restoration.
* **Storage:** A local S3-compatible MinIO deployment acts as the backup backend.
* **Security by Design:** Plain-text credentials are intentionally avoided in the database custom resource. S3 credentials are stored securely in a Kubernetes `Secret` and dynamically injected into the database pods via the Operator's ConfigMap (`pod_environment_secret`).
* **Encryption Handling:** Server-Side Encryption (SSE) is deliberately disabled via `WALG_DISABLE_S3_SSE: "true"` to allow seamless backups to the local test environment lacking an external Key Management Service (KMS).

## Prerequisites
* A running Kubernetes cluster (e.g., `kind`, `minikube`, or Docker Desktop).
* `kubectl` CLI installed and configured.

## Deployment Instructions

To successfully deploy the environment and initialize the automated backup mechanism, please follow the commands in the exact order below.

### 1. Deploy the Local S3 Storage (MinIO)
```bash
kubectl apply -f minio-secret.yaml
kubectl apply -f minio-deployment.yaml
kubectl apply -f minio-service.yaml
```
(Wait until the MinIO pod is in the Running state before proceeding).

### 2. Configure Operator Credentials and Settings
```bash
kubectl apply -f spilo-secret.yaml
kubectl apply -f operator-setup/operatorconfiguration.crd.yaml
kubectl apply -f operator-setup/operator-service-account-rbac.yaml
kubectl apply -f operator-setup/operator-configmap.yaml
kubectl apply -f operator-setup/postgres-operator.yaml
```

### 3. Deploy PostgreSQL Cluster
```bash
kubectl apply -f postgres.yaml
```
(Wait until the cluster pods are fully ready)

### 4. Verify the Backup Mechanism
Note: Before triggering the first backup, ensure the S3 bucket is created:
1. Establish a port-forwarding session to the MinIO UI:
```bash
kubectl port-forward svc/minio-service 9001:9001
```
2. Open your browser and go to: http://localhost:9001.
3. Log in using credentials defined in `minio-secret.yaml`.
4. Create a bucket named `postgres-backups`.
Once the bucket is created and primary database pod (e.g., prod-db-0) is running, you can manually trigger a WAL segment switch to force an immediate backup to MinIO:
```bash
kubectl exec -it prod-db-0 -- su postgres -c "psql -c 'SELECT pg_switch_wal();'"
```
To verify that the backup files have been uploaded:
5. Inspect the `postgres-backups` bucket. There should be a directory `spilo/` with created backup.
<img width="1412" height="517" alt="image" src="https://github.com/user-attachments/assets/0991d383-93ad-4560-974e-ba92a23122ae" />
