# Backup and Snapshot in Kubernetes (etcd & Application Databases)

## Overview

In Kubernetes, "backup" typically refers to two distinct things:

| What | Why |
|---|---|
| **etcd backup** | Backs up the entire cluster state (all objects, configs, secrets) |
| **Application DB backup** | Backs up data inside a database Pod (MySQL, PostgreSQL, MongoDB, etc.) |

Both are critical for disaster recovery.

---

## Part 1 — etcd Backup (Cluster State)

### What is etcd?

etcd is the key-value store that holds the entire Kubernetes cluster state — every Pod, Deployment, ConfigMap, Secret, RBAC rule, and more. If etcd is lost without a backup, the entire cluster configuration is gone.

### Where etcd runs

In a kubeadm cluster, etcd runs as a static Pod on the control plane node:

```bash
kubectl get pods -n kube-system | grep etcd
# etcd-controlplane   1/1   Running   0   10d
```

### etcd backup using etcdctl

```bash
# Set environment variables
export ETCDCTL_API=3

# Take a snapshot
etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### Verify the snapshot

```bash
etcdctl snapshot status /backup/etcd-snapshot.db \
  --write-out=table
```

Output:
```
+----------+----------+------------+------------+
|   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
+----------+----------+------------+------------+
| abc12345 |    12345 |       1200 |     3.2 MB |
+----------+----------+------------+------------+
```

### Restore etcd from snapshot

```bash
# Stop the API server first (on kubeadm, move the static pod manifest)
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# Restore
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Update etcd static pod to use the new data directory
# Edit /etc/kubernetes/manifests/etcd.yaml
# Change: --data-dir=/var/lib/etcd
# To:     --data-dir=/var/lib/etcd-restore

# Also update the hostPath volume in the same file:
# path: /var/lib/etcd  →  path: /var/lib/etcd-restore

# Restart kubelet
systemctl restart kubelet

# Restore the API server manifest
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

### Finding etcd certificates (on kubeadm clusters)

```bash
# View etcd pod to find cert paths
kubectl describe pod etcd-controlplane -n kube-system

# Or check the static pod manifest
cat /etc/kubernetes/manifests/etcd.yaml | grep -E "cert|key|ca|data-dir"
```

### Key certificate paths (kubeadm defaults)

| Parameter | Default Path |
|---|---|
| `--cacert` | `/etc/kubernetes/pki/etcd/ca.crt` |
| `--cert` | `/etc/kubernetes/pki/etcd/server.crt` |
| `--key` | `/etc/kubernetes/pki/etcd/server.key` |
| `--data-dir` | `/var/lib/etcd` |
| `--endpoints` | `https://127.0.0.1:2379` |

---

## Part 2 — Application Database Backup

### MySQL / MariaDB Backup

#### Using mysqldump inside a running Pod

```bash
# Dump a single database
kubectl exec -it <mysql-pod> -n <namespace> -- \
  mysqldump -u root -p<password> mydb > /backup/mydb-$(date +%F).sql

# Dump all databases
kubectl exec -it <mysql-pod> -n <namespace> -- \
  mysqldump -u root -p<password> --all-databases > /backup/all-dbs-$(date +%F).sql
```

#### Using a CronJob for scheduled backups

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
  namespace: production
spec:
  schedule: "0 2 * * *"   # every day at 2am
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: mysql-backup
              image: mysql:8.0
              env:
                - name: MYSQL_PWD
                  valueFrom:
                    secretKeyRef:
                      name: mysql-secret
                      key: password
              command:
                - /bin/sh
                - -c
                - |
                  mysqldump -h mysql-service -u root mydb \
                  > /backup/mydb-$(date +%F-%H%M).sql
              volumeMounts:
                - name: backup-vol
                  mountPath: /backup
          restartPolicy: OnFailure
          volumes:
            - name: backup-vol
              persistentVolumeClaim:
                claimName: backup-pvc
```

---

### PostgreSQL Backup

#### Using pg_dump inside a running Pod

```bash
# Dump a single database
kubectl exec -it <postgres-pod> -n <namespace> -- \
  pg_dump -U postgres mydb > /backup/mydb-$(date +%F).sql

# Dump in custom format (compressed)
kubectl exec -it <postgres-pod> -n <namespace> -- \
  pg_dump -U postgres -Fc mydb > /backup/mydb-$(date +%F).dump

# Dump all databases
kubectl exec -it <postgres-pod> -n <namespace> -- \
  pg_dumpall -U postgres > /backup/all-dbs-$(date +%F).sql
```

#### CronJob for PostgreSQL

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
  namespace: production
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: postgres-backup
              image: postgres:15
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-secret
                      key: password
              command:
                - /bin/sh
                - -c
                - |
                  pg_dump -h postgres-service -U postgres mydb \
                  > /backup/mydb-$(date +%F-%H%M).sql
              volumeMounts:
                - name: backup-vol
                  mountPath: /backup
          restartPolicy: OnFailure
          volumes:
            - name: backup-vol
              persistentVolumeClaim:
                claimName: backup-pvc
```

---

### MongoDB Backup

```bash
# mongodump inside a running Pod
kubectl exec -it <mongo-pod> -n <namespace> -- \
  mongodump --db mydb --out /backup/mongo-$(date +%F)

# Restore
kubectl exec -it <mongo-pod> -n <namespace> -- \
  mongorestore --db mydb /backup/mongo-2024-01-01/mydb
```

---

## Part 3 — Volume Snapshots (PVC Snapshots)

Kubernetes supports **VolumeSnapshots** — point-in-time snapshots of PersistentVolumeClaims. Requires a CSI driver that supports snapshots (e.g., AWS EBS CSI, GCE PD CSI, Longhorn).

### VolumeSnapshotClass

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-aws-vsc
driver: ebs.csi.aws.com
deletionPolicy: Delete
```

### Creating a VolumeSnapshot

```yaml
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: mysql-data-snapshot
  namespace: production
spec:
  volumeSnapshotClassName: csi-aws-vsc
  source:
    persistentVolumeClaimName: mysql-data-pvc
```

```bash
kubectl apply -f snapshot.yaml

# Check snapshot status
kubectl get volumesnapshot -n production
kubectl describe volumesnapshot mysql-data-snapshot -n production
```

### Restoring from a VolumeSnapshot

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data-restored
  namespace: production
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  dataSource:
    name: mysql-data-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

---

## Backup Strategies

### 3-2-1 Rule
- **3** copies of data
- **2** on different media
- **1** offsite (S3, GCS, Azure Blob)

### Approaches

| Strategy | Tool | Best For |
|---|---|---|
| etcd snapshot | `etcdctl snapshot save` | Cluster state recovery |
| Logical dump | `mysqldump`, `pg_dump` | DB schema + data, portable |
| Volume snapshot | CSI VolumeSnapshot | Fast, consistent disk-level backup |
| Velero | Velero (open source) | Full cluster backup (resources + volumes) |

### Velero — Full Cluster Backup

Velero backs up all Kubernetes resources AND PVC data to object storage (S3, GCS, etc.).

```bash
# Install velero (example with AWS S3)
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.7.0 \
  --bucket my-backup-bucket \
  --secret-file ./credentials-velero \
  --backup-location-config region=us-east-1

# Create a backup
velero backup create my-cluster-backup --include-namespaces production

# Schedule a backup
velero schedule create daily-backup \
  --schedule="0 1 * * *" \
  --include-namespaces production

# Restore from backup
velero restore create --from-backup my-cluster-backup

# Check backup status
velero backup describe my-cluster-backup
velero backup logs my-cluster-backup
```

---

## Backup vs Snapshot Comparison

| | Logical Backup (dump) | Volume Snapshot | etcd Snapshot |
|---|---|---|---|
| **What it captures** | DB schema + data as SQL/binary | Raw disk blocks | Entire cluster state |
| **Speed** | Slow (large DBs) | Very fast | Fast |
| **Consistency** | Application-level | Crash-consistent | Cluster-consistent |
| **Portability** | High (SQL is portable) | Low (tied to CSI driver) | Low (Kubernetes only) |
| **Restore time** | Slow | Fast | Medium |
| **Use for** | Data migration, archival | Fast recovery | Cluster DR |

---

## Useful Commands Summary

```bash
# etcd backup
etcdctl snapshot save <path> --endpoints=... --cacert=... --cert=... --key=...

# etcd verify
etcdctl snapshot status <path> --write-out=table

# etcd restore
etcdctl snapshot restore <path> --data-dir=<new-dir>

# MySQL dump from pod
kubectl exec -it <pod> -- mysqldump -u root -p<pass> <db> > backup.sql

# MySQL restore to pod
kubectl exec -i <pod> -- mysql -u root -p<pass> <db> < backup.sql

# PostgreSQL dump
kubectl exec -it <pod> -- pg_dump -U postgres <db> > backup.sql

# PostgreSQL restore
kubectl exec -i <pod> -- psql -U postgres <db> < backup.sql

# List volume snapshots
kubectl get volumesnapshot -A

# Velero backup
velero backup create <name> --include-namespaces <ns>
velero restore create --from-backup <name>
```

---

## Interview Questions

### Conceptual

**Q: What is etcd and why is it important to back it up?**
A: etcd is the key-value store that holds the entire Kubernetes cluster state — every object, configuration, secret, and RBAC rule. Without a backup, a failed etcd means total loss of cluster state. The application workloads may still run, but the cluster cannot be managed or recovered.

**Q: What tool is used to back up etcd?**
A: `etcdctl` with the `snapshot save` command. You must set `ETCDCTL_API=3` and provide the etcd TLS certificate paths.

**Q: What is the difference between an etcd backup and a database backup?**
A: An etcd backup captures Kubernetes cluster state (Pods, Deployments, ConfigMaps, Secrets, etc.) — not the application data stored inside database Pods. A database backup (mysqldump, pg_dump) captures the actual data inside the running database.

**Q: What are VolumeSnapshots in Kubernetes?**
A: VolumeSnapshots are point-in-time copies of a PersistentVolumeClaim at the storage layer. They require a CSI driver that supports snapshots and are faster than logical database dumps.

**Q: What is the difference between a logical backup and a volume snapshot?**
A: A logical backup (e.g., `mysqldump`) exports data as portable SQL or binary files — slow but highly portable. A volume snapshot copies raw disk blocks — fast but tied to the storage driver and not directly portable across providers.

**Q: What is Velero and when would you use it?**
A: Velero is an open-source tool for backing up and restoring entire Kubernetes namespaces — both the resource definitions and PVC data — to object storage like S3 or GCS. Use it for full cluster disaster recovery or namespace migration.

**Q: What certificates are required for etcdctl to take a snapshot?**
A: The etcd CA certificate (`--cacert`), the server certificate (`--cert`), and the server key (`--key`). All are typically found under `/etc/kubernetes/pki/etcd/` on kubeadm clusters.

**Q: What must you do before restoring etcd from a snapshot on a kubeadm cluster?**
A: Stop the API server (move its static pod manifest out of `/etc/kubernetes/manifests/`), run `etcdctl snapshot restore` pointing to a new data directory, update the `etcd.yaml` manifest to use that new directory, and restart kubelet.

### Scenario-Based

**Q: The etcd data directory is corrupted. You have a snapshot at `/backup/etcd-snapshot.db`. Walk through the restore.**
A:
```bash
# 1. Stop the API server
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# 2. Restore snapshot to new directory
etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restore \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 3. Update /etc/kubernetes/manifests/etcd.yaml
#    --data-dir=/var/lib/etcd-restore
#    hostPath.path: /var/lib/etcd-restore

# 4. Restart kubelet
systemctl restart kubelet

# 5. Restore API server
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
```

**Q: You need to back up a MySQL database running in Kubernetes every night at midnight. How do you implement this?**
A: Create a Kubernetes CronJob with schedule `"0 0 * * *"` that runs a container with `mysqldump`, writes the output to a PVC or pushes it to S3, and uses a Kubernetes Secret for the database password.

**Q: A developer accidentally deleted a namespace with all its resources. How do you recover?**
A: If Velero is configured, run `velero restore create --from-backup <latest-backup> --include-namespaces <ns>`. If only etcd backup is available, restore etcd from the latest snapshot. If neither is available, the resources are gone — this is why regular backups are critical.

**Q: How do you verify an etcd backup is valid without restoring it to production?**
A:
```bash
etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table
```
This shows the hash, revision, total keys, and size — confirming the snapshot is not corrupted.

**Q: How would you back up a PostgreSQL database running as a StatefulSet and store the backup in S3?**
A: Create a CronJob that runs `pg_dump`, pipes the output to `aws s3 cp` or writes to a PVC that syncs to S3. Use a Kubernetes Secret for both the PostgreSQL password and AWS credentials. Example:
```bash
pg_dump -h postgres-svc -U postgres mydb | \
  aws s3 cp - s3://my-backup-bucket/mydb-$(date +%F).sql
```

---

## Summary

| Topic | Key Point |
|---|---|
| etcd backup | `etcdctl snapshot save` with TLS certs |
| etcd restore | Stop API server → restore → update data-dir → restart kubelet |
| MySQL backup | `mysqldump` via `kubectl exec` or CronJob |
| PostgreSQL backup | `pg_dump` via `kubectl exec` or CronJob |
| Volume snapshot | CSI-based, fast, disk-level, requires VolumeSnapshotClass |
| Velero | Full cluster backup (resources + volumes) to object storage |
| Best practice | Automate with CronJobs, store offsite, verify regularly |
