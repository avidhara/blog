---
date: "2025-04-21T18:49:04+05:30"
draft: true
title: "Introduction to Fleet"
tags:
  [
    "edge",
    "k3s",
    "rancher",
    "fleet",
    "rke2",
    "ci-cd",
    "gitops",
    "kubernetes",
    "k8s",
    "continuous delivery",
    "multi-cluster",
    "gitops-based",
    "declarative configuration",
  ]
author: Edge Computing team
---

## Introduction

Fleet is a GitOps-based continuous delivery tool for Kubernetes clusters. It is designed to manage the deployment of applications and configurations across multiple clusters, making it an ideal solution for organizations that operate at scale. Fleet provides a unified way to manage your Kubernetes resources, ensuring that your resources are always in the desired state.
Fleet is built on top of Rancher and RKE2, which are popular tools for managing Kubernetes clusters. It allows you to define your desired state for your clusters in a Git repository, and Fleet will automatically apply those configurations to your clusters. This approach ensures that your clusters are always in sync with your desired state, reducing the risk of configuration drift and making it easier to manage large numbers of clusters.

## Key Features

- **GitOps-based**: Fleet uses Git as the source of truth for your cluster configurations, allowing you to manage your clusters using familiar Git workflows.
- **Multi-cluster management**: Fleet can manage thousands of clusters, making it ideal for organizations with large-scale Kubernetes deployments.
- **Declarative configuration**: Fleet allows you to define your desired state for your clusters using YAML files, making it easy to version control and manage your configurations.
- **Integration with Rancher and RKE2**: Fleet is built on top of Rancher and RKE2, providing a seamless experience for managing your Kubernetes clusters.
- **Continuous delivery**: Fleet automatically applies your configurations to your clusters, ensuring that they are always in the desired state.

### Architecture

Fleet consists of several components that work together to provide a unified way to manage your Kubernetes clusters. The key components of Fleet include:

![alt](/images/fleet-arch.png)

- **Fleet Controller**: The Fleet Controller is responsible for managing the state of your clusters. It monitors your Git repository for changes and applies those changes to your clusters.
- **Fleet Agent**: The Fleet Agent is deployed on each of your clusters and is responsible for applying the configurations defined in your Git repository to the cluster. It communicates with the Fleet Controller to receive updates and report the status of the cluster.
- **GitJob**: GitJob is a Kubernetes custom resource that represents a job that runs in your cluster. It is used to manage the deployment of applications and configurations to your clusters.

![alt](/images/fleetComponents.svg)

### Installation

Fleet can be installed using Helm, a popular package manager for Kubernetes. The installation process involves adding the Fleet Helm repository, installing the Fleet CRDs (Custom Resource Definitions), and then installing Fleet itself.

#### pre-requisites

- A Kubernetes cluster (RKE2 or Rancher)
- kubectl installed and configured to access your cluster
- Helm installed

```bash
# Add the Fleet Helm repository
helm repo add fleet https://rancher.github.io/fleet-helm-charts/
"fleet" has been added to your repositories
```

Install Fleet using Helm:

```bash
# Install Fleet CRDs
helm -n cattle-fleet-system install --create-namespace --wait fleet-crd \
    fleet/fleet-crd
NAME: fleet-crd
LAST DEPLOYED: Tue Apr 22 00:12:42 2025
NAMESPACE: cattle-fleet-system
STATUS: deployed
REVISION: 1
TEST SUITE: None

### Install Fleet
helm -n cattle-fleet-system install --create-namespace --wait fleet \
    fleet/fleet
NAME: fleet
LAST DEPLOYED: Tue Apr 22 00:12:59 2025
NAMESPACE: cattle-fleet-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

### Verify the installation

```bash
# Check the status of the Fleet components
kubectl get pods -n cattle-fleet-system
NAME                               READY   STATUS    RESTARTS   AGE
fleet-agent-6cfd675dc4-vpmbb       1/1     Running   0          13s
fleet-controller-8bf79cc4b-hfgts   3/3     Running   0          31s
gitjob-5f545884fc-ngprk            1/1     Running   0          31s

kubectl get svc -n cattle-fleet-system
NAME                          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
gitjob                        ClusterIP   10.105.218.209   <none>        80/TCP     78s
monitoring-fleet-controller   ClusterIP   10.98.137.109    <none>        8080/TCP   78s
monitoring-gitjob             ClusterIP   10.97.173.80     <none>        8081/TCP   78s

```

### Using Fleet for Continuous Delivery

Once Fleet is installed, you can start using it to manage your Kubernetes clusters. The first step is to create a Git repository that will serve as the source of truth for your cluster configurations. You can define your desired state for your clusters using YAML files and commit those files to your Git repository.
Fleet will automatically monitor your Git repository for changes and apply those changes to your clusters.
This approach ensures that your clusters are always in sync with your desired state, reducing the risk of configuration drift and making it easier to manage large numbers of clusters.

#### Example

```bash
cat > example.yaml << "EOF"
kind: GitRepo
apiVersion: fleet.cattle.io/v1alpha1
metadata:
  name: simple-namespace
  namespace: fleet-local
spec:
  repo: https://github.com/rajivreddy/fleet-examples.git
  branch: main
  paths:
  - namespace
EOF

kubectl apply -f example.yaml

gitrepo.fleet.cattle.io/sample created

```

Check the status of the GitRepo:

```bash
# Check the status of the GitRepo
kubectl get gitrepo  -n fleet-local
NAME               REPO                                               COMMIT                                     BUNDLEDEPLOYMENTS-READY   STATUS
simple-namespace   https://github.com/rajivreddy/fleet-examples.git   cf4faf61e442044bcf3464145d2d5b9158457211   1/1

### check the resources created in the cluster(namespace)
NAME                                     STATUS   AGE
cattle-fleet-clusters-system             Active   16m
cattle-fleet-system                      Active   17m
cluster-fleet-local-local-1a3d67d0a899   Active   16m
default                                  Active   14d
fleet-local                              Active   16m
fleet-manifest-example                   Active   4m14s
kube-node-lease                          Active   14d
kube-public                              Active   14d
kube-system                              Active   14d
monitoring                               Active   9s
```

### Check the resouces bundle created in the cluster

```bash
# Check the status of the Bundle
kubectl get bundles.fleet.cattle.io -n fleet-local
NAME                         BUNDLEDEPLOYMENTS-READY   STATUS
fleet-agent-local            1/1
simple-namespace-namespace   1/1
####
kubectl getbundles.fleet.cattle.io simple-namespace-namespace -n fleet-local -o yaml
apiVersion: fleet.cattle.io/v1alpha1
kind: Bundle
metadata:
  creationTimestamp: "2025-04-21T18:59:55Z"
  finalizers:
  - fleet.cattle.io/bundle-finalizer
  generation: 1
  labels:
    fleet.cattle.io/commit: cf4faf61e442044bcf3464145d2d5b9158457211
    fleet.cattle.io/repo-name: simple-namespace
  name: simple-namespace-namespace
  namespace: fleet-local
  resourceVersion: "221507"
  uid: 0cecf7b3-a0fd-472a-a1dd-87845ceb501e
spec:
  ignore: {}
  resources:
  - content: |
      apiVersion: v1
      kind: Namespace
      metadata:
        name: monitoring
        annotations:
          owner: "DevOps Team"
    name: namespace.yaml
  targetRestrictions:
  - clusterGroup: default
    name: default
  targets:
  - clusterGroup: default
    ignore: {}
    name: default
```

```bash
kubectl get clusters.fleet.cattle.io -A
NAMESPACE     NAME    BUNDLES-READY   LAST-SEEN              STATUS
fleet-local   local   2/2             2025-04-21T18:58:30Z

kubectl get clustergroups.fleet.cattle.io -A
NAMESPACE     NAME      CLUSTERS-READY   BUNDLES-READY   STATUS
fleet-local   default   1/1              2/2
```

### Registering an Downstream cluster

Assuming you are trying to register a downstream cluster to the fleet server without Rancher, you can do so by running the following command:

```bash
# Add the Fleet Agent Helm repository
helm repo add fleet https://rancher.github.io/fleet-helm-charts/
```

```bash
# Install the Fleet Agent on the downstream cluster
helm -n cattle-fleet-system install --create-namespace --wait \
    $CLUSTER_LABELS \
    --values values.yaml \
    --set apiServerCA="$API_SERVER_CA_DATA" \
    --set apiServerURL="$API_SERVER_URL" \
    fleet-agent fleet/fleet-agent
```

Validate Fleet agent Deployments

```bash
# Ensure kubectl is pointing to the right cluster
kubectl -n cattle-fleet-system logs -l app=fleet-agent
kubectl -n cattle-fleet-system get pods -l app=fleet-agent
```

once fleet agent is installed, you can check the status of the downstream cluster using kubectl commands.

```bash
kubectl -n clusters get clusters.fleet.cattle.io

```

### Conclusion
