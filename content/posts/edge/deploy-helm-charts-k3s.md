---
date: "2025-04-22T08:48:17+05:30"
draft: false
title: "Simplifying Helm Chart Deployment on K3s: A Step-by-Step Guide"
tags: ["edge", "k3s", "rancher", "helm", "helm-controller","k8s","helm-charts","kubernetes"]
author: Edge Computing team
---

## Introduction

K3s is a lightweight Kubernetes distribution designed for resource-constrained environments. It is ideal for edge computing, IoT devices, and development environments. Helm is a package manager for Kubernetes that simplifies the deployment and management of applications on Kubernetes clusters.

If you are looking for a simple way to deploy Helm charts on K3s, you can use the Helm controller.
Helm Controller comes with a set of custom resources that allow you to manage Helm releases declaratively. This means you can define your Helm releases in YAML files and apply them to your K3s cluster using kubectl.

## Prerequisites

- A K3s cluster up and running. You can install K3s on your local machine or on a remote server. For more information, see the [Deploying K3s on ARM-Based Architecture](https://blog.avidhara.cloud/posts/edge/running-k3s-arm/).
- kubectl installed and configured to access your K3s cluster. You can install kubectl by following the [kubectl installation guide](https://kubernetes.io/docs/tasks/tools/install-kubectl/).

### Deploying Helm Charts on K3s using Helm Operator

In this example, we will be creating an namespace and Helm Release for Bitnami nginx chart.

current helm chats that are already deployed into the cluster can be listed using the following command:

```bash
kubectl get HelmChart -A
NAMESPACE     NAME          JOB                        CHART                                                                      TARGETNAMESPACE   VERSION   REPO   HELMVERSION   BOOTSTRAP
kube-system   traefik       helm-install-traefik       https://%{KUBERNETES_API}%/static/charts/traefik-34.2.1+up34.2.0.tgz
kube-system   traefik-crd   helm-install-traefik-crd   https://%{KUBERNETES_API}%/static/charts/traefik-crd-34.2.1+up34.2.0.tgz
```

```bash
cat > nginx-helm-release.yaml <<EOF
# Create a namespace for the Helm release
apiVersion: v1
kind: Namespace
metadata:
  name: web
---
apiVersion: helm.cattle.io/v1
kind: HelmChart
metadata:
  name: apache # Name of the Release
  namespace: kube-system
spec:
  repo: https://charts.bitnami.com/bitnami # URL of the Helm chart repository
  chart: apache # Name of the Helm chart
  targetNamespace: web # Namespace where the chart will be deployed
  # Override the default values.yaml file of the chart
  valuesContent: |-
    service:
      type: ClusterIP
EOF

kubectl apply -f nginx-helm-release.yaml
namespace/web created
helmchart.helm.cattle.io/apache created
```

Internally this will create an Kubernetes Job to deploy the Helm chart. You can check the status of the job using the following command:

```bash
kubectl get job -n kube-system | grep web
kube-system           helm-install-apache-wd8gn                 0/1     Completed   0               17s
```

you can check the resources that are deployed in the web namespace using the following command:

```bash
kubectl get all -n web
NAME                          READY   STATUS    RESTARTS   AGE
pod/apache-7768c88d7b-6zzwz   1/1     Running   0          90s

NAME             TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
service/apache   ClusterIP   10.43.46.12   <none>        80/TCP,443/TCP   90s

NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/apache   1/1     1            1           90s

NAME                                DESIRED   CURRENT   READY   AGE
replicaset.apps/apache-7768c88d7b   1         1         1       90s
```

### Troubleshooting

- If the Helm chart fails to deploy, check the logs of the respective Kubernetes job:

```bash
kubectl get jobs -n kube-system
NAME                       STATUS     COMPLETIONS   DURATION   AGE
helm-install-apache        Complete   1/1           15s        3m
helm-install-traefik       Complete   1/1           22s        22h
helm-install-traefik-crd   Complete   1/1           22s        22h
## To check the logs of a specific job, use:
kubectl logs -f helm-install-apache-wd8gn -n kube-system

```

### Conclusion

In this blog post, we have demonstrated how to deploy Helm charts on K3s using the Helm controller. This approach allows you to manage your Helm releases declaratively, making it easier to maintain and update your applications on K3s clusters. The Helm controller is a powerful tool that simplifies the deployment process and provides a consistent way to manage Helm charts in Kubernetes.

## Additional Resources

- [K3s Documentation](https://rancher.com/docs/k3s/latest/en/)
- [Helm Controller Documentation](https://github.com/k3s-io/helm-controller)

Ready to simplify your Kubernetes deployments? Try deploying your own Helm charts on K3s today and share your experience in the comments!

You can find other posts related to K3s and edge computing on our [blog](https://blog.avidhara.cloud/tags/k3s/). If you have any questions or need assistance, feel free to reach out!

Happy Kubernetes-ing! 🚀
