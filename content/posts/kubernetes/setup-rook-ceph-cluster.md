---

date: '2026-10-16T12:26:26+08:00'
draft: true
title: "Setup Rook Ceph cluster"
tags: [ "rke2",  "kubernetes", "k8s", "storage", "rook", "ceph", "longhorn", "csi"]
author: K8S team
---

## Why we started looking at Rook

Storage is usually the part of a Kubernetes setup that nobody wants to own. Stateless apps are easy, you scale them up, kill pods, nothing breaks. The moment you bring in a database, a message queue or anything that writes to disk, you need persistent volumes that survive node failures, and that's where things get interesting.

We had been running Longhorn on a few smaller clusters and it did the job. But as the clusters grew and we started needing shared filesystems and S3-compatible buckets alongside block volumes, we kept ending up with three different tools for three different storage types. That's what pushed us to take a proper look at Rook with Ceph.

These are my notes from that exercise. The actual setup steps come later in this post.

## So what is Rook?

[Rook](https://rook.io/docs/rook/latest-release/Getting-Started/intro/) is an open source cloud-native storage orchestrator for Kubernetes. On its own Rook doesn't store anything — it's the operator that deploys, configures and looks after a Ceph cluster running inside your Kubernetes cluster.

Ceph is the actual storage system underneath. It's been around for well over a decade, it's battle tested in some very large deployments, and it gives you three kinds of storage from one cluster:

- **Block storage** (RBD) – a volume mounted by a single pod, the typical `ReadWriteOnce` PVC.
- **Shared filesystem** (CephFS) – a POSIX filesystem that many pods can mount at the same time (`ReadWriteMany`).
- **Object storage** (RGW) – an S3 compatible API, handy for backups, artifacts, logs and so on.

Historically the pain with Ceph was running it. It has a lot of moving parts (MONs, MGRs, OSDs, MDS, RGW) and getting them all healthy by hand took real effort. Rook's job is to take that pain away. You describe what you want in a `CephCluster` custom resource, and the Rook operator handles the bootstrapping, wiring up the CSI driver, watching the daemons, and doing upgrades. Rook is a graduated CNCF project, so it's not going anywhere soon.

### The pieces you'll see in the cluster

When Rook is running you'll typically notice these pods in the `rook-ceph` namespace:

- **rook-ceph-operator** – the brain. Watches the CRDs and reconciles the Ceph cluster.
- **mon** – Ceph monitors, keep the cluster map and quorum. Usually 3.
- **mgr** – manager daemons, metrics and the dashboard.
- **osd** – one per disk. This is where your data actually lives.
- **mds** – only if you're using CephFS.
- **rgw** – only if you're using object storage.
- **csi-rbdplugin / csi-cephfsplugin** – the Ceph CSI drivers that Kubernetes talks to when you create a PVC.

## Rook-Ceph vs Longhorn

This is the comparison most people actually care about, because both show up in the same conversations, especially if you're on Rancher/RKE2 where Longhorn is basically one click away.

Longhorn is great, and I don't want to make it sound otherwise. It's simple, the UI is nice, and you can have it running in about five minutes. But it was built with a narrower goal in mind: replicated block storage for Kubernetes. Ceph was built as a general purpose distributed storage system, and that difference shows up everywhere.

| | Rook-Ceph | Longhorn |
| --- | --- | --- |
| Storage types | Block, shared filesystem, object (S3) | Block only (RWX via an NFS share-manager on top) |
| How data is stored | Data spread across OSDs using CRUSH, striped across many disks | Each volume has its own engine with full replicas on a few nodes |
| Scale | Comfortable at hundreds of nodes and petabytes | Best on small to medium clusters |
| Performance | Strong, especially on raw disks and fast networks | Good for most workloads, engine per volume adds overhead |
| Erasure coding | Yes | No, replication only |
| Failure domains | Very flexible (host, rack, zone, region) through CRUSH rules | Basic node/zone anti-affinity |
| Operational complexity | Higher, you need to understand some Ceph | Low, very beginner friendly |
| Resource usage | Heavier, MONs and OSDs want real CPU and RAM | Lighter footprint |
| External cluster | Can connect to an existing Ceph cluster outside k8s | Runs only inside the cluster |
| Disk requirements | Wants dedicated raw disks/partitions | Can use a directory on an existing filesystem |

A few things from that table are worth calling out.

**RWX volumes.** With Longhorn, `ReadWriteMany` is done by spinning up an NFS server pod in front of a block volume. It works, but that NFS pod becomes a bottleneck and a single point you need to think about. CephFS is natively shared, every client talks directly to the OSDs.

**How replication works.** In Longhorn every volume is its own little world with a few full copies sitting on a few nodes. Rebuilding a large volume after a node dies means copying the whole thing over. Ceph chops data into objects and spreads them across all the OSDs, so a recovery is done by many disks in parallel and finishes a lot faster on a decent sized cluster.

**Erasure coding.** If you're storing a lot of data, keeping 3 full copies gets expensive. Ceph lets you use erasure coded pools (say 4+2) which gives you similar durability for a lot less raw disk. Longhorn doesn't have this.

**The flip side.** Ceph is not small. You need dedicated disks, a reasonable amount of memory per OSD (plan for around 4GB), and ideally a fast network, 10GbE or better. And when something does go wrong you will end up running `ceph status` and reading about placement groups. Longhorn is much more forgiving if you just want things to work on three small nodes.

## What about other CSI options?

Quick notes on the others we looked at:

- **Cloud provider CSI (EBS, Azure Disk, GCE PD)** – easiest choice if you're in a single cloud. But it ties you to that cloud, volumes are usually zone bound, and it's no help on bare metal or at the edge.
- **local-path / TopoLVM / OpenEBS LocalPV** – fast because the data stays on the node, but there's no replication. Lose the node, lose the data, unless the app replicates itself (like Postgres with streaming replication or Kafka).
- **OpenEBS Mayastor** – very good performance with NVMe-oF, block only, and needs hugepages and a fairly recent kernel.
- **NFS provisioner** – fine for a homelab, but you're relying on one NFS server and that's your single point of failure.
- **Vendor SAN/NAS CSI drivers (NetApp Trident, Pure, etc.)** – solid if you already own that hardware, otherwise it's an expensive way in.

## When I'd pick Rook-Ceph

I'd choose Rook-Ceph when:

- You need block, shared filesystem **and** S3 from the same platform and don't want to run three separate systems.
- You're on bare metal or on-prem, and want something that behaves like cloud storage without depending on a cloud.
- The cluster is going to grow, more nodes, more disks, more data, and you want storage that scales out by just adding OSDs.
- You care about failure domains (racks, zones) and want control over where replicas end up.
- You have real `ReadWriteMany` needs, like shared upload directories, ML datasets or CI caches.
- The team is willing to learn a bit of Ceph. This is the real cost, don't skip it.

And I'd probably stick with Longhorn (or something simpler) when the cluster is small, the team is small, you only need block volumes, or the nodes don't have spare disks to give to Ceph.

There's no single right answer here. For us the deciding factor was not having to run separate tools for files, blocks and buckets, and Rook-Ceph was the only option that covered all three properly.

## Prerequisites

Before touching any YAML, it's worth spending ten minutes checking the nodes. Most of the "my OSDs never came up" problems I've seen come down to one of the things below. These are taken from the [Rook prerequisites page](https://rook.io/docs/rook/latest-release/Getting-Started/Prerequisites/prerequisites/) plus a few notes of my own.

### Kubernetes and CPU

- Kubernetes **v1.32 through v1.37** is supported by the current Rook release. Check yours with `kubectl version`.
- Nodes need to be **amd64 (x86_64)** or **arm64**. Nothing else is supported.
- For anything more than a test cluster, plan for at least **3 nodes** with storage, so the 3 MONs and 3 replicas can land on different hosts.

### Disks for Ceph

Ceph wants storage that nothing else is using. You need at least one of these on your storage nodes:

- Raw devices (no partitions, no filesystem)
- Raw partitions (no filesystem)
- LVM logical volumes (no filesystem)
- PVs from a storage class in `block` mode (useful in the cloud)

we will be using `Raw devices` in this setup. to keep the configuration simple, All the nodes will be have similiar configurations.

The easy way to check is `lsblk`:

```bash
$ lsblk
NAME    MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
vda     253:0    0   20G  0 disk
├─vda1  253:1    0   19G  0 part /
├─vda14 253:14   0    4M  0 part
├─vda15 253:15   0  106M  0 part /boot/efi
└─vda16 259:0    0  913M  0 part /boot
vdb     253:16   0   50G  0 disk
vdc     253:32   0    1M  0 disk
```

```bash
kubectl get nodes -o wide
NAME           STATUS   ROLES                       AGE   VERSION          INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION              CONTAINER-RUNTIME
rook-ceph-01   Ready    control-plane,etcd,worker   18m   v1.36.5+rke2r1   172.21.0.4    <none>        Ubuntu 24.04.4 LTS   6.8.0-134-generic (amd64)   containerd://2.3.4-k3s1
rook-ceph-02   Ready    control-plane,etcd,worker   14m   v1.36.5+rke2r1   172.21.0.5    <none>        Ubuntu 24.04.4 LTS   6.8.0-134-generic (amd64)   containerd://2.3.4-k3s1
rook-ceph-03   Ready    control-plane,etcd,worker   13m   v1.36.5+rke2r1   172.21.0.6    <none>        Ubuntu 24.04.4 LTS   6.8.0-134-generic (amd64)   containerd://2.3.4-k3s1
```

## Setting up rook-ceph

There are two ways to install Rook: apply the example manifests from the repo, or use the Helm charts. I went with Helm because the values file ends up being a nice record of what was actually deployed, and upgrades are a lot less painful.

Rook ships two charts:

- **rook-ceph** – the operator and the CSI drivers.
- **rook-ceph-cluster** – the `CephCluster` itself, plus pools, filesystems, object stores and StorageClasses.

Operator first, cluster second. Don't try to install both at once, the cluster chart needs the CRDs from the operator chart.

### Deploy the Rook Operator

Add the repo:

```bash
helm repo add rook-release https://charts.rook.io/release

helm repo update

helm search repo rook-ceph
NAME                           CHART VERSION APP VERSION DESCRIPTION
rook-release/rook-ceph         1.21.0        v1.21.0     File, Block, and Object Storage Services for yo...
rook-release/rook-ceph-cluster 1.21.0        v1.21.0     Manages a single Ceph cluster namespace for Rook
```

Pull the helm charts locally.

```bash
helm pull rook-release/rook-ceph
helm pull rook-release/rook-ceph-cluster
ls
get_helm.sh  rook-ceph-1.21.0.tgz  rook-ceph-cluster-1.21.0.tgz
```

Give it a minute and check the operator is up:

```bash
$ kubectl -n rook-ceph get pods
NAME                                  READY   STATUS    RESTARTS   AGE
rook-ceph-operator-6c8f7b9d5f-x2kqp   1/1     Running   0          58s
```

Nothing else is running yet, and that's expected. The operator just sits there waiting for a `CephCluster` to show up.

### Provision a Ceph Cluster

This is the part that needs a bit of thought. Out of the box the cluster chart tries to grab every empty disk on every node (`useAllDevices: true`) and also creates a block pool, a CephFS filesystem and an object store. That's a lot for a first run, and I'd rather be explicit about which disks Ceph gets.

On our nodes the free disk is `vdb` (50G). There's also a tiny 1M `vdc` on each node, which is the cloud-init config drive, and Ceph should stay well away from it. So I filter on `vdb` only:

```yaml
# cluster-values.yaml
operatorNamespace: rook-ceph

toolbox:
  enabled: true          # gives us a pod with the ceph CLI, used below

cephClusterSpec:
  mon:
    count: 3
    allowMultiplePerNode: false
  mgr:
    count: 2
    allowMultiplePerNode: false

  dashboard:
    enabled: true
    ssl: true

  storage:
    useAllNodes: true
    useAllDevices: false
    deviceFilter: "^vdb$"

  # these are small test VMs, so tone down the default requests
  resources:
    mon:
      requests: { cpu: "250m", memory: "512Mi" }
    mgr:
      requests: { cpu: "250m", memory: "512Mi" }
    osd:
      requests: { cpu: "500m", memory: "2Gi" }

cephBlockPools:
  - name: ceph-blockpool
    spec:
      failureDomain: host
      replicated:
        size: 3
    storageClass:
      enabled: true
      name: ceph-block
      isDefault: true
      reclaimPolicy: Delete
      allowVolumeExpansion: true
      parameters:
        imageFormat: "2"
        imageFeatures: layering,fast-diff,object-map,deep-flatten,exclusive-lock
        csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
        csi.storage.k8s.io/provisioner-secret-namespace: "{{ .Release.Namespace }}"
        csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
        csi.storage.k8s.io/controller-expand-secret-namespace: "{{ .Release.Namespace }}"
        csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
        csi.storage.k8s.io/node-stage-secret-namespace: "{{ .Release.Namespace }}"
        csi.storage.k8s.io/fstype: ext4

# CephFS and object storage stay off for now, they get their own post
cephFileSystems: []
cephObjectStores: []
```

A few notes on that file:

- `failureDomain: host` with `size: 3` puts one copy of every piece of data on each of our three nodes. Lose a node and you've still got two copies.
- Our kernel is 6.8, so the extra `imageFeatures` are safe (see the kernel notes above).
- `isDefault: true` makes `ceph-block` the default StorageClass. RKE2 doesn't ship one, but if you've already got `local-path` as default, unset one of them. Two defaults just causes confusion.
- `deviceFilter` is a regex. If your disks are named differently (`sdb`, `nvme1n1`), change it to match.

Install it:

```bash
helm install rook-ceph-cluster rook-release/rook-ceph-cluster \
  --namespace rook-ceph \
  -f cluster-values.yaml
```

Now go make a coffee. The operator brings things up in order: MONs first, then MGRs, then an `osd-prepare` job on each node to set up the disk, and finally the OSDs. On our three small VMs it took around 5 minutes. You can watch it with:

```bash
kubectl -n rook-ceph get pods -w
```

When it's done it should look roughly like this (pod suffixes will differ, and there's one set of csi pods per node):

```bash
$ kubectl -n rook-ceph get pods
NAME                                          READY   STATUS      RESTARTS   AGE
csi-rbdplugin-4kq9x                           3/3     Running     0          6m
csi-rbdplugin-provisioner-7d9c8f6b4-lm2xp     6/6     Running     0          6m
rook-ceph-mgr-a-5f7b9c6d8-9wq4r               3/3     Running     0          4m
rook-ceph-mgr-b-6c8d7f5b9-tz8kp               3/3     Running     0          4m
rook-ceph-mon-a-7b6c5d9f8-hx2lm               2/2     Running     0          5m
rook-ceph-mon-b-5d8f7c6b9-pq4wn               2/2     Running     0          5m
rook-ceph-mon-c-6f9b8d7c5-zr6tj               2/2     Running     0          5m
rook-ceph-operator-6c8f7b9d5f-x2kqp           1/1     Running     0          12m
rook-ceph-osd-0-7c9d8b6f5-k4mxn               2/2     Running     0          3m
rook-ceph-osd-1-8d7c6f9b5-w2plq               2/2     Running     0          3m
rook-ceph-osd-2-6b5f9d8c7-j7rtv               2/2     Running     0          3m
rook-ceph-osd-prepare-rook-ceph-01-8xkzq      0/1     Completed   0          3m
rook-ceph-osd-prepare-rook-ceph-02-m4lpw      0/1     Completed   0          3m
rook-ceph-osd-prepare-rook-ceph-03-q9vnc      0/1     Completed   0          3m
rook-ceph-tools-5b8c7d9f6-r2xmk               1/1     Running     0          6m
```

The thing to look for is **three `rook-ceph-osd-*` pods**, one per node. If the prepare jobs completed but there are no OSD pods, the disk got skipped. Nine times out of ten that's an old filesystem or partition table on it. The prepare logs will tell you:

```bash
kubectl -n rook-ceph logs -l app=rook-ceph-osd-prepare --tail=50
```

### Check the cluster health

The `CephCluster` resource gives a quick summary:

```bash
$ kubectl -n rook-ceph get cephcluster
NAME        DATADIRHOSTPATH   MONCOUNT   AGE   PHASE   MESSAGE                        HEALTH
rook-ceph   /var/lib/rook     3          7m    Ready   Cluster created successfully   HEALTH_OK
```

For more detail use the toolbox pod, it's just a container with the `ceph` CLI already configured:

```bash
$ kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
  cluster:
    id:     ...
    health: HEALTH_OK

  services:
    mon: 3 daemons, quorum a,b,c
    mgr: a(active), standbys: b
    osd: 3 osds: 3 up, 3 in

  data:
    pools:   2 pools, 33 pgs
    usage:   80 MiB used, 150 GiB / 150 GiB avail
    pgs:     33 active+clean
```

What you want: `HEALTH_OK`, all 3 MONs in quorum, `3 up, 3 in` for the OSDs, and every PG `active+clean`. Note that 150 GiB raw with 3x replication is only about 50 GiB usable, worth remembering when sizing things.

Two more I end up running all the time:

```bash
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph osd tree
kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph df
```

If you get `HEALTH_WARN` right after install, don't panic. Usually it's PGs still settling. Give it a few minutes and look at `ceph health detail`.

### Test it with a PVC

I don't trust storage until I've actually written something to it. A quick PVC and pod:

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
      command: ["sh", "-c", "echo hello from ceph > /data/hello.txt && sleep 3600"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: ceph-test
```

```bash
kubectl apply -f test-pvc.yaml
kubectl get pvc ceph-test
kubectl exec ceph-test -- cat /data/hello.txt
```

The PVC should go `Bound` within a few seconds and you should see `hello from ceph`. For a better test, delete the pod, recreate it (ideally on a different node) and check the file is still there.

Clean up after:

```bash
kubectl delete -f test-pvc.yaml
```

### Ceph dashboard

Handy for a quick visual check. Grab the admin password:

```bash
kubectl -n rook-ceph get secret rook-ceph-dashboard-password \
  -o jsonpath="{['data']['password']}" | base64 -d && echo
```

Port-forward to it:

```bash
kubectl -n rook-ceph port-forward svc/rook-ceph-mgr-dashboard 8443:8443
```

Open `https://localhost:8443`, log in as `admin` and click through the self-signed cert warning. For anything long-term, put it behind an Ingress instead.

That's a working Rook-Ceph cluster with replicated block storage. Next post I'll add CephFS for `ReadWriteMany` volumes and an object store for S3.
