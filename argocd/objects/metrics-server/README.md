Upstream Docs: https://www.talos.dev/v1.11/kubernetes-guides/configuration/deploy-metrics-server/

A local version of the two YAML files needed to enable the metrics server.

## Standalone Kubelet Cert Approver
https://github.com/alex1989hu/kubelet-serving-cert-approver

```
wget https://raw.githubusercontent.com/alex1989hu/kubelet-serving-cert-approver/v0.10.0/deploy/standalone-install.yaml -O kubelet-cert.yaml
```

Current version is the same as the 0.10.0 release from Nov. 22, 2025

## Kubernetes Metrics Server
https://github.com/kubernetes-sigs/metrics-server

```
wget https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml -O metrics-server.yaml
```

Current version is the same as the 0.8.0 release from July 3, 2025
