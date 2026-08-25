---
date: '2026-08-25T12:54:26+08:00'
draft: false
title: "How to Back Up Local Path Provisioner PVCs with Velero FSB"
tags: [ "edge", "k3s", "rancher", "rke2", "ci-cd", "gitops", "kubernetes", "k8s", "backup", "local-path-provisioner", "velero", "fs-backup"]
author: K8S team
---
# How to Back Up Local Path Provisioner PVCs with Velero FSB

Local Path Provisioner is a practical storage choice for edge, development, and lab Kubernetes clusters. It exposes node-local storage through a Kubernetes `StorageClass`, which keeps deployment simple but introduces an important backup consideration: the underlying data is a directory on a node, not a portable block volume.

This guide shows how to back up Local Path Provisioner PVCs with Velero file-system backup (FSB). When a Velero backup contains Kubernetes objects but not PersistentVolumeClaim (PVC) data, the usual cause is a snapshot-based backup configuration. Local Path Provisioner PVCs need Velero's node agent to back up the mounted filesystem to object storage such as MinIO.

This article uses Velero file-system backup (FSB) specifically for PVCs provisioned by Local Path Provisioner. It does not cover direct `hostPath` volumes declared in a pod spec; the application data in this guide is mounted through a PVC.

**In this guide:** configure Local Path Provisioner for FSB, deploy MinIO, install Velero with its node agent, back up a PVC, and restore the workload.

## Contents

- [How to Back Up Local Path Provisioner PVCs with Velero FSB](#how-to-back-up-local-path-provisioner-pvcs-with-velero-fsb)
  - [Contents](#contents)
  - [Prerequisites and architecture](#prerequisites-and-architecture)
  - [Configure Local Path Provisioner for FSB](#configure-local-path-provisioner-for-fsb)
  - [Deploy MinIO as Velero backup storage](#deploy-minio-as-velero-backup-storage)
    - [Install MinIO with Helm](#install-minio-with-helm)
  - [Install Velero with file-system backup](#install-velero-with-file-system-backup)
    - [Deploy Velero with the node agent](#deploy-velero-with-the-node-agent)
  - [Create a Local Path PVC test workload](#create-a-local-path-pvc-test-workload)
  - [Create a Velero FSB backup](#create-a-velero-fsb-backup)
  - [Restore a Local Path PVC from Velero FSB](#restore-a-local-path-pvc-from-velero-fsb)
    - [Create the restore resource](#create-the-restore-resource)
  - [Migrate existing `hostPath` Local Path PVCs to `local`](#migrate-existing-hostpath-local-path-pvcs-to-local)
    - [1. Configure future PVCs to use `local`](#1-configure-future-pvcs-to-use-local)
    - [2. Record the existing PV configuration](#2-record-the-existing-pv-configuration)
    - [3. Stop the workload and retain the backing directory](#3-stop-the-workload-and-retain-the-backing-directory)
    - [4. Recreate the PV with a `local` volume source](#4-recreate-the-pv-with-a-local-volume-source)

## Prerequisites and architecture

The workflow has three parts:

1. Local Path Provisioner creates a local directory for each PVC.
2. Velero's node agent reads the mounted PVC data from each node.
3. Velero stores file-system backups (FSB) in an object-storage bucket.

Before you begin, make sure you have:

- A Kubernetes cluster with access to an S3-compatible object store. This example uses MinIO.
- `kubectl`, the Velero CLI, and Helm installed on your administration machine.
- Sufficient free space in the object-storage bucket for the workload data.
- Access to every Kubernetes node. The Velero node-agent DaemonSet must run on every node that hosts PVC data.

> **Important:** A local volume is tied to the node on which it was created. Velero protects the data, but restoring a workload to a different cluster or node still requires suitable storage and scheduling. Test the recovery process before relying on it for production workloads.

## Configure Local Path Provisioner for FSB

For new clusters, install Rancher's Local Path Provisioner and create a storage class that waits until a workload is scheduled before binding a volume.

```bash
kubectl apply -f \
	https://raw.githubusercontent.com/rancher/local-path-provisioner/master/deploy/local-path-storage.yaml

```

Local Path Provisioner's `volumeType` defaults to `hostPath`. Velero FSB does not support direct `hostPath` volumes, so configure the provisioner to create `local` volumes before creating PVCs.

```bash
kubectl patch storageclass local-path \
  --type=merge \
  -p '{"metadata":{"annotations":{"defaultVolumeType":"local"}}}'
```

Reference: [Local Path Provisioner volume types](https://github.com/rancher/local-path-provisioner#volume-types).

Confirm that the provisioner is ready:

```bash
kubectl get pods,storageclass -n local-path-storage
NAME                                          READY   STATUS    RESTARTS      AGE
pod/local-path-provisioner-79b7b99b5d-m6qkl   1/1     Running   2 (24h ago)   7d

NAME                                               PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
storageclass.storage.k8s.io/local-path (default)   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  7d
```

The `WaitForFirstConsumer` setting helps Kubernetes select a node before creating node-local storage. It prevents a PVC from being provisioned on a node where its pod cannot run.

## Deploy MinIO as Velero backup storage

Velero needs an S3-compatible backup destination. For a lab cluster, MinIO is a convenient option. In production, use a highly available object store with a retention policy appropriate for your recovery requirements.

Install MinIO with your preferred deployment method. This lab example creates the `velero` bucket and uses the Local Path Provisioner storage class. Replace the sample credentials before using this configuration outside a disposable test environment.

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

```bash
cat << 'EOF' > override.yaml
mode: standalone
persistence:
    size: 5Gi
    storageClass: "local-path"

resources:
    requests:
        memory:512Mi
auth:
    rootUser:admin
    rootPassword: admin123
provisioning:
    buckets:
    - name: velero
EOF
```
### Install MinIO with Helm

```bash
helm upgrade --install minio bitnami/minio --version 17.0.21 -n minio -f override.yaml
```
Wait until MinIO is running:

```bash
kubectl get pods,deploy,svc -n minio
NAME                                     READY   STATUS    RESTARTS      AGE
pod/minio-c95868c75-5hk2q   1/1     Running   2 (24h ago)   6d23h

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/minio   1/1     1            1           6d23h

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/minio           ClusterIP   10.43.161.125   <none>        9000/TCP   6d23h
service/minio-console   ClusterIP   10.43.244.52    <none>        9001/TCP   6d23h
```


## Install Velero with file-system backup

Install the Velero AWS plugin for the S3-compatible API and enable the node agent. The node agent is the component that captures Local Path Provisioner data.

```bash
helm repo add vmware-tanzu https://vmware-tanzu.github.io/helm-charts/
helm repo update
```
```bash
cat << 'EOF' > velero-override.yaml
configuration:
  volumeSnapshotLocation:
    - config:
        region: minio
      name: default
      provider: aws
  backupStorageLocation:
    - bucket: velero
      config:
        region: minio
        s3ForcePathStyle: 'true'
        s3Url: http://ninio.minio.svc.cluster.local:9000
      name: default
      provider: aws
credentials:
  secretContents:
    cloud: |
      [default]
      aws_access_key_id=admin
      aws_secret_access_key=admin123
deployNodeAgent: true

initContainers:
  - image: velero/velero-plugin-for-aws:v1.14.2
    name: velero-plugin-for-aws
    volumeMounts:
      - mountPath: /target
        name: plugins
upgradeCRDs: true
EOF
```
### Deploy Velero with the node agent

```bash
helm install velero vmware-tanzu/velero -n velero -f velero-override.yaml
```

Validate the installation:

```bash
kubectl get deploy,daemonset -n velero
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/velero   1/1     1            1           18h

NAME                        DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/node-agent   1         1         1       1            1           <none>          18h
```
```bash
kubectl get backuprepositories.velero.io,backupstoragelocations.velero.io,volumesnapshotlocations.velero.io -A
NAMESPACE   NAME                                      PHASE       LAST VALIDATED   AGE   DEFAULT
velero      backupstoragelocation.velero.io/default   Available   3s               18h   true

NAMESPACE   NAME                                       AGE
velero      volumesnapshotlocation.velero.io/default   18h
```

Wait until the backup location is `Available`. Velero creates the namespace-specific backup repository during the first FSB backup, so it may not appear yet. If the backup location is unavailable, inspect the controller logs before continuing:

```bash
kubectl logs deployment/velero -n velero
```

## Create a Local Path PVC test workload

Create a small workload with a PVC backed by the `local-path` storage class. The example below writes the current time to the volume every five seconds.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: writer-data
  namespace: demo
  annotations:
    volumeType: local
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: writer
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: writer
  template:
    metadata:
      labels:
        app: writer
    spec:
      containers:
        - name: writer
          image: busybox:1.36
          command:
            - /bin/sh
            - -c
            - while true; do date >> /data/events.log; sleep 5; done
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: writer-data
```

Apply it and verify that data exists:

```bash
kubectl apply -f writer.yaml
kubectl exec -n demo deployment/writer -- tail -n 5 /data/events.log
```
## Create a Velero FSB backup

Create a backup for the demo namespace:

```bash
apiVersion: velero.io/v1
kind: Backup
metadata:
  name: demo-backup
  namespace: velero
spec:
  includedNamespaces:
  - demo
  includeClusterResources: true   # <--- Must be true to capture PV objects
  defaultVolumesToFsBackup: true
  ttl: 720h0m0s
```

Apply the backup resource, then inspect its detailed results:

```bash
kubectl describe backups.velero.io demo-backup -n velero
```
Confirm that the backup phase is `Completed`:

```bash
Status:
  Completion Timestamp:  2026-08-25T04:04:07Z
  Expiration:            2026-09-24T04:04:00Z
  Format Version:        1.1.0
  Hook Status:
  Phase:  Completed
```


Look for a completed pod-volume backup in the detailed output. A successful object backup alone is not enough; confirm that Velero created an FSB backup for the `data` volume that mounts the Local Path Provisioner PVC.

Velero's FSB controller creates a `PodVolumeBackup` for each protected pod volume. This is the most direct confirmation that Velero copied the data rather than only saving Kubernetes resources:

```bash
kubectl get PodVolumeBackups -A | grep demo
velero      demo-backup-8ghdn   Completed   2m40s     551          551           default            2m40s          kopia
```
In the MinIO bucket, confirm that Velero created the `backups` and `kopia` prefixes. The `kopia` repository is created when the first FSB backup runs.

## Restore a Local Path PVC from Velero FSB

Delete the test namespace, then restore it from the backup:

```bash
kubectl delete namespace demo

```
### Create the restore resource

```yml
apiVersion: velero.io/v1
kind: Restore
metadata:
  name: demo-backup-restore
  namespace: velero
spec:
  backupName: demo-backup

  includedNamespaces:
  - demo
  # namespaceMapping:
  #   postgresql: postgresql-restore

  restorePVs: true
  existingResourcePolicy: none
```

```bash
kubectl get restores -A | grep demo
velero      demo-backup-restore   23s
```
```bash
kubectl describe restores demo-backup-restore -n velero
```
Confirm that the restore phase is `Completed`:
```bash
Status:
  Completion Timestamp:  2026-08-25T04:11:20Z
  Hook Status:
  Phase:  Completed
  Progress:
    Items Restored:  8
    Total Items:     8
```

```bash
kubectl get deploy,pod,pvc -n demo
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/writer   1/1     1            1           100s

NAME                         READY   STATUS    RESTARTS   AGE
pod/writer-8dbf84dbb-xcls2   1/1     Running   0          100s

NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/writer-data   Bound    pvc-02be3cbe-4926-4865-8a44-aeab1d53cc1c   1Gi        RWO            local-path     <unset>                 100s
```
```bash
kubectl get podvolumerestores -A | grep demo
velero      demo-backup-restore-g6jgs   Completed   2m56s     551          551           default            3m2s   cnpg-01   kopia
```

After the restored pod becomes ready, verify that the Local Path PVC data was restored:

```bash
kubectl exec -n demo deployment/writer -- tail -n 5 /data/events.log
```

The restored log should include entries created before the backup. For a real stateful application, also verify application-level consistency, not just the presence of files.

Confirm that all file-system restores completed successfully.

## Migrate existing `hostPath` Local Path PVCs to `local`

Velero FSB supports Kubernetes `local` PersistentVolumes, not direct `hostPath` pod volumes. For an existing Local Path Provisioner PVC created with `volumeType: hostPath`, changing the PVC annotation does not convert the existing PV. The PV volume source is immutable.

The migration is therefore a controlled replacement of the PV object while preserving the directory that contains the data. Perform this during a maintenance window, with the workload stopped and a verified application backup or filesystem copy available.

> **Warning:** Do not run this procedure on a running workload. Deleting the original Local Path Provisioner PV can trigger provisioner cleanup of its backing directory. Confirm the directory path and retain a verified copy before proceeding.

### 1. Configure future PVCs to use `local`

This affects only PVCs created after the change:

```bash
kubectl patch storageclass local-path \
  --type=merge \
  -p '{"metadata":{"annotations":{"defaultVolumeType":"local"}}}'
```

### 2. Record the existing PV configuration

Set values for the workload you are migrating, then save the original resources. The PV manifest identifies the current backing directory and the node where it resides.

```bash
export NAMESPACE=postgresql
export PVC_NAME=test
export PV_NAME=$(kubectl get pvc -n "${NAMESPACE}" "${PVC_NAME}" \
  -o jsonpath='{.spec.volumeName}')

kubectl get pvc -n "${NAMESPACE}" "${PVC_NAME}" -o yaml > pvc-original.yaml
kubectl get pv "${PV_NAME}" -o yaml > pv-original.yaml
kubectl get pv "${PV_NAME}" \
  -o jsonpath='path={.spec.hostPath.path}{"\n"}node={.metadata.annotations.local\.path\.provisioner/selected-node}{"\n"}'
```

Copy the data from the reported path to independent storage before continuing. This copy is the rollback point for the migration.

### 3. Stop the workload and retain the backing directory

Scale down every workload that mounts the PVC. Replace the deployment command with the controller type used by your application.

```bash
kubectl scale deployment/<workload-name> -n "${NAMESPACE}" --replicas=0
kubectl wait --for=delete pod \
  -n "${NAMESPACE}" -l app=<workload-label> --timeout=5m

kubectl patch pv "${PV_NAME}" \
  --type=merge \
  -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'
```

### 4. Recreate the PV with a `local` volume source

Deleting the PVC releases the claim. Delete the old PV only after you have confirmed that its directory is preserved. Local Path Provisioner may use a finalizer to clean up dynamically provisioned storage; remove that finalizer only after taking the independent data copy described above.

```bash

kubectl patch pv "${PV_NAME}" \
  --type=json \
  -p='[{"op":"remove","path":"/metadata/finalizers"}]'
kubectl delete pv "${PV_NAME}" --wait=true
```

Create a new static PV. Substitute the path, node name, capacity, access modes, and storage class from `pv-original.yaml`. Use a new PV name so it is clear that this is the replacement object.

```yaml
cat <<EOF > pv-new.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  annotations:
    local.path.provisioner/selected-node: {{NODE}}
    pv.kubernetes.io/provisioned-by: rancher.io/local-path
  finalizers:
    - kubernetes.io/pv-protection
  name: pvc-ce442382-e3b8-4c33-beaa-e2c9ed044843
spec:
  accessModes:
    - ReadWriteOnce
  capacity:
    storage: 1Gi
  claimRef:
    apiVersion: v1
    kind: PersistentVolumeClaim
    name: test
    namespace: postgresql
    uuid: {{UUID}}
  local:
    path: >-
      /opt/local-path-provisioner/{{PVC_NAME}}_{{NAMESPACE}}_{{NAME}}
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - {{ NODE_NAME }}
  persistentVolumeReclaimPolicy: Delete
  storageClassName: local-path
  volumeMode: Filesystem
EOF
```

Apply the replacement PV:

```bash
kubectl apply -f pv-new.yaml
```
```bash
kubectl get pv local-pv-postgresql-test
kubectl get pvc -n "${NAMESPACE}" "${PVC_NAME}"
```

After the application is running, verify its data and run the Velero FSB backup and restore test described above. Keep the independent copy until the restore test succeeds.


## Wrapping Up

In this guide, we walked through the process of backing up Local Path Provisioner PVCs using Velero's File-System Backup (FSB) capabilities. Because Local Path Provisioner is bound to specific nodes, conventional volume snapshot mechanisms often fail to capture the underlying data. By switching the default volume type to `local` and deploying the Velero node-agent daemonset, we enabled FSB to successfully extract and secure data from node-local storage. We also established a safe path for migrating older `hostPath`-based PVCs to the supported `local` configuration.

### Next Steps
1. **Establish Backup Schedules**: Automate your backups by creating a Velero [`Schedule`](https://velero.io/docs/v1.12/scheduling-backups/) resource to run backups at regular intervals (e.g., daily or weekly).
2. **Set Up Retention Policies**: Configure TTL (Time to Live) on your Velero backups and lifecycle policies on your S3/MinIO buckets to prune outdated backups and control storage costs.
3. **Monitor Backup Status**: Integrate Velero metrics with Prometheus and Alertmanager to receive notifications if backups fail or backup repository connections degrade.
4. **Conduct Regular DR Drills**: Periodically perform dry-run restores in a separate development or staging namespace to guarantee that your backup configuration remains valid as your cluster configuration changes.

## Reference Links

- [Velero Installation Guide](https://velero.io/docs/v1.12/basic-install/)
- [Velero File System Backup (FSB) Documentation](https://velero.io/docs/v1.12/file-system-backup/)


Happy Kubernetes-ing! 🚀