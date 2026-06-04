# Homelab

## Cluster Setup

* [Talos](https://talos.dev) cluster configured with [talhelper](https://github.com/budimanjojo/talhelper)

## Argo CD

Two app of apps to create all the applications in Argo CD. Values are stored in one directory, and objects in a separate directory.

### Infra

* [Argo CD](https://argoproj.github.io/cd/)
* [Authentik](https://goauthentik.io/)
* [cert-manager](https://cert-manager.io/)
* [Cilium](https://cilium.io)
* [CloudNativePG](https://cloudnative-pg.io/)
* [ExternalDNS](https://kubernetes-sigs.github.io/external-dns/)
* [External Secrets Operator](https://external-secrets.io)
* [Kube Prometheus Stack](https://github.com/prometheus-community/helm-charts/tree/main)
* [Local Path Provisioner](https://github.com/rancher/local-path-provisioner)
* [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
* [Renovate](https://docs.renovatebot.com/)
* [Tailscale](https://tailscale.com/kb/1236/kubernetes-operator#helm)

### Applications

* [Headlamp](https://github.com/headlamp/headlamp)
* [Readeck](https://readeck.github.io/)
* [Unpoller](https://unpoller.app/)

## Resources and Inspiration
* https://github.com/joeypiccola/k8s_home

## Annoyances

* 1Password API limits
* `kubectl annotate es <name> force-sync=$(date +%s) --overwrite -n <namespace>`
