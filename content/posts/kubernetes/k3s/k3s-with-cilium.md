---
date: '2026-05-28T11:02:11+08:00'
draft: false
title: "Setup Cilium as CNI for K3S cluster"
tags: [ "edge", "k3s", "rancher", "fleet", "rke2", "kubernetes", "k8s"]
author: K8S team
---

If you have ever spun up a K3s cluster, you know how easy it is to get going. Out of the box, K3s uses Flannel for its Container Network Interface (CNI). It is lightweight, works immediately, and is great for local development or basic setups. 

But once you start pushing K3s into production edge environments—like retail shops, factory floors, or remote gateway devices—you quickly run into a different set of challenges. Edge nodes are usually resource-constrained, exposed to unstable networks, physically vulnerable, and geographically scattered. 

Under these conditions, basic host-to-host overlay networking is not enough. You need fine-grained control, robust security, and deep visibility, all without eating up the limited CPU and RAM on your hardware.

This is why many teams swap Flannel for **Cilium**. By moving networking, security, and observability into the Linux kernel using **eBPF (Extended Berkeley Packet Filter)**, Cilium changes how we manage edge cluster networks.

Here is a breakdown of why Cilium makes sense for edge Kubernetes setups, followed by a step-by-step walkthrough to configure it on your K3s cluster.

---

## Why Cilium Matters at the Edge

Traditional Linux networking relies on `iptables` or `IPVS` to route packets and load balance services. Every new service adds more rules. At the edge, on single-board computers or low-power gateways, sequentially evaluating hundreds of `iptables` rules wastes CPU cycles and spikes latency.

Cilium avoids this entirely. It runs eBPF programs directly at the network socket layer, providing several distinct advantages for edge architectures:

### 1. eBPF Performance with Minimal Footprint
Instead of traversing the full TCP/IP stack and checking `iptables` lists, Cilium handles packet routing in kernel space. 
- **Low Overhead**: Significantly reduces CPU and memory usage, leaving more resources for your actual workloads.
- **Fast Local Paths**: If two containers on the same edge node talk to each other, Cilium short-circuits the path at the socket layer (`sockops`), bypassing the network stack entirely.

### 2. Sidecarless Service Mesh
If you need service mesh capabilities like mutual TLS (mTLS), L7 traffic splitting, or ingress control, standard solutions like Istio inject an Envoy sidecar proxy into every single pod. In resource-constrained environments, this sidecar tax can easily double your memory overhead. 
Cilium handles these features at the kernel level or through a single, shared Envoy instance per node. You get the benefits of a service mesh without paying the sidecar resource tax.

### 3. Traffic Control over Unstable WANs
Edge locations often rely on cellular links, satellite connections, or public internet connections with unpredictable latency. 
- **Native Bandwidth Allocation**: Cilium's Bandwidth Manager uses eBPF to limit egress traffic directly at the container level. This stops a single noisy workload from consuming all available bandwidth.
- **BBR Congestion Control**: BBR optimizes container traffic flow over high-loss, high-latency WAN links, keeping communication stable even during network drops.

### 4. Pragmatic Security for Exposed Nodes
Physical nodes at edge locations are vulnerable to tampering or unauthorized network access. Traditional IP-based firewall rules do not work well in dynamic Kubernetes environments.
- **Identity-Based Policies**: Cilium assigns secure cryptographic identities based on pod labels. Even if a pod's IP changes or the host network is compromised, policies remain consistent.
- **L7 and DNS-Aware Filtering**: You can write rules restricting traffic to specific domains (e.g., only allow connections to your telemetry endpoint at `telemetry.company.com`) or specific HTTP methods, shutting down unauthorized lateral movement.

### 5. Hubble: Observability That Actually Works Remotely
Debugging networking issues on a remote gateway is difficult. You rarely have direct ssh access, and shipping verbose logs over cellular networks is expensive.
Hubble captures L3/L4 flow logs and HTTP/DNS metadata directly in the kernel with negligible overhead. You can visualize service maps and troubleshoot connectivity issues from a central dashboard without overloading the edge device.

---

## Moving K3s from Flannel to Cilium

To get Cilium running, we must install K3s with its default CNI backend disabled, then deploy Cilium.

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
EOF
```

```bash
curl -sfL https://get.k3s.io | sh -
```

Check the node status. It will show as `NotReady` because there is no CNI installed to manage pod networking:

```bash
kubectl get nodes
# NAME          STATUS     ROLES                  AGE   VERSION
# edge-node-1   NotReady   control-plane,master   10s   v1.32.3+k3s1
```

### Step 2: Deploy Cilium via K3s Helm Controller

One of K3s' best features is its built-in Helm controller. It lets you deploy Helm charts declaratively simply by placing a manifest file in `/var/lib/rancher/k3s/server/manifests/`. K3s will pick it up on startup and handle the installation automatically.

We will create a `HelmChart` definition for Cilium. This defines the repository, version, namespace, and includes the customized values file we need for edge setups (resource limits, eBPF routing, bandwidth management, and Hubble observability).

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

Save this file, and the K3s Helm controller will immediately begin installing Cilium.

> [!NOTE]
> Make sure to replace `[IP_ADDRESS]` with the actual IP address of your control plane node. By default, K3s uses port `6443` for the API server. If you are deploying this on RKE2, you may need to use `6444` instead.
> Since we set `disable-kube-proxy: true` in the K3s configuration, we must set `kubeProxyReplacement: true` and define the API server endpoint so the Cilium agent can bootstrap and connect to the API server without relying on kube-proxy.

### Step 3: Monitor the Cilium Installation

Once you save the manifest file, K3s' built-in Helm controller will start a job to install Cilium.

Because the cluster does not have a CNI running yet, all other system pods (like CoreDNS, local-path-provisioner, and Traefik) will remain in a `Pending` state. This is completely normal and expected.

You can watch the bootstrap process by running:

```bash
kubectl get pods -A
```

At first, you will see the `helm-install-cilium` pod running, while everything else waits:

```text
NAMESPACE     NAME                                      READY   STATUS              RESTARTS   AGE
kube-system   coredns-8db54c48d-b4nbl                   0/1     Pending             0          40s
kube-system   helm-install-cilium-hlbc5                 0/1     ContainerCreating   0          3s
kube-system   helm-install-traefik-crd-bwpqx            0/1     Pending             0          35s
kube-system   helm-install-traefik-l9sjp                0/1     Pending             0          35s
kube-system   local-path-provisioner-5d9d9885bc-lpx4c   0/1     Pending             0          40s
kube-system   metrics-server-786d997795-2nzjm           0/1     Pending             0          40s
```

Once the Helm install job completes, the Cilium daemonset and Envoy pods will deploy:

```bash
kubectl get pods -n kube-system -l k8s-app=cilium
```

You should see the Cilium pods transition to `Running`:

```text
NAME                                      READY   STATUS    RESTARTS   AGE
cilium-envoy-mmd6l                        1/1     Running   0          21s
cilium-qrm75                              1/1     Running   0          22s
```

### Step 4: Verify the eBPF Datapath

To make sure everything is fully operational, we can install the Cilium CLI to run deep connectivity tests and check the status of the eBPF datapath.

First, download and install the CLI on your control plane node:

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi

curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

Now, verify the cluster node status. The K3s node status will shift to `Ready`:

```bash
kubectl get nodes
# NAME          STATUS   ROLES                  AGE    VERSION
# edge-node-1   Ready    control-plane,master   2m5s   v1.32.3+k3s1
```

Confirm that the CNI is healthy and active:

```bash
cilium status --wait
```

You can also run a comprehensive connectivity test to verify eBPF packet routing across pods and hosts:

```bash
cilium connectivity test
```

---

## Wrapping Up

Moving your K3s setup from Flannel to Cilium is a solid upgrade if you are targeting production edge deployments. Replacing `iptables` with eBPF addresses the exact pain points edge developers face: high resource overhead, unpredictable networks, and complex security requirements. 

With Cilium, you get a clean path to multi-cluster connectivity (via Cluster Mesh), network observability (via Hubble), and container-level bandwidth control, all running efficiently in kernel space.


## Reference Links
- [Official Cilium Docs](https://docs.cilium.io/)
- [K3s Installation Parameters](https://docs.k3s.io/installation/advanced)

Happy Kubernetes-ing! 🚀