---
date: '2025-04-02T14:48:15+01:00'
draft: false
title: 'A wedding website powered by Kubernetes, Flux, and GitHub Actions - Part 2'
tags: ["kubernetes", "gitops", "flux", "k8s", "terraform", "ansible", "iac"]
slug: 'wedding-pt2'
---

{{< admonition warning "yet another warning, deal with it" >}}
This is not intended as a tutorial for setting up an environment like this, as some things are automated and some other things (like HAProxy setup) are carried out manually. This is still an incomplete mess.

It's meant to be something like a blueprint you can take inspiration from. You will not be able to find detailed commands or detailed code, as I'm way too ashamed of my git history to make my repos public atm. I swear I'll fix them in the near future.
{{< /admonition >}}

In the previous episode, I've tried to gave you an idea on how I've been setting up my environment to host a very simple wedding website.

# GitOps

The repo containing all the manifests is loosely based on the [Flux D1 Architectural Reference](https://fluxcd.control-plane.io/guides/d1-architecture-reference/), but made simpler without the multitenancy stuff because that would be way overkill for my setup.

This is a multistage reconciliation, managed by different Kustomization CRs. In order:

## cluster-vars
```yaml
---
apiVersion: v1
kind: ConfigMap
data:
  cluster_name: odyssey
  dns_ip: 192.168.20.66
  ingress_ip: 192.168.20.67
  domain: k8s.cicci8ino.it
  mgmt_domain: k8s.cicci8ino.it
  load_balancer_pool: 192.168.20.65-192.168.20.126
  storage_replica: "1"
metadata:
  name: cluster-vars
  namespace: flux-system
```
These are the information for this specific cluster. Each key in the `ConfigMap` should be self-explanatory. If I ever decide to create another cluster, I have to make sure to properly configure these variables.

## common-configs-reconciliation
This will only build the manifests from the `./common/config` folder, as specified in `.spec.path.`
```yaml
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: common-configs-reconciliation
  namespace: flux-system
  labels:
    sharding.fluxcd.io/key: infra
spec:
  commonMetadata:
    labels:
      toolkit.fluxcd.io/tenant: common
      sharding.fluxcd.io/key: infra
  interval: 1h
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./common/configs
  prune: false
```
The `ConfigMap` will only bring some common variables that are going to be shared across many Flux CRs. 

{{< admonition tip "Post Variable Substitution" >}}
Have a look at the [official Flux doc](https://fluxcd.io/flux/components/kustomize/kustomizations/#post-build-variable-substitution) to understand how to use post variable substitution.
{{< /admonition >}}

```yaml
apiVersion: v1
kind: ConfigMap
data:
  mgmt_domain: k8s.cicci8ino.it
metadata:
  name: common-configs-vars
  namespace: flux-system
```
This `ConfigMap` is mainly used to share the cluster domain for ingress registration. As an example, the wedding website `Ingress` uses a template on the `spec.rules[].host` field that is substituted in the final YAML manifests before being applied by Flux.
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wedding-website-ingress
  namespace: wedding-website
  labels:
    app: wedding-website
spec:
  rules:
  ...
  - host: wedding.${domain}
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: wedding-website-svc
            port:
              name: web
```
## network-reconciliation
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: network-reconciliation
  namespace: flux-system
  labels:
    sharding.fluxcd.io/key: infra
spec:
  commonMetadata:
    labels:
      toolkit.fluxcd.io/tenant: network
      sharding.fluxcd.io/key: infra
  interval: 1h
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./common/network
  prune: false
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: cluster-vars
```
This kickstart the reconciliation of network relevant manifests. To be more specific, this will use kustomize to build some manifests:
```yaml
kustomization.yaml

---
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: flux-system
resources:
  - nginx/nginx.yaml
  - metallb/metallb.yaml
  - k8s-gateway/k8s-gateway.yaml
  - cert-manager/cert-manager.yaml
  - trust-manager/trust-manager.yaml
  - cilium/cilium.yaml
```
As every other component in the cluster, each of the network pieces will require the reconciliation of two subcomponents:
- `<component>-controllers`: this can include `HelmRelease`, `HelmRepository` and `Namespace` objects, everything that will deploy the required workload.
- `<component>-configs`: this can include CRs or `ConfigMaps` used by the component.

As an example, let's see how `MetalLB` is installed.

{{< admonition tip "MetalLB" >}}
MetalLB is a nice [LoadBalancer](https://kubernetes.io/docs/concepts/services-networking/service/#loadbalancer) implementation meant for bare metal Kubernetes installation, where you can't leverage cloud provided load balancer. Have a look at the official doc [HERE](https://metallb.io). This will basically create a Virtual IP where your workload (i.e. IngressController) will be listening on, using either L2 ([Gratuitous ARP](https://wiki.wireshark.org/Gratuitous_ARP)) or L3 ([BGP](https://en.wikipedia.org/wiki/Border_Gateway_Protocol))
{{< /admonition >}}
```yaml
metallb.yaml

---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: metallb-controllers
  namespace: flux-system
spec:
  interval: 1h
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./common/network/metallb/controllers
  prune: true
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: metallb-configs
  namespace: flux-system
spec:
  dependsOn:
    - name: metallb-controllers
  interval: 1h
  retryInterval: 1m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./common/network/metallb/configs
  prune: false
  postBuild:
    substituteFrom:
      - kind: ConfigMap
        name: cluster-vars
```
Flux first install the `metallb-controllers` and then the `metallb-configs`, as you can see from the `.spec.dependsOn` field (more info available [HERE](https://fluxcd.io/flux/components/kustomize/kustomizations/#dependencies)).
`metallb-controllers` is responsible to deploy:
- `metallb` Namespace
- `HelmRepository` CR, for the MetalLB helm repository;
- `HelmRelease` CR, to install MetalLB helm chart from the MetalLB helm repository

`metallb-configs` is responsible to deploy the `IPAddressPool` for MetalLB (which CIDR is going to be used to instantiate a new VIP, when a new Service with `spec.type: LoadBalancer` is created) and the `L2Advertisment` CR (to setup MetalLB in L2 mode).

Folder structure will look something like this:
```
└── common
    ├── configs
    ├── infra
    └── network
        ├── cert-manager
        ├── cilium
        ├── k8s-gateway
        ├── kustomization.yaml
        ├── metallb
        │   ├── configs
        │   │   ├── ipaddresspool.yaml
        │   │   └── l2advertisement.yaml
        │   ├── controllers
        │   │   ├── helmrelease.yaml
        │   │   ├── helmrepository.yaml
        │   │   └── namespace.yaml
        │   └── metallb.yaml
        ├── nginx
        └── trust-manager
```
Have a look on how the post variable substitution is used for MetalLB too.
```yaml
ipaddresspool.yaml

---
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: standard
  namespace: metallb
spec:
  addresses:
  - ${load_balancer_pool}
```
`${load_balancer_pool}` is going to be replaced with `cluster-vars.data.cluster-vars`.

{{< admonition tip "Is MetalLB really needed?" >}}
If you are using Cilium, you may want to use [LB IPAM](https://docs.cilium.io/en/stable/network/lb-ipam/). To replicate this L2 setup, you would need to set `l2announcements.enabled=true` during the helm install, as documented [HERE](https://docs.cilium.io/en/stable/network/l2-announcements/#l2-announcements-l2-aware-lb-beta).
{{< /admonition >}}

This reconciliation will also be responsible of installing:
- [Cilium](https://cilium.io): the CNI I'm using in the cluster, powered by eBPF. Do you remember the chicken-in-the-egg thing?
- [NGINX Ingress Controller](https://docs.nginx.com/nginx-ingress-controller/installation/installing-nic/installation-with-helm/): [Ingress Controller](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/) listening on a Service LoadBalancer. And no, that's not the "default" community based Kubernetes Ingress Controller. This is maintained by nginx/F5
- [cert-manager](https://cert-manager.io): generate SSL certificates from the cluster sub-CA I've talked about in part 1. You can use it to automatically generate SSL certificates with [Let's Encrypt](https://cert-manager.io/docs/tutorials/acme/nginx-ingress/) if you don't want to manage your own CA
- [trust-manager](https://cert-manager.io/docs/trust/trust-manager/): automatically replicate my homelab root CA certificate across all the namespaces, so it will be easier for any workload to trust it and to connect to internal services
- [k8s-gateway](https://github.com/ori-edge/k8s_gateway): expose a DNS service and automatically register any entries defined as `Ingress` or `Gateway` resources
## secret-reconciliation
This Kustomization depends on the `network-reconciliation` `Kustomization` and is responsible to deploy the 1Password Connect server. This will connect to my 1Password Homelab safe, using the secrets deployed in part 1.
## infra-reconciliation
This Kustomization will install many components, such as:
- [External Secret Operator](https://external-secrets.io/latest/): automatically sync secrets with external secret manager solution (in my case 1Password through the 1Password Connect server)
- [Promtail](https://grafana.com/docs/loki/latest/send-data/promtail/): collect logs and send it to an external logging solution
- [kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack): collect metrics and send it to an external metrics collection solution and make them available for [FreeLens](https://github.com/freelensapp/freelens)
- [Longhorn](https://longhorn.io): cloud native distributed block storage (even if I'm only using a two node cluster, where only the worker node will need persistent storage)
# Management cluster
As shared in part 1, this cluster is used to expose some management services too, on top of the 1Password Connect Server previously installed. These are managed by a specific reconciliation step.
## log-reconciliation
This Kustomization will wait for `infra-reconciliation` to be completed. This will install:
- [VictoriaMetrics](https://victoriametrics.com): collect metrics, alternative to [Prometheus](https://prometheus.io). Receive logs from `kube-prometheus-stack`
- [VictoriaLogs](https://docs.victoriametrics.com/victorialogs/): collect logs, alternative to [Loki](https://grafana.com/oss/loki/). Receive logs from Promtail
- [Grafana](https://grafana.com): show logs and metrics in nice dashboards

# Copy and paste everywhere?
As you may have seen from the repository structure, there's some kind of pattern that is always repeated across all the components. Each component is usually shipped with a `configs` and a `controllers` `Kustomization`. The controllers usually deploys an `HelmRepository`/`OCIRepository`, an `HelmRelease` and a `Namespace` objects.

Wouldn't it be nice to be able to create something like a template, to avoid copying and pasting resources all over again? That's what `ResourceSet` API is meant for. With `ResourceSet` you can generate multiple Kubernetes objects following a matrix of input values and a template. This is part of the new Flux Operator and you can find more details [HERE](https://fluxcd.control-plane.io/operator/resourceset/).

{{< admonition tip "Templates vs inputs" >}}
I'm personally still have to use it as I would like templates and inputs to be defined in separate objects. I've proposed to decouple the two concepts and an issue has already been opened [HERE](https://github.com/controlplaneio-fluxcd/flux-operator/issues/203) to track progresses. In the meantime, I was considering start using [Kube Resouce Orchestrator (KRO)](https://kro.run) but didn't have the time yet.
{{< /admonition >}}

In the next episode, I'm going to talk about how the website is packaged, uploaded on GHCR via a GitHub action and how this is installed on the cluster by Flux.