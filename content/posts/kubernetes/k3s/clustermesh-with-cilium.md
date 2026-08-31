---
date: '2026-08-31T15:17:48+08:00'
draft: true
title: 'Setup Clustermesh with Cilium'
tags: [ "kubernetes", "k3s", "cilium", "clustermesh", "multicluster", "edge", "networking" ]
author: K8S team
---
# Why we need ClusterMesh for Kubernetes

Kubernetes multi-cluster architecture requires a way to connect workloads across distinct clusters without forcing all inter-cluster traffic through public internet load balancers or edge gateways. **Cluster Mesh** solves this by unifying the networking layer across multiple independent Kubernetes clusters into a single flat network plane.

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
        name: k3s-02
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

repeat this step for cluster-2 as well.

### Enable Cluster Mesh on both the clusters
