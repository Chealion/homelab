# Homelab

## Cluster Setup

* [Talos](https://talos.dev) cluster configured with [talhelper](https://github.com/budimanjojo/talhelper) - separate repository.

## Argo CD

Managed via two app-of-apps entrypoints: [`argocd_infra_applications.yaml`](./argocd/argocd_infra_applications.yaml) (Infra) and [`argocd_applications.yaml`](./argocd/argocd_applications.yaml) (Applications). Values are stored in [`argocd/values/`](./argocd/values/), and manifests/objects in [`argocd/objects/`](./argocd/objects/).

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

* [ClickStack](https://github.com/ClickHouse/ClickStack)
* [Headlamp](https://github.com/headlamp/headlamp)
* [OpenTelemetry Collector](https://opentelemetry.io/docs/collector/)
* [Readeck](https://readeck.github.io/)
* [SearXNG](https://github.com/searxng/searxng)
* [Unpoller](https://unpoller.app/)

## Resources and Inspiration
* https://github.com/joeypiccola/k8s_home

## Operations & Troubleshooting

* **1Password API Limits:** [1Password limits personal accounts to 1000 API calls in 24 hours](https://www.1password.dev/service-accounts/rate-limits) - hence why `OnChange` for all ESO objects. An automated check for changes for a single item once a minute is 1440 calls in a day.
* **Force Sync External Secrets:** Run the following to manually force a sync:
  ```bash
  kubectl annotate es <name> force-sync=$(date +%s) --overwrite -n <namespace>
  ```

## Credential Rotation
* Argo CD - OIDC: OIDC creds from Authentik - restart `argocd-dex-server`
* Authentik - PostgreSQL: DB access - restart `authentik`
* Cert Manager - CloudFlare: API key
* ClickStack - ClickHouse: Password is `clickhouse-app-password`. After rotation, the connection password must be manually updated in the HyperDX UI (Settings -> Connections).
* ClickStack - MongoDB: Rotates when secret is updated. Also need to restart `clickstack-app` (connection strings).
* ExternalDNS - CloudFlare: API key
* Grafana - Admin: Default creds. Restart `grafana`
* Grafana - OAUTH/OIDC: OIDC creds from Authentik - restart `grafana`
* Grafana - ClickHouse data source: Access creds. Restart `grafana`.
* Headlamp - OIDC: OIDC creds from Authentik - restart `headlamp`
* Otel Collector - ClickStack Ingest: API key from HyperDX. Restart `otel-k8s-collector`
* Renovate - Token: GitHub PAT to access GitHub
* SearXNG - Encryption secret. Restart `searxng`
* Tailscale Operator - Oauth: Tailscale API key. Restart `tailscale-operator`
* Unpoller - Secret: Creds for UniFi. Restart `unpoller`
