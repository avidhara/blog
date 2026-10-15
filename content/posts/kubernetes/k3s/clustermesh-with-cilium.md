---
date: '2026-08-31T15:17:48+08:00'
draft: true
title: 'Setup Cluster-mesh with Cilium'
tags: [ "kubernetes", "k3s", "cilium", "clustermesh", "multicluster", "edge", "networking" ]
author: K8S team
---
# Why we need ClusterMesh for Kubernetes

Kubernetes multi-cluster architecture requires a way to connect workloads across distinct clusters without forcing all inter-cluster traffic through public internet load balancers or edge gateways. **Cluster Mesh** solves this by unifying the networking layer across multiple independent Kubernetes clusters into a single flat network plane.
![alt](/images/cilium-mesh.webp)
---

**Core Reasons to Use Cluster Mesh**

* **Direct, High-Performance Pod-to-Pod Communication:** Pods in `Cluster-A` can talk directly to Pods in `Cluster-B` using internal IPs without traversing ingress controllers, external load balancers, or NAT proxies. This drastically reduces network latency and cuts cloud ingress/egress costs.
* **High Availability & Failover (Global Services):** You can define "Global Services" that span across regions or availability zones. If the local backends in `Cluster-A` crash or experience an outage, traffic automatically routes to healthy backends running in `Cluster-B` without application code changes.
* **Unified Security Policies (Identity-Aware Filtering):** Workload security identity (such as Kubernetes labels/namespaces) is preserved across cluster boundaries. You can write native `NetworkPolicies` that enforce rules like *"Allow Frontend Pods in Cluster-A to access Database Pods in Cluster-B"*.
* **Shared Central Services:** Instead of duplicating expensive infrastructure services (like Prometheus, Vault, DNS, or logging aggregators) inside every single cluster, you can run them once in a dedicated management cluster and share them securely across tenant clusters.
* **Blast-Radius Isolation & Compliance:** Hard multi-tenancy often requires keeping production, staging, or PCI-compliant environments on separate Kubernetes control planes. Cluster Mesh connects these isolated environments securely over private IP space without compromising control-plane isolation.

---

**Cluster Mesh vs. Alternatives**

| Feature | Ingress Gateway / External LB | Service Mesh (e.g., Istio Multicluster) | Cluster Mesh (e.g., Cilium) |
| --- | --- | --- | --- |
| **Routing Layer** | Layer 7 / Edge | Layer 7 (Envoy Sidecars/Proxies) | Layer 3/4 (eBPF Kernel Level) |
| **Performance Overhead** | High (Multiple hops & SSL terminations) | Medium-High (Proxy CPU/Memory consumption) | Negligible (Direct node-to-node routing) |
| **Traffic Scope** | Exposed Services only | HTTP/gRPC/TCP mesh traffic | Flat Pod network for all protocols |
| **Setup Prerequisites** | Public/External IPs required | Sidecar injection & proxy management | Non-overlapping Pod CIDRs & CNI support |

---

**Primary Prerequisites for setting up ClusterMesh**

1. **CNI Support:** Modern cluster mesh functionality is typically provided by container network interfaces (CNIs) like **Cilium** (using eBPF) or **Calico**.
2. **Non-Overlapping Pod CIDRs:** Each cluster must use distinct Pod/Service IP ranges so routing between clusters does not suffer IP address collisions.
3. **Direct Node-to-Node IP Reachability:** Nodes across clusters must be able to reach each other via direct routing or VPN tunnels.

## Setup K3S Clusters with Cilium as CNI

### Step 1: Install K3s without Flannel

We will run the K3s installation script with the `--flannel-backend=none` flag. We also disable the default network policy controller, as Cilium will handle policy enforcement.

Add configurations for K3S.

```bash
mkdir -p /etc/rancher/k3s
```

```bash
cat << EOF > /etc/rancher/k3s/config.yaml
flannel-backend: none
disable-kube-proxy: true
disable-network-policy: true
cluster-cidr: "10.42.0.0/16"
service-cidr: "10.43.0.0/16"
EOF
```

```bash
curl -sfL https://get.k3s.io | sh -
```

Check the node status. It will show as `NotReady` because there is no CNI installed to manage pod networking:

```bash
kubectl get nodes
# NAME          STATUS     ROLES                  AGE   VERSION
k3s-02   NotReady   control-plane   7s    v1.36.4+k3s1
```

Repeat Above steps for cluster-2 as well, Make sure Cluster CIDR & Service CIDR are different between the clusters.

Here is an example for cluster-2,

```bash
cat << EOF > /etc/rancher/k3s/config.yaml
flannel-backend: none
disable-kube-proxy: true
disable-network-policy: true
cluster-cidr: "10.52.0.0/16"
service-cidr: "10.53.0.0/16"
EOF
```

### Install Helm and Cilium CLI

**Helm**

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4
chmod 700 get_helm.sh
./get_helm.sh
```

**Cilium CLI**

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

### Deploy Cilium as CNI

Create the manifest file:

```yaml
cat > /var/lib/rancher/k3s/server/manifests/cilium.yaml <<EOF
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: cilium
  namespace: kube-system
spec:
  chart: cilium
  repo: https://helm.cilium.io/
  targetNamespace: kube-system
  version: 1.20.1
  bootstrap: true
  valuesContent: |
    kubeProxyReplacement: true
    autoDirectNodeRoutes: true 
    bpf:
      hostLegacyRouting: false 
      masquerade: true
    cluster:
        id: 1
        name: k3s-01
    operator:
      replicas: 1
    cni:
      chainingMode: none 
      exclusive: true 
      installImage: true
    dashboards:
      enabled: true
    enableIPv4Masquerade: true 
    hubble:
      enabled: true 
      relay:
        enabled: true
      ui:
        enabled: true
    ipv4NativeRoutingCIDR: 10.42.0.0/16
    k8sServiceHost: 127.0.0.1
    k8sServicePort: '6443'
    kubeProxyReplacement: 'true' 
    routingMode: native 
    securityContext:
      privileged: true
EOF
```

check cilium status

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Check Cilium status

cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 1, Ready: 1/1, Available: 1/1
DaemonSet              cilium-envoy             Desired: 1, Ready: 1/1, Available: 1/1
Deployment             cilium-operator          Desired: 1, Ready: 1/1, Available: 1/1
Deployment             hubble-relay             Desired: 1, Ready: 1/1, Available: 1/1
Deployment             hubble-ui                Desired: 1, Ready: 1/1, Available: 1/1
Containers:            cilium                   Running: 1
                       cilium-envoy             Running: 1
                       cilium-operator          Running: 1
                       clustermesh-apiserver
                       hubble-relay             Running: 1
                       hubble-ui                Running: 1
Cluster Pods:          7/7 managed by Cilium
Helm chart version:    1.20.1
Image versions         cilium             quay.io/cilium/cilium:v1.20.1@sha256:ae9ea21f7427fe24bc6ea7247eb552157a1b0a431744045d3f641545ca71d11b: 1
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.37.5-1786810558-766ccfb37260a43e9d228837aa84ce3faf9f64e7@sha256:75b8094c7127736a2ffd2dce3945e0931cb6df21b0372ff661940eca26730b91: 1
                       cilium-operator    quay.io/cilium/operator-generic:v1.20.1@sha256:6c3885fc7b629099fdbe2a5c87869c86feb825fa18fae299eac0f61918d16ecf: 1
                       hubble-relay       quay.io/cilium/hubble-relay:v1.20.1@sha256:59be0ae7d475ab9011a5e954618c0f27b5778b17140381425b308b55ba4917f4: 1
                       hubble-ui          quay.io/cilium/hubble-ui-backend:v0.13.5@sha256:fac0c300ae119274edca11fd89b1ad23c788792d8bc4ea2ba631c709e8d3c688: 1
                       hubble-ui          quay.io/cilium/hubble-ui:v0.13.5@sha256:f7d514fc54d784ed6df9d58cf0e97648b143f92b766dd1780ed3fc845bd4c516: 1
```

you should be able to see all the componets are up and running and `ClusterMesh:        disabled` state.

repeat this step for `cluster-2` as well.

update  cluster ID and and me before deploying to `cluster-2`.

```yaml
cluster:
  id: 2
  name: k3s-02
ipv4NativeRoutingCIDR: 10.52.0.0/16
```

Check cilium status on cluster-2 as well.

### Enable Cluster Mesh on both the clusters

Once cilium us up and running in both cluster-1 and cluster-2 we are ready to configure the cilium cluster-mesh.

#### Configure TLS certificates

Cluster Mesh uses mutual TLS between the Cilium agents of one cluster and the `clustermesh-apiserver` of the other. For the clusters to trust each other, **all clusters must share the same Cilium CA** (stored in the `cilium-ca` secret in `kube-system`).

By default each Helm install generates its own random CA, so we generate one CA up front and pass it to both clusters through Helm values.

we will be using Helm approch for this installation you can use cert-manager, or custom ca certificates as well.

to use cert-manager or custom ca please refere /[here](https://docs.cilium.io/en/stable/network/clustermesh/setup/#configure-tls-certificates)

#### Configure Cluster Mesh

update cilium HelmChart config for k3s-01, update `$CLUSTER1_NODE_IP` and `$CLUSTER2_NODE_IP`

```yaml
clustermesh:
  useAPIServer: true
  config:
    enabled: true
    clusters:
      cluster1:
        address: $CLUSTER1_NODE_IP
        port: 32379
      cluster2:
        address: $CLUSTER2_NODE_IP
        port: 32379
  apiserver:
    service:
      type: NodePort
      nodePort: 32379
    tls:
      auto:
        enabled: true
        method: cronJob
```

Once the configs are updated, cilium will deploy `clustermesh-apiserver` and creates CA certificate in secret with name `cilium-ca`.

Copy the CA certificate from k3s-01 to k3s-02.

Both clusters must share the same CA, so k3s-02's own `cilium-ca` (created when Cilium was first installed) has to be replaced with the one from k3s-01. The commands below assume both clusters are reachable as kubeconfig contexts (`$CLUSTER1=k3s-01`, `$CLUSTER2=k3s-02`).

```bash
export CLUSTER1=k3s-01
export CLUSTER2=k3s-02

# Replace the existing cilium-ca on k3s-02 with the one from k3s-01
kubectl --context $CLUSTER1 -n kube-system get secret cilium-ca -o yaml \
  | grep -vE '^\s+(resourceVersion|uid|creationTimestamp):' \
  | kubectl --context $CLUSTER2 -n kube-system replace --force -f -
```

> **Note:** `create` fails because k3s-02 already has a `cilium-ca` secret. `replace --force` deletes it and recreates it. The `grep` removes metadata that only applies to the source cluster.

Check that both clusters now have the same CA:

```bash
diff <(kubectl --context $CLUSTER1 -n kube-system get secret cilium-ca -o jsonpath='{.data.ca\.crt}') \
     <(kubectl --context $CLUSTER2 -n kube-system get secret cilium-ca -o jsonpath='{.data.ca\.crt}') \
  && echo "CA matches"
```

Now repeat the cilium HelmChart config update on k3s-02. Then restart Cilium so the agents and operator pick up the shared CA:

```bash
kubectl --context $CLUSTER2 -n kube-system rollout restart ds/cilium deploy/cilium-operator
```

Now enable the `clustermesh-apiserver` and tell each cluster how to reach its peer. Each k3s cluster here has a single node, so we expose the apiserver via `NodePort` and point the peer at the node IP.

> **Note:** Because both clusters share the same CA, the per-cluster `tls.cert`/`tls.key`/`tls.caCert` fields under `clustermesh.config.clusters[]` can be omitted.

| Value | Purpose |
| --- | --- |
| `cluster.id` / `cluster.name` | Set earlier in the HelmChart. Must be unique per cluster (`id` 1–255). Keys under `clustermesh.config.clusters` must match these names. |
| `clustermesh.useAPIServer: true` | Deploys `clustermesh-apiserver` (etcd + kvstoremesh). It shares this cluster's state with its peers. |
| `clustermesh.config.enabled: true` | Generates the `cilium-clustermesh` secret the agents use to connect to remote clusters. |
| `clustermesh.config.clusters.<name>.address` | Address where the peer's `clustermesh-apiserver` can be reached. Here, the node IP of each k3s cluster. |
| `clustermesh.config.clusters.<name>.port` | Port of the peer's apiserver. It must match the peer's `nodePort` (`32379`). |
| `clustermesh.apiserver.service.type: NodePort` | Exposes the apiserver on every node. This suits single-node k3s without a LoadBalancer. |
| `clustermesh.apiserver.service.nodePort: 32379` | Fixed NodePort, so the peer config stays the same on both clusters. |
| `clustermesh.apiserver.tls.auto.enabled: true` | Generates the server, admin and remote client certificates automatically. |
| `clustermesh.apiserver.tls.auto.method: cronJob` | A Job/CronJob generates the certificates and renews them periodically, signed by `cilium-ca`. The alternatives are `helm` (generated at install time) and `certmanager`. |

> **Tip:** The same `clusters` list can be used on both clusters. Cilium ignores the entry that matches its own `cluster.name`, so each cluster only connects to its peer.

#### Validate Cluster Mesh

Wait for the Cluster Mesh components to come up by running `cilium clustermesh status --wait` against both clusters:

```bash
cilium clustermesh status --context $CLUSTER1 --wait
cilium clustermesh status --context $CLUSTER2 --wait
```

The output should look similar to this (from k3s-01):

```bash
✅ Cluster access information is available:
  - <CLUSTER1_NODE_IP>:32379
✅ Service "clustermesh-apiserver" of type "NodePort" found
⌛ Waiting (12s) for clusters to be connected: 1 nodes are not ready
✅ All 1 nodes are connected to all clusters [min:1 / avg:1.0 / max:1]
🔌 Cluster Connections:
  - k3s-02: 1/1 configured, 1/1 connected
```

`cilium status --context $CLUSTER1` should now show `ClusterMesh: 1/1 remote clusters ready` instead of `disabled`.

#### Test Pod Connectivity Between Clusters

Once both clusters report as connected, validate cross-cluster connectivity by running the connectivity test in multi-cluster mode:

```bash
cilium connectivity test --context $CLUSTER1 --multi-cluster $CLUSTER2
```

#### Troubleshooting

If validation does not succeed:

1. Validate that Cilium pods are healthy and ready:

   ```bash
   cilium status --context $CLUSTER1
   cilium status --context $CLUSTER2
   ```

2. Validate that Cluster Mesh is enabled and operational:

   ```bash
   cilium clustermesh status --context $CLUSTER1
   cilium clustermesh status --context $CLUSTER2
   ```

3. Check that both clusters share the same `cilium-ca`, that TCP `32379` is reachable between the node IPs, and that the keys under `clustermesh.config.clusters` match each cluster's `cluster.name`.

For more detail, see the [Cluster Mesh Troubleshooting](https://docs.cilium.io/en/stable/operations/troubleshooting/#troubleshooting-clustermesh) guide.

#### Test with a Global Service

Deploy the same Service in both clusters with the `service.cilium.io/global` annotation:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: echo
  annotations:
    service.cilium.io/global: "true"
spec:
  selector:
    app: echo
  ports:
    - port: 80
      targetPort: 8080
```

Requests to `echo` from either cluster are now load-balanced across backends in both clusters. Add `service.cilium.io/affinity: "local"` to prefer local backends and only fail over to the remote cluster when local ones are unavailable.
