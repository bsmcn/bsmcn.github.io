---
title: "Bootstrapping Flux on a CNI-less cluster"
last_modified_at: 2026-08-31T21:00:00+02:00
categories:
  - gitops
tags:
  - kubernetes
  - k8s
  - flux
  - cilium
  - terraform
---

## Goal
The goal is to bootstrap Flux on a Kubernetes cluster before a CNI plugin is installed. The CNI (Cilium in this case) would then be deployed by Flux and managed as a Helm resource. Without network connectivity, Flux initially has to run in the host network.

## Setup
- 3-node Talos cluster running without a CNI
- Flux operator bootstrap (Terraform)[^1]
- Flux operator Helm chart[^2]

## Config
### Talos (k8s cluster)
Disable the CNI in your cluster patch file.
```yaml
cluster:
  network:
    cni:
      name: none
```

### Terraform Flux Bootstrap module
- The variable definitions are not included
- The labels in the `common_metadata` block relax the Talos namespace security defaults and let the pods run in the host network
- The toleration list has to be extended so the workloads can be scheduled
- The k8s API endpoint is overridden for both the job and the chart to force traffic over the LAN network

```json
provider "helm" {
  kubernetes = {
    host           = var.cluster_endpoint
    config_path    = "~/.kube/config"
    config_context = "somecontext"
  }
}

provider "kubernetes" {
  host           = var.cluster_endpoint
  config_path    = "~/.kube/config"
  config_context = "somecontext"
}

module "flux_operator_bootstrap" {
  source  = "controlplaneio-fluxcd/flux-operator-bootstrap/kubernetes"
  version = "0.8.0"

  revision = var.bootstrap_revision

  common_metadata = {
    labels = {
      "pod-security.kubernetes.io/enforce" = "privileged"
      "pod-security.kubernetes.io/audit"   = "privileged"
      "pod-security.kubernetes.io/warn"    = "privileged"
    }
  }

  job = {
    host_network = true
    tolerations = [{
      key      = "node.kubernetes.io/not-ready"
      operator = "Exists"
      effect   = "NoSchedule"
    }]
    env = {
      KUBERNETES_SERVICE_HOST       = "192.168.20.60"
      KUBERNETES_SERVICE_PORT       = "6443"
      KUBERNETES_SERVICE_PORT_HTTPS = "6443"
    }
  }

  gitops_resources = {
    instance_yaml = file("${path.root}/../clusters/${var.cluster_name}/flux-instance.yaml")
    operator_chart = {
      values_yaml = yamlencode({
        tolerations = [{
          key      = "node.kubernetes.io/not-ready"
          operator = "Exists"
          effect   = "NoSchedule"
        }]
        hostNetwork = true
        extraArgs = [
          "--metrics-addr=:10001"
        ]
        extraEnvs = [{
          name  = "KUBERNETES_SERVICE_HOST"
          value = "192.168.20.60"
          }, {
          name  = "KUBERNETES_SERVICE_PORT"
          value = "6443"
          }, {
          name  = "KUBERNETES_SERVICE_PORT_HTTPS"
          value = "6443"
        }]
      })
    }
  }
}
```

### `flux-instance.yaml`
```yaml
apiVersion: fluxcd.controlplane.io/v1
kind: FluxInstance
metadata:
  name: flux
  namespace: flux-system
spec:
  distribution:
    version: "2.9.x"
    registry: "ghcr.io/fluxcd"
  components:
    - source-controller
    - kustomize-controller
    - helm-controller
  cluster:
    type: kubernetes
    size: small
  sync:
    kind: GitRepository
    url: my-git-address.git
    ref: "refs/heads/testing-flux"
    path: "gitops/clusters/test"
  kustomize:
    patches:
      - target:
          kind: Deployment
        patch: |
          - op: add
            path: /spec/template/spec/tolerations
            value:
              - key: "node.kubernetes.io/not-ready"
                operator: "Exists"
                effect: "NoSchedule"
          - op: add
            path: /spec/template/spec/hostNetwork
            value: true
```

## What works
- The Flux operator bootstrap pod runs in the host network and tries to deploy the resources. It also respects the overridden variables with the k8s API endpoint
- The Flux controllers run in the host network

## What does not work
- The Flux operator ends up not being scheduled because of a port conflict in the host network namespace. It looks like port 8080 (a scrape endpoint) is the culprit, since it's exposed on both the operator and Flux controllers. Changing the metrics port with the extra arg `--metrics-addr` in the Helm chart does not seem to help.
  ```
  Events:
    Type     Reason            Age                  From               Message
    ----     ------            ----                 ----               -------
    Warning  FailedScheduling  28m                  default-scheduler  0/3 nodes are available: 3 node(s) didn't have free ports for the requested pod ports. no new claims to deallocate, preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod.
    Warning  FailedScheduling  8m38s (x5 over 28m)  default-scheduler  0/3 nodes are available: 3 node(s) didn't have free ports for the requested pod ports. no new claims to deallocate, preemption: 0/3 nodes are available: 3 No preemption victims found for incoming pod.
  ```
- The scheduling failure above is the reason why the Flux operator bootstrap pod eventually exits with the error: `timed out waiting for flux-system/flux to become ready`
- The Flux controllers try to contact the API server over the ClusterIP despite the overriden k8s API endpoint in the Helm chart

## Conclusion
While following the deployment order that would seem to be logical isn't gonna working here (Flux in host network -> CNI -> rest), it's still interesting how far you can get when running the regular Flux deployment on a CNI-less cluster.

### Alternatives
1. The Terraform bootstrap job offers `gitops_resources.prerequisites.charts`:
> list of Helm charts to install before Flux; useful for components that must exist before Flux can bootstrap (e.g. CSI drivers, CNI plugins)

2. The Flux AIO[^3] is a compact deployment that is optimized for running on clusters without a CNI plugin installed.
All its components are packed into a single pod.

## References
[^1]: [Flux Operator Bootstrap](https://github.com/controlplaneio-fluxcd/terraform-kubernetes-flux-operator-bootstrap)
[^2]: [Flux Operator Helm Chart](https://fluxoperator.dev/docs/charts/flux-operator/)
[^3]: [Flux AIO](https://github.com/stefanprodan/flux-aio)