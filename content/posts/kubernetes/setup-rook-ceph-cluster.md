---
date: '2026-11-01T12:26:26+08:00'
draft: false
title: "How to Set Up a Rook Ceph Storage Cluster on Kubernetes: A Step-by-Step Guide"
description: "Step-by-step tutorial on deploying a production-grade Rook-Ceph distributed storage cluster on Kubernetes with Helm. Learn raw disk preparation, CephCluster CRD setup, and PVC verification."
tags: ["kubernetes", "rook", "ceph", "storage", "csi", "rke2", "cloud-native", "longhorn", "devops"]
categories: ["Kubernetes", "Storage"]
author: K8S team
canonicalURL: "https://blog.avidhara.cloud/posts/kubernetes/setup-rook-ceph-cluster/"
---


## Why We Started Looking at Rook-Ceph

Storage is usually the part of a Kubernetes setup that nobody wants to own. Stateless apps are easy: scale them up, kill pods, deploy canary versions, and nothing breaks. But the moment you introduce a database, a message broker, or any stateful workload that writes to persistent disks, you need volumes that seamlessly survive node restarts and hardware failures. That is where Kubernetes storage gets interesting.

We had been running [Longhorn](https://longhorn.io/) on smaller clusters, and it did the job well. However, as our infrastructure matured and we needed shared filesystems (`ReadWriteMany`) and S3-compatible object buckets alongside replicated block volumes, we constantly ended up deploying and maintaining three separate storage engines. That friction prompted us to evaluate **Rook-Ceph**.

In this guide, we walk through our architecture decisions, compare Rook-Ceph against Longhorn, prepare raw storage devices on Kubernetes nodes (RKE2), deploy Rook-Ceph with Helm, and validate block storage with an end-to-end PersistentVolumeClaim (PVC).

---

## What is Rook? Architecture & Core Daemons

[Rook](https://rook.io/) is an open-source, cloud-native storage orchestrator for Kubernetes and a graduated CNCF project. It turns Ceph into a self-healing Kubernetes service.

Rook itself does not store user data. Instead, it acts as the Kubernetes operator that automates the deployment, provisioning, configuration, scaling, and upgrade lifecycles of Ceph daemons inside your cluster.

[Ceph](https://ceph.io/) is the underlying distributed software-defined storage engine. Battle-tested for over a decade in hyperscale environments, Ceph unifies three distinct storage types from a single physical pool:

1. **Block Storage (RBD):** Low-latency block devices mounted by a single pod (`ReadWriteOnce` / RWO PVCs) for databases, queues, and persistent application state.
2. **Shared Filesystem (CephFS):** POSIX-compliant shared filesystem mounted concurrently by multiple pods (`ReadWriteMany` / RWX PVCs) for file uploads, CMS platforms, or shared datasets.
3. **Object Storage (RGW):** S3 and Swift-compatible object store gateway for cloud-native backups, logs, ML models, and media artifacts.

![Rook Ceph Architecture](/images/rook-architecture.png)

```
       +-------------------------------------------------------+
       |                  Kubernetes Workloads                 |
       +-------------------------------------------------------+
           | (RWO Block)           | (RWX File)          | (S3 API)
           v                       v                     v
     +------------+         +---------------+      +-------------+
     |  Ceph RBD  |         |    CephFS     |      |  Ceph RGW   |
     | CSI Plugin |         |  CSI Plugin   |      |  (S3 REST)  |
     +------------+         +---------------+      +-------------+
           |                       |                     |
           +-----------------------+---------------------+
                                   |
                   +-------------------------------+
                   |    Rook Ceph Operator (CRDs)  |
                   +-------------------------------+
                                   |
        +--------------------------+--------------------------+
        |                          |                          |
   +---------+                +---------+                +---------+
   |  MONs   |                |  MGRs   |                |  OSDs   |
   | (Quorum)|                |(Metrics)|                | (Disks) |
   +---------+                +---------+                +---------+
```

### Core Rook-Ceph Pods in the Cluster

Once deployed, the `rook-ceph` namespace runs several specialized daemons:

- **`rook-ceph-operator`**: The operator controller that watches Rook Custom Resource Definitions (CRDs) and reconciles Ceph cluster state.
- **`rook-ceph-mon`**: Ceph Monitors maintaining the master cluster map, consensus, and state quorum. A production cluster typically runs 3 or 5 monitors across different nodes.
- **`rook-ceph-mgr`**: Ceph Manager daemons providing monitoring metrics, the built-in management dashboard, and external Prometheus exporters.
- **`rook-ceph-osd`**: Object Storage Daemons. Each physical raw drive or partition assigned to Ceph runs an OSD pod. OSDs handle data reads, replication, erasure coding, rebalancing, and disk writes.
- **`rook-ceph-mds`**: Metadata Servers required whenever CephFS shared filesystems are enabled.
- **`rook-ceph-rgw`**: RADOS Gateway HTTP daemons presenting the S3-compatible REST API.
- **`csi-rbdplugin` & `csi-cephfsplugin`**: Ceph CSI driver controller and node plugins that interface Kubernetes volume requests with Ceph pools.

---

## Rook-Ceph vs. Longhorn: Which Should You Choose?

Both Rook-Ceph and Longhorn are widely used for persistent storage in Kubernetes environments, especially on Rancher and RKE2 clusters. Here is an architectural and operational comparison:

| Feature | Rook-Ceph | Longhorn |
| :--- | :--- | :--- |
| **Supported Storage Types** | Block (RBD), Shared File (CephFS), Object (S3 RGW) | Block storage only (RWX requires an extra NFS share-manager) |
| **Data Distribution** | CRUSH algorithm stripes data evenly across all disks | Controller/replica model per volume placed on specific nodes |
| **Replication Strategy** | Distributed object striping across OSDs | Full disk copy per replica on target nodes |
| **Data Redundancy** | Multi-way replication & Erasure Coding (e.g., 4+2) | Replicas only (higher storage overhead) |
| **Scalability** | Petabyte-scale, hundreds of nodes | Ideal for small to mid-sized clusters |
| **ReadWriteMany (RWX)** | Native kernel/CSI driver directly to OSDs | Simulated via NFS pod proxy (single point of contention) |
| **Hardware Requirements** | Dedicated raw disks, ≥ 4GB RAM per OSD, ≥ 10GbE network | Uses host filesystems, lightweight CPU & RAM footprint |
| **Failure Domains** | Granular CRUSH maps (node, rack, row, zone, DC) | Basic node and zone scheduling anti-affinity |
| **Operational Curve** | Moderate to High (requires understanding Ceph commands) | Low (very beginner-friendly web UI) |

### Key Architectural Takeaways

1. **Native RWX Performance:** Longhorn implements `ReadWriteMany` volumes by deploying an in-cluster NFS server pod in front of an RWO volume. While functional, that NFS pod becomes an I/O bottleneck and failure point. CephFS is inherently distributed; every client node talks directly to the Ceph OSDs.
2. **Rebuild Speed:** If a node fails in Longhorn, rebuilding a 1TB volume involves copying the entire 1TB replica sequentially across the network. Ceph slices data into discrete objects across all OSDs, allowing parallel recovery across dozens of disks simultaneously.
3. **Erasure Coding Savings:** For large-scale data storage, 3x replication consumes 300% raw disk space. Ceph supports erasure coding (e.g., $k=4, m=2$), achieving enterprise-grade durability with only 50% storage overhead.
4. **Hardware Commitment:** Ceph requires dedicated disks (not shared with host OS mounts), ample memory (3–4 GB per OSD recommended), and fast networks (10GbE or higher). For small test clusters or resource-constrained nodes, Longhorn is much easier to spin up.

---

## When to Choose Rook-Ceph

**Choose Rook-Ceph if:**

- You require unified Block, Shared File, and Object storage under one control plane.
- You are operating on bare-metal, on-premise, or hybrid clouds and want a consistent storage API without cloud vendor lock-in.
- Your workloads demand native high-throughput `ReadWriteMany` performance (CI/CD caches, AI/ML datasets, document stores).
- You manage petabyte-scale storage clusters and want to save disk capacity with erasure coding.
- Your infrastructure spans multiple failure domains (e.g., racks, power zones) and requires strict placement rules.

**Stick with Longhorn or simpler solutions if:**

- Your cluster has fewer than 3 nodes or lacks dedicated empty storage disks.

- You only need simple block volumes (`ReadWriteOnce`).
- Your team prefers an intuitive zero-overhead GUI without learning Ceph internals (`ceph status`, placement groups, CRUSH hierarchy).

---

## Prerequisites and Infrastructure Planning

Before applying any Kubernetes manifests, verify that your nodes satisfy the operational requirements.

### 1. Kubernetes & Compute Requirements

- Kubernetes **v1.32+** (tested here on RKE2 **v1.36.5**).

- Architecture: **amd64 (x86_64)** or **arm64**.
- Cluster sizing: Minimum of **3 worker/storage nodes** to satisfy default Ceph monitor quorum (3 MONs) and standard 3-replica pools.

Check your cluster nodes:

```bash
kubectl get nodes -o wide
```

```bash
kubectl get nodes -o wide
NAME           STATUS   ROLES                AGE    VERSION          INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION              CONTAINER-RUNTIME
rook-ceph-01   Ready    control-plane,etcd   6d1h   v1.36.5+rke2r1   172.21.0.9    <none>        Ubuntu 24.04.4 LTS   6.8.0-134-generic (amd64)   containerd://2.3.4-k3s1
```

### 2. Disk Preparation: Wiping and Resetting Storage Drives

Ceph requires clean, unformatted raw devices or partitions with no existing filesystem signatures, LVM headers, or GPT partition tables.

> [!WARNING]
> Running the following disk wipe commands destroys all existing data on the target drive. Ensure you verify the device identifier (e.g., `/dev/vdb`) before executing!

Check the existing storage layout using `lsblk`:

```bash
lsblk
```

```output
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda     253:0    0   20G  0 disk
├─vda1  253:1    0   19G  0 part /
├─vda14 253:14   0    4M  0 part
├─vda15 253:15   0  106M  0 part /boot/efi
└─vda16 259:0    0  913M  0 part /boot
vdb     253:16   0   50G  0 disk
vdc     253:32   0    1M  0 disk
```

In our environment, `/dev/vda` is the OS disk, `/dev/vdc` is a cloud-init config drive, and `/dev/vdb` (50 GB) is our dedicated Ceph target disk on all three nodes.

To completely reset `/dev/vdb` and wipe lingering partition tables, GPT signatures, and filesystem headers, run these commands on each storage node:

```bash
# Discard sectors and wipe beginning/end blocks
sudo blkdiscard -f /dev/vdb 2>/dev/null || true
sudo dd if=/dev/zero of=/dev/vdb bs=1M count=200 status=progress oflag=direct,dsync
sudo dd if=/dev/zero of=/dev/vdb bs=1M count=1000 seek=$(($(blockdev --getsz /dev/vdb)/2048 - 1000)) status=progress oflag=direct,dsync

# Wipe filesystem magic signatures and zap partition tables
sudo wipefs -a -f /dev/vdb
sudo sgdisk --zap-all /dev/vdb
sudo partprobe /dev/vdb
```

---

## Step-by-Step Installation Using Helm

We deploy Rook-Ceph using official Helm charts. Rook structures deployment across two charts:

1. `rook-ceph`: Installs the operator, CRDs, admission controllers, and CSI drivers.
2. `rook-ceph-cluster`: Deploys the actual `CephCluster` resource, storage pools, and StorageClasses.

### Step 1: Deploy the Rook-Ceph Operator

Add the official Rook repository and update your Helm cache:

```bash
helm repo add rook-release https://charts.rook.io/release
helm repo update
```

Install the `rook-ceph` operator in the `rook-ceph` namespace:

```bash
helm upgrade --install --create-namespace \
  --namespace rook-ceph \
  rook-ceph rook-release/rook-ceph
```

Verify that the operator pod is running:

```bash
kubectl get pods -n rook-ceph
```

```bash
NAME                                           READY   STATUS    RESTARTS   AGE
ceph-csi-controller-manager-6dc6ff9685-llzs7   1/1     Running   0          46s
rook-ceph-operator-797f87d897-srp5x            1/1     Running   0          46s
```

### Step 2: Install the Ceph CSI Drivers

To allow Kubernetes to mount Ceph volumes via CSI plugins, install the `ceph-csi-drivers` chart:

```bash
helm repo add ceph-csi-operator https://ceph.github.io/ceph-csi-operator
helm repo update

helm install ceph-csi-drivers --namespace rook-ceph ceph-csi-operator/ceph-csi-drivers \
  -f https://raw.githubusercontent.com/rook/rook/v1.21.1/deploy/charts/ceph-csi-drivers/values.yaml
```

Verify the CSI controller and node plugin daemonsets:

```bash
kubectl get pods -n rook-ceph -l 'app in (ceph-csi-controller-manager, rook-ceph-operator)'
kubectl get daemonset -n rook-ceph
```

You should see active pods for `rook-ceph.cephfs.csi.ceph.com-nodeplugin` and `rook-ceph.rbd.csi.ceph.com-nodeplugin` running on every storage node.

---

### Step 3: Provision the Ceph Cluster (`overrides.yaml`)

By default, the `rook-ceph-cluster` chart attempts to claim all available storage devices on all nodes (`useAllDevices: true`). To prevent accidental claims of host partitions or cloud-init config drives (`vdc`), we create a customized `overrides.yaml` explicitly targeting `/dev/vdb`.

Create `overrides.yaml`:

```yaml
# overrides.yaml - Custom configuration for Rook-Ceph cluster
operatorNamespace: rook-ceph

cephClusterSpec:
  cephConfig:
    global:
      osd_pool_default_size: "1"
      osd_pool_default_min_size: "1"
      mon_warn_on_pool_no_redundancy: "false"
      mon_max_pg_per_osd: "500"
  mon:
    count: 1               # In production with 3+ nodes, set count to 3
  mgr:
    count: 1
    allowMultiplePerNode: false
  dashboard:
    enabled: true
    ssl: false
  storage:
    useAllNodes: true
    useAllDevices: false
    deviceFilter: "^vdb$"  # Strictly bind to vdb disk only
  resources:
    mgr:
      limits:
        memory: 1Gi
      requests:
        cpu: 200m
        memory: 512Mi
    mon:
      limits:
        memory: 2Gi
      requests:
        cpu: 250m
        memory: 1Gi
    osd:
      limits:
        memory: 4Gi
      requests:
        cpu: 250m
        memory: 4Gi

cephBlockPools:
  - name: ceph-blockpool
    spec:
      failureDomain: osd
      replicated:
        size: 1            # Set to 3 for production multi-node replica safety
        requireSafeReplicaSize: false

cephFileSystems:
  - name: ceph-filesystem
    spec:
      metadataPool:
        failureDomain: osd
        replicated:
          size: 1
          requireSafeReplicaSize: false
      dataPools:
        - failureDomain: osd
          name: data0
          replicated:
            size: 1
            requireSafeReplicaSize: false
      metadataServer:
        activeCount: 1
        activeStandby: true
        priorityClassName: system-cluster-critical
        resources:
          limits:
            memory: 4Gi
          requests:
            cpu: 250m
            memory: 1Gi

cephObjectStores:
  - name: ceph-objectstore
    spec:
      metadataPool:
        failureDomain: osd
        replicated:
          size: 1
          requireSafeReplicaSize: false
      dataPool:
        failureDomain: osd
        replicated:
          size: 1
          requireSafeReplicaSize: false
        parameters:
          bulk: "true"
      preservePoolsOnDelete: true
      gateway:
        port: 80
        instances: 1
        priorityClassName: system-cluster-critical
        resources:
          limits:
            memory: 2Gi
          requests:
            cpu: 250m
            memory: 1Gi
    ingress:
      enabled: true
      host:
        name: rook-ceph-s3.internal.lab
        path: /
        pathType: Prefix
      ingressClassName: traefik

ingress:
  dashboard:
    host:
      name: rook-ceph.internal.lab
      path: /
      pathType: Prefix
    ingressClassName: traefik
```

Deploy the Ceph cluster chart using our custom overrides:

```bash
helm install --create-namespace --namespace rook-ceph rook-ceph-cluster \
   --set operatorNamespace=rook-ceph rook-release/rook-ceph-cluster -f overrides.yaml
```

Monitor the cluster initialization:

```bash
kubectl get cephclusters -A -w
```

The operator spins up components in a deterministic sequence:

1. **Monitors (`mon`)** reach quorum.
2. **Managers (`mgr`)** initialize metrics and dashboard services.
3. **`osd-prepare` pods** inspect `/dev/vdb` on each node, format with BlueStore, and create persistent metadata.
4. **`osd` pods** launch and join the storage pool.
5. **MDS and RGW daemons** start up for file and object protocols.

When deployment is complete, verify all pods:

```bash
kubectl get pods -n rook-ceph
```

```output
NAME                                                        READY   STATUS      RESTARTS   AGE
ceph-csi-controller-manager-6dc6ff9685-llzs7                1/1     Running     0          19m
rook-ceph-crashcollector-rook-ceph-02-66f7c8c47d-6kwp6      1/1     Running     0          68s
rook-ceph-exporter-rook-ceph-02-d95c64685-fv8cb             2/2     Running     0          64s
rook-ceph-mds-ceph-filesystem-a-655d58c9c4-jlwvl            2/2     Running     0          70s
rook-ceph-mds-ceph-filesystem-b-7d6bc5559-rl28t             2/2     Running     0          64s
rook-ceph-mgr-a-5df7569f9-qw5j9                             2/2     Running     0          3m39s
rook-ceph-mon-a-d9fdbc974-4nzlg                             2/2     Running     0          4m13s
rook-ceph-operator-7647d9896-5jcdx                          1/1     Running     0          4m26s
rook-ceph-osd-0-7f9cbff7c5-89ksf                            2/2     Running     0          2m46s
rook-ceph-osd-prepare-rook-ceph-02-t8sx9                    0/1     Completed   0          3m17s
rook-ceph-tools-748887bfd7-l5724                            1/1     Running     0          7m53s
rook-ceph.cephfs.csi.ceph.com-ctrlplugin-6d7c986d9d-8jdvz   6/6     Running     0          17m
rook-ceph.cephfs.csi.ceph.com-nodeplugin-997lx              3/3     Running     0          17m
rook-ceph.rbd.csi.ceph.com-ctrlplugin-79ddcd4948-4rf67      5/5     Running     0          17m
rook-ceph.rbd.csi.ceph.com-nodeplugin-sw4t5                 3/3     Running     0          17m
```

> [!TIP]
> Always check for `Completed` status on `rook-ceph-osd-prepare-*` jobs. If prepare jobs fail or no OSD pods start, inspect the prepare logs:
>
> ```bash
> kubectl -n rook-ceph logs -l app=rook-ceph-osd-prepare --tail=100
> ```

---

## Step 4: Validating Ceph Health and Status

To inspect cluster health using native `ceph` CLI commands, run commands through the `rook-ceph-tools` pod:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
```

```output
  cluster:
    id:     c19b585d-a300-4c19-88a2-77a6929063f8
    health: HEALTH_OK

  services:
    mon: 1 daemons, quorum a (age 5d) [leader: a]
    mgr: a(active, since 4d)
    mds: 1/1 daemons up, 1 standby
    osd: 1 osds: 1 up, 1 in
    rgw: 2 daemons active

  data:
    volumes: 1/1 healthy
    pools:   22 pools, 441 pgs
    objects: 1.07k objects, 686 KiB
    usage:   1.3 GiB used, 74 GiB / 75 GiB avail
    pgs:     441 active+clean

  io:
    client:   20 KiB/s rd, 39 op/s rd, 0 op/s wr
```

Inspect the physical disk hierarchy and pool allocations:

```bash
# Check OSD placement tree
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree

# View disk capacity breakdown
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph df
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph health detail
```

> [!NOTE]
> If you observe `HEALTH_WARN` with warnings like `AUTH_INSECURE_CLIENT_KEY_TYPE`, this typically relates to Ceph's transition to msgr2 encryption ciphers. In test/lab environments, you can silence or remediate this via:
>
> ```bash
> kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph config set global mon_warn_on_insecure_global_id_reclaim false
> ```

---

## Step 5: Validating Persistent Storage with a Test PVC

To verify that the Ceph CSI driver dynamically provisions block storage volumes, deploy a sample `PersistentVolumeClaim` and test pod:

```yaml
# test-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-test
spec:
  accessModes: ["ReadWriteOnce"]
  storageClassName: ceph-block
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: ceph-test
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo 'Hello from Rook-Ceph Persistent Storage' > /data/hello.txt && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: ceph-test
```

Apply the manifest and inspect the PVC binding:

```bash
kubectl apply -f test-pvc.yaml
kubectl get pvc ceph-test
```

Verify that the application successfully wrote data to the mounted Ceph RBD image:

```bash
kubectl exec ceph-test -- cat /data/hello.txt
```

```output
Hello from Rook-Ceph Persistent Storage
```

Clean up the test resources:

```bash
kubectl delete -f test-pvc.yaml
```

---

## Step 6: Accessing the Ceph Management Dashboard

Rook deploys the Ceph Dashboard for real-time monitoring of cluster IOPS, pool capacity, OSD latency, and PG states.

Retrieve the generated dashboard administrator password:

```bash
kubectl -n rook-ceph get secret rook-ceph-dashboard-password \
  -o jsonpath="{['data']['password']}" | base64 -d && echo
```

Navigate to `https://rook-ceph.internal.lab`, log in as username `admin` with the retrieved password, and view your cluster topology.

---

## Troubleshooting Common Rook-Ceph Issues

| Symptom | Root Cause | Solution |
| :--- | :--- | :--- |
| **OSD pods never start** | Target disk contains previous filesystem signatures or LVM headers. | Run `blkdiscard`, `wipefs -a -f /dev/sdX`, and `sgdisk --zap-all /dev/sdX`. Restart the operator. |
| **Monitors stuck out of quorum** | Inter-node latency is high or firewall drops port `6789`/`3300`. | Ensure open TCP traffic on ports `6789`, `3300`, and `6800-7300`. Verify node clocks are synced via NTP. |
| **PVC remains in `Pending` state** | Missing CSI driver or `deviceFilter` regex failed to match disk. | Verify `kubectl get sc` matches `ceph-block` and check CSI provisioner logs with `kubectl logs -n rook-ceph -l app=csi-rbdplugin`. |
| **CrashLoopBackOff on `rook-ceph-mgr`** | Resource limits too constrained or multiple managers conflicting. | Increase memory requests/limits in `overrides.yaml` (minimum 1GB RAM for mgr). |

---

## What We Covered

In this guide, we built a complete cloud-native distributed storage platform on Kubernetes with Rook-Ceph in a single node config mode.:

- **Storage Architecture Decoded:** Explored how the Rook operator automates Ceph daemons (MONs, MGRs, OSDs, MDS, RGW, and CSI plugins).
- **Rook-Ceph vs. Longhorn:** Analyzed the technical trade-offs between distributed object-based striping and volume-level replication.
- **Disk Sanitation:** Cleaned and wiped raw block storage devices using `blkdiscard`, `dd`, `wipefs`, and `sgdisk` to prepare them for Ceph BlueStore.
- **Declarative Helm Deployment:** Deployed the `rook-ceph` operator and `ceph-csi-drivers`, configuring a production-aligned `overrides.yaml` with explicit disk filters.
- **Health Validation & Tooling:** Leveraged `rook-ceph-tools` to run `ceph status`, `ceph osd tree`, and `ceph df` to audit cluster health.
- **End-to-End PVC Testing:** Provisioned an automated RBD PersistentVolumeClaim and verified data persistence inside a test pod.
- **Dashboard Access:** Retrieved credentials and accessed the Ceph administrative dashboard.

---

## Conclusion & Next Steps

Deploying Rook-Ceph provides your Kubernetes clusters with enterprise-grade, software-defined distributed storage that rivals commercial SAN/NAS appliances and hyperscale cloud providers. By eliminating single points of failure and avoiding separate engines for files, blocks, and object buckets, Rook-Ceph provides a unified, self-healing foundation for all stateful workloads.

**Next in this Series:**

- Setting up **CephFS** shared filesystems for high-throughput `ReadWriteMany` (RWX) workloads.
- Configuring **Multi-Site Replication** for Rook-Ceph S3 (RGW) across independent clusters for disaster recovery and cross-region sync.

---

**Join the Discussion:**
Have questions about sizing your OSDs or choosing between Ceph and Longhorn? Drop your comments below or connect with us on [GitHub](https://github.com/avidhara) and [LinkedIn](https://www.linkedin.com/company/avidhara).

Happy Kubernetes-ing! 🚀
