# Homelab

## Cluster Setup

* [Talos](https://talos.dev) cluster configured with [talhelper](https://github.com/budimanjojo/talhelper)

## Argo CD

Two app of apps to create all the applications in Argo CD. Values are stored in one directory, and objects in a separate directory.

### Infra

* [Argo CD](https://argoproj.github.io/cd/)
* [Cilium](https://cilium.io)
* [External Secrets Operator](https://external-secrets.io)
* [Kube Prometheus Stack](https://github.com/prometheus-community/helm-charts/tree/main)
* [Local Path Provisioner](https://github.com/rancher/local-path-provisioner)
* [Tailscale](https://tailscale.com/kb/1236/kubernetes-operator#helm)

### Applications

* 

## Resources and Inspiration
* https://github.com/joeypiccola/k8s_home

## Annoyances

* 1Password API limits
* `kubectl annotate es <name> force-sync=$(date +%s) --overwrite -n <namespace>`
