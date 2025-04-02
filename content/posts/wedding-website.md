---
date: '2025-02-18T10:48:15+01:00'
draft: false
title: 'A wedding website powered by Kubernetes, Flux, and GitHub Actions - Part 1'
tags: ["kubernetes", "gitops", "flux", "k8s", "terraform", "ansible", "iac"]
slug: 'wedding-pt1'
---

{{< admonition warning "yet another warning, deal with it" >}}
This is not intended as a tutorial for setting up an environment like this, as some things are automated and some other things (like HAProxy setup) are carried out manually. This is still an incomplete mess.

It's meant to be something like a blueprint you can take inspiration from. You will not be able to find detailed commands or detailed code, as I'm way too ashamed of my git history to make my repos public atm. I swear I'll fix them in the near future.
{{< /admonition >}}

Ok, **I'm getting married**. As an IT guy, my first priority has to be:
> How can we deploy a wedding website no one is ever going to take a look at?

Let's start then.

# Domain
Registering a domain is always the first step. Miriana is my future wife's name, also known as *Miri*. My surname is Martino, also known as *tino*. Here you go, `miritino.it`.
That domain is then managed on Cloudflare and the web server is going to be exposed via [Cloudflare Tunnels](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/), as I'm using that for managing my other domains (yes, more than one, included `martino.wtf` which I'm super proud of).

# Kubernetes on Proxmox VMs
Everything is running on a very small Kubernetes cluster hosted on an even smaller 3-node Proxmox cluster. This Kubernetes cluster is going to be used to host:
- management services: Observability stack (Grafana, VictoriaLogs, VictoriaMetrics) and 1Password Connect server. Maybe I will add a nice IdP in the future (Keycloak), who knows?
- applications: at this stage, only the wedding website

The VMs are created from an already existing Ubuntu 24.04 LTS template (nothing fancy, IIRC only `cloud-init` and `qemu agent` have been installed on this) using some terraform plans (courtesy of [proxmox terraform provider](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs)). 

{{< admonition info "Terraform Modules" >}}
I will probably rewrite these plans as [modules](https://developer.hashicorp.com/terraform/language/modules) just to learn something new, but that's another task for future me.
{{< /admonition >}}

I've built a nice wrapper around this provider so that the VMs are declared in the following way:

```json
vms = {
  k8s-cp-01 = {
    cpu = 2
    disk_size = "30"
    disk_location = "local-lvm"
    template = "ubuntu-2404-template"
    ip = "192.168.20.1",
    ram = 4096, 
    node = "nuc",
    user = "ci",
    onboot = false,
    network_zone = "k8s",
    vm_state = "running"
    tags = "k8s;k8s_cps"
    hostname = "cp-01"
  }, 
  k8s-wrk-01 = {
    cpu = 4
    disk_size = "100"
    disk_location = "local-lvm"
    template = "ubuntu-2404-template"
    ip = "192.168.20.11"
    ram = 16384, 
    node = "chinuc",
    user = "ci",
    onboot = false,
    network_zone = "k8s"
    vm_state = "running"
    tags = "k8s;k8s_workers"
    hostname = "wrk-01"
  }
}
```

The VM is created, with the correct network interfaces with the correct VLAN tagging. The correct network configuration and the SSH public keys are applied then via `cloud-init` ([HERE](https://pve.proxmox.com/wiki/Cloud-Init_Support) and [HERE](https://cloudinit.readthedocs.io/en/latest/index.html)). On top of that, relevant DNS entries/firewall aliases are created in my opnsense installation (courtesy of [opnsense terraform provider](https://registry.terraform.io/providers/browningluke/opnsense/latest/docs)).

![VM deployed on Proxmox](/images/wedding-website/proxmox.jpg)

![opnsense kubernetes dedicated sub-CA](/images/wedding-website/dns.png)

Then a magic Ansible playground is executed, which will:
- install some useful tools (i.e. `netools`)
- install the container runtime (`cri-o`, in my case)
- install `kubeadm` and `kubelet`
- do some preparation stuff on the OS
- create the cluster, bootstrapping the controlplane
- join the worker node to the cluster
- install Cilium CNI

I'm exposing the API endpoint behind a VIP, so I'm using a specific `kubeadm` command to spin up the cluster, specifying the control plane endpoint.

```bash
kubeadm init --control-plane-endpoint={{ control_plane_endpoint }} --apiserver-bind-port={{ apiserver_bind_port }} --cri-socket={{ cri_socket }} --pod-network-cidr={{ pod_network_cidr }} --ignore-preflight-errors=FileContent--proc-sys-net-bridge-bridge-nf-call-iptables --skip-phases=addon/kube-proxy --upload-certs
```

The relevant Ansible variable is `control_plane_endpoint`.

```yaml
control_plane_endpoint: api.k8s.cicci8ino.it
```
{{< admonition info "iptables" >}}
As you can see, I'm ignoring the `iptables` warning because I won't be using `iptables` rules to reach cluster services, as `kube-proxy` is being replaced by the [Cilium kube-proxy replacement](https://docs.cilium.io/en/stable/network/kubernetes/kubeproxy-free/).
{{< /admonition >}}


Behind that, an HAProxy VIP is going to load balance to the real control plane servers behind the scenes (with a dumb TCP health check that I should replace with a proper https [health check endpoint](https://kubernetes.io/docs/reference/using-api/health-checks/#api-endpoints-for-health)). This will make things easier when I decide (probably never) to add new master nodes.
{{< admonition note "VIP" >}}
If you don't want to rely on another appliance/service for VIP and API /HA, you can take a look at [kube-vip](https://kube-vip.io) or [keepalived](https://www.keepalived.org)
{{< /admonition >}}


The cluster is now ready. Now things start to get interesting.

# Cluster Setup
Let's set up Odyssey (have you ever seen Apollo 13?), my small Kubernetes cluster.

## CNI

A Kubernetes cluster needs a [Container Network Interface](https://www.cni.dev), aka CNI. I will use [Cilium](https://cilium.io). This is installed by the same Ansible playground, simply applying the official `helm` chart.

```yaml
- name: Add cilium helm
  kubernetes.core.helm_repository:
    name: cilium
    repo_url: https://helm.cilium.io/
    force_update: true

- name: Deploy cilium helm chart
  kubernetes.core.helm:
    name: cilium
    chart_ref: cilium/cilium
    release_namespace: "{{ cilium_namespace }}"
    chart_version: "{{ cilium_version }}"
    kubeconfig: "{{ kubeconfig_dest }}"
    values:
      kubeProxyReplacement: True
      k8sServiceHost: "{{ control_plane_endpoint }}"
      k8sServicePort: "{{ apiserver_bind_port }}"
```

## Bootstrap

Cluster is now ready to get some real love. So a nice [`Taskfile`](https://taskfile.dev) will make sure that:
- [`cert-manager`](https://cert-manager.io) namespace is created. This is going to be used to create certificates from the Kubernetes sub-CA I've already created on my `opnsense` installation

![opnsense kubernetes dedicated sub-CA](/images/wedding-website/subca.png)
That sub-CA was generated from my opnsense root CA.

- The sub-CA certificate and the private key are added in `cert-manager` namespace as secret, fetched from my 1Password `homelab` safe with `op` CLI
```bash
op run --env-file="./.op.env" -- <command>
```
- [`1Password connect`](https://developer.1password.com/docs/connect/), the needed 1Password op-credentials and [`external-secrets-operator`](https://external-secrets.io/latest/) are installed. 1Password connect is going to be exposed via the `nginx` Ingress Controller when installed in the next steps.
> I've decided to rely on `external-secrets-operator` (aka `ESO`) even though 1Password already has something similar ([1Password K8s operator](https://developer.1password.com/docs/k8s/k8s-operator/)) because it was much easier for it to trust a custom certificate generated by `cert-manager` signed by the cluster sub-CA.

Here you can see an extract of the tasks.
```yaml
  deploy-1password-connect-secret:
    desc: "Create 1password namespace and deploy the connect secret"
    cmds:
      - kubectl create ns 1password || true
      - kubectl delete secret op-credentials --namespace 1password || true
      - |
        kubectl create secret generic op-credentials \
          --namespace 1password \
          --from-literal=1password-credentials.json={{.ENCODED_OP_CREDENTIALS}} \
          --type=Opaque
```
Where some environment variables are populated by the OP CLI.
```bash
export OP_CREDENTIALS=op://Homelab/1p-homelab-credentials/1password-credentials.json
```
[HERE](https://developer.1password.com/docs/cli/secrets-environment-variables/) some additional details.

## Flux

Then:
- [Flux Operator](https://fluxcd.control-plane.io/operator/) is installed from the official helm chart
```yaml
  flux-operator-bootstrap:
    desc: "Bootstrap Flux Operator"
    cmds:
      - |
        helm install flux-operator oci://ghcr.io/controlplaneio-fluxcd/charts/flux-operator \
        --namespace flux-system \
        --create-namespace
```
- A `flux secret` is created to the git repo containing all the manifests that flux is going to reconcile
- a [`FluxInstance`](https://fluxcd.control-plane.io/operator/fluxinstance/) is created by the Flux Operator using a `kustomize` overlay
```yaml
  deploy-flux-instance:
    desc: "Build flux instance kustomize overlay and deploy"
    cmds: 
      - kubectl apply --kustomize ./kustomize/overlays/{{.CLI_ARGS}}
```
```yaml
resources:
  - ../../base

patches:
  - patch: |-
      - op: add
        path: /spec/sync/path
        value: "clusters/odyssey"
      - op: add
        path: /spec/sharding
        value:
          shards:
            - mgmt
            - infra
      - op: add
        path: /spec/cluster/tenantDefaultServiceAccount
        value: flux
    target:
      kind: FluxInstance
      name: flux
      namespace: flux-system
```
The `kustomize` patch will mainly enable [sharding](https://fluxcd.io/flux/installation/configuration/sharding/) (not that I really needed it, considering how small my cluster is, but I wanted to have a quick look at it).

# Recap
At this stage, Odyssey is ready to reconcile stuff. It now has:
- `1password connect` server, that can be reached by any other future Kubernetes cluster or any other service that needs credentials from my 1Password dedicated safe.
- `cert-manager`, that will manage ingress certificate generation from now on.

{{< admonition tip "chicken-and-egg" >}}
There's a nice chicken-and-egg scenario here, as Flux is installed after several objects have been created "manually". So how can Flux reconcile these objects? I would suggest you have a look at the wonderful blog post from my colleague Fabian [HERE](https://blog.kammel.dev/post/k8s_home_lab_2025_01/).
{{< /admonition >}}

In the next episode, we are going to take a look at how I've structured my GitOps repository and how I'm deploying the website in the cluster.