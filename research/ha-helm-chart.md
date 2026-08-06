# Research: Home Assistant Helm Chart Landscape

**Ticket:** #152 — HA Helm chart landscape
**Date:** 2026-08-06
**Repo:** Chealion/homelab

## Summary

Home Assistant does **not** ship an official Helm chart. The home-assistant GitHub org (106 public repositories) contains no helm-charts repository, and the official installation documentation lists no Kubernetes/Helm path — only Home Assistant OS and Home Assistant Container. Per the homelab's chart policy (official chart first → bjw-s `app-template` fallback), the recommendation is to use **bjw-s `app-template` v5.0.1** from `https://bjw-s-labs.github.io/helm-charts` wrapping the official image **`ghcr.io/home-assistant/home-assistant:stable`** (version-pinned with a Renovate annotation, matching the existing repo pattern). This is consistent with how readeck, searxng, and unpoller are already deployed in this homelab.

---

## Findings

### 1. No Official Helm Chart Exists

The home-assistant GitHub organization has **106 public repositories** — including `core`, `supervisor`, `operating-system`, `addons`, `frontend`, `iOS`, `android`, and others — but **no helm-charts or Kubernetes chart repository**. [Source: github.com/home-assistant](https://github.com/home-assistant)

The official HA installation page lists only two installation types:
- **Home Assistant Operating System** (recommended for most users; runs on bare metal / VM / Raspberry Pi)
- **Home Assistant Container** (Docker/OCI container; bring-your-own orchestration)

There is no mention of Kubernetes, Helm, or any chart-based deployment in the official docs. [Source: home-assistant.io/installation](https://www.home-assistant.io/installation/)

**Conclusion:** No official Helm chart exists. The homelab policy's fallback to bjw-s `app-template` applies.

### 2. The k8s-at-home Chart Lineage (Deprecated/Archived)

The former community standard was the `k8s-at-home/charts` repository, which hosted a `home-assistant` chart built on the `common` library chart. This repository was **deprecated and archived on August 22, 2022**. [Source: k8s-at-home/charts issue #1761](https://github.com/k8s-at-home/charts/issues/1761)

The deprecation announcement by @onedr0p stated:
> "The charts repo is not going away but we are no longer going to be accepting PRs or triaging issues. We will continue to develop and support the library chart but at a new home."

The library chart moved to `bjw-s/helm-charts` (now `bjw-s-labs/helm-charts`). Maintainers recommended using the `app-template` chart for standard apps. [Source: k8s-at-home/charts](https://github.com/k8s-at-home/charts)

The even older `helm/charts` stable/home-assistant chart (the original Helm stable repo) was archived on **February 22, 2022** and is also defunct. [Source: github.com/helm/charts/tree/master/stable/home-assistant](https://github.com/helm/charts/tree/master/stable/home-assistant)

### 3. bjw-s `app-template` — The Designated Fallback (Recommended)

**Repository:** [bjw-s-labs/helm-charts](https://github.com/bjw-s-labs/helm-charts)
**Chart:** `app-template`
**Current version:** `5.0.1` (released 2026-05-14)
**Helm repo URL:** `https://bjw-s-labs.github.io/helm-charts`

This is a generic application chart (powered by the `common` library chart) that can deploy any containerized application. It is the direct successor to the k8s-at-home `common` library chart and is maintained by @bjw-s (Bernd Schönböck), one of the original k8s-at-home maintainers.

**Official HA example:** The bjw-s docs include a [Home Assistant with code-server example](https://bjw-s-labs.github.io/helm-charts/docs/app-template/examples/home-assistant-codeserver/) demonstrating a StatefulSet deployment with `/config` persistence, ingress, and an optional code-server sidecar.

**Important caveat:** The bjw-s example uses `ghcr.io/onedr0p/home-assistant` — a community-maintained image fork by @onedr0p, **not** the official HA image. The homelab should use the official `ghcr.io/home-assistant/home-assistant` image instead (see §5 below).

**Homelab consistency:** This homelab already uses `app-template` v5.0.1 from `https://bjw-s-labs.github.io/helm-charts` for:
- **readeck** — `argocd/apps/readeck.yaml`, values in `argocd/values/readeck.yaml`
- **searxng** — `argocd/apps/searxng.yaml`, values in `argocd/values/searxng.yaml`
- **unpoller** — `argocd/apps/unpoller.yaml`, values in `argocd/values/unpoller.yaml`

All three follow the same Argo CD Application pattern: `chart: app-template`, `repoURL: https://bjw-s-labs.github.io/helm-charts`, `targetRevision: 5.0.1`, with values in `argocd/values/<app>.yaml` and objects in `argocd/objects/<app>/`. Using app-template for Home Assistant maintains this consistency.

### 4. Reputable Community Charts (Alternatives, Not Recommended per Policy)

The homelab policy designates bjw-s `app-template` as the fallback when no official chart exists. The charts below are community-maintained alternatives that could be considered if app-template proves unsuitable for HA's specific needs, but they are **not** the policy-preferred choice.

#### pajikos/home-assistant-helm-chart
- **Repo:** [github.com/pajikos/home-assistant-helm-chart](https://github.com/pajikos/home-assistant-helm-chart)
- **Stars:** ~333 | **License:** MIT | **Created:** 2023-04-29
- **Helm repo:** `http://pajikos.github.io/home-assistant-helm-chart/`
- **Default image:** `ghcr.io/home-assistant/home-assistant` (the official image)
- **Features:** Auto-updates chart AppVersion with each HA release; supports StatefulSet or Deployment; hostNetwork; hostPort; PVC persistence; ingress; Gateway API HTTPRoute; code-server addon; serviceMonitor; init containers
- **Status:** Actively maintained. The most feature-complete dedicated HA chart.
- **Note:** This is the strongest community alternative if a purpose-built HA chart is ever preferred over app-template. It supports hostNetwork and hostPort natively, which may be relevant for HA's networking requirements (see §6).

#### gabe565/home-assistant
- **Repo:** [charts.gabe565.com](https://charts.gabe565.com/charts/home-assistant/)
- **Dependency:** Uses `bjw-s/common` as a library (similar lineage to app-template but a dedicated HA chart wrapper)
- **Status:** Maintained, but explicitly notes "not maintained by upstream project"

#### Sironite/helm-home-assistant
- **Repo:** [github.com/Sironite/helm-home-assistant](https://github.com/Sironite/helm-home-assistant)
- **Features:** StatefulSet, code-server, Authentik outpost, External Secrets Operator integration
- **Status:** Actively maintained (v3.0.1)
- **Note:** More opinionated/integrated than app-template; ties into specific infrastructure choices

#### Wrenix/helm-charts (home-assistant)
- **Repo:** [codeberg.org/wrenix/helm-charts](https://wrenix.eu/docs/helm-charts/home-assistant/) (OCI)
- **Status:** Maintained, k3s-focused

#### andrenarchy/helm-charts (home-assistant)
- **Repo:** [github.com/andrenarchy/helm-charts](https://github.com/andrenarchy/helm-charts/tree/main/charts/home-assistant)
- **Status:** Based on old k8s-at-home chart; less actively maintained

### 5. Container Image

Regardless of chart choice, the base container image is the same:

| Property | Value |
|---|---|
| **Official image** | `ghcr.io/home-assistant/home-assistant` |
| **Registry** | GitHub Container Registry (ghcr.io) |
| **`stable` tag** | Most recent stable release (recommended by HA docs) |
| **`beta` tag** | Most recent beta release |
| **Version-pinned tags** | e.g., `2026.8.0`, `2026.4.4` — exact HA Core release versions |
| **`latest` tag** | Most recent build (may lag behind `stable` in practice; community reports of tag lag) |
| **Architectures** | `amd64`, `aarch64` (armhf, i386, armv7 removed per ADR-0013 changelog, 2026-04-02) |

**Recommendation for this homelab:** Use `ghcr.io/home-assistant/home-assistant` with a **version-pinned tag** and a Renovate annotation, matching the pattern used by readeck (`tag: 0.22.3` with `# renovate: datasource=docker ...`) and unpoller (`tag: v3.3.4` with `# renovate: ...`). The `stable` tag is acceptable for initial testing but version-pinning is better for GitOps/Argo CD reproducibility.

**Do NOT use** `ghcr.io/onedr0p/home-assistant` (community fork used in the bjw-s example) — the homelab policy prefers official images, and the official `ghcr.io/home-assistant/home-assistant` image is fully functional in containers.

[Sources: home-assistant.io/installation/linux (Docker instructions use `ghcr.io/home-assistant/home-assistant:stable`); github.com/orgs/home-assistant/packages/container/package/home-assistant; ADR-0013 changelog]

### 6. HA Container Constraint (Forced by Kubernetes)

**Architecture Decision Record ADR-0013** ([source](https://github.com/home-assistant/architecture/blob/master/adr/0013-home-assistant-container.md)) defines "Home Assistant Container" as a supported installation method:

> "This is for running just the Home Assistant Core application on native OCI compatible containerization system. It does not provide the Supervisor experience, and thus does not provide the Supervisor panel and add-ons."

Key constraints:
- **HA OS and HA Supervised cannot run in Kubernetes.** HA OS is an embedded operating system (requires bare metal or VM). HA Supervised requires systemd and a Docker daemon with specific privileges that are not available in k8s pods. Only **HA Container** works on Kubernetes.
- **HA Container does not include the Supervisor** — no add-ons, no one-click updates, no built-in backup management via the Supervisor UI. Backups must be handled externally (HA's built-in backup feature still works from the UI; add-ons like code-server must be deployed as sidecars if desired).
- **Some integrations are limited:** Thread and Z-Wave are controlled by add-ons and have "no out-of-the-box support on Container installations." [Source: home-assistant.io/installation](https://www.home-assistant.io/installation/)
- **ADR-0013 states:** "The only supported way to run the container is on the host network as root with full privileges." This is a networking/deployment-flavor concern relevant to ticket #157 (hostNetwork, hostPort), not this chart-landscape ticket.

**Confirmation:** The chart must support running the plain `ghcr.io/home-assistant/home-assistant` container image. Both bjw-s `app-template` and the pajikos chart support this. app-template wraps any OCI image; pajikos defaults to the official image.

---

## Recommendation

> **Use bjw-s `app-template` v5.0.1 from `https://bjw-s-labs.github.io/helm-charts` wrapping image `ghcr.io/home-assistant/home-assistant:stable`** (version-pinned with Renovate annotation for production).

### Rationale

1. **No official Helm chart exists** — confirmed by exhaustive search of the home-assistant GitHub org (106 repos, zero Helm chart repos) and official installation documentation.
2. **Homelab policy** designates bjw-s `app-template` as the fallback when no official chart exists.
3. **Consistency** — three existing apps in this repo (readeck, searxng, unpoller) already use app-template v5.0.1 from `https://bjw-s-labs.github.io/helm-charts`. Using the same chart for HA keeps the deployment pattern uniform.
4. **The official HA container image works with any chart** — `ghcr.io/home-assistant/home-assistant` is a standard OCI image that app-template can wrap without issue.
5. **k8s-native** — app-template supports StatefulSet (recommended for HA's `/config` persistence), PVCs, ingress/Gateway API HTTPRoute, init containers, sidecars (e.g., code-server), serviceMonitor, and all standard k8s primitives.

### For ticket #157 (deployment flavor) to confirm

The chart choice is settled. The following deployment-flavor decisions remain for #157:
- **Controller type:** StatefulSet (recommended for `/config` persistent storage) vs Deployment
- **hostNetwork:** ADR-0013 prescribes host network for full HA functionality (mDNS, discovery, local integrations). app-template supports `hostNetwork: true` via pod options. The pajikos chart also supports this natively if app-template's pod options prove insufficient.
- **Image tag:** `stable` for initial testing → version-pinned (e.g., `2026.8.0`) with Renovate annotation for production
- **Image:** Use `ghcr.io/home-assistant/home-assistant` (official), NOT `ghcr.io/onedr0p/home-assistant` (community fork used in bjw-s example)
- **Storage:** PVC for `/config` (local-path storage class, matching readeck's pattern)
- **Add-ons:** code-server sidecar (if desired) — app-template supports multi-container pods

### Argo CD Application pattern (matching existing apps)

```yaml
# argocd/apps/home-assistant.yaml (sketch — deployment flavor TBD in #157)
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: home-assistant
  namespace: argocd
  annotations:
    argocd.argoproj.io/sync-wave: "160"
    argocd.argoproj.io/compare-options: ServerSideDiff=true
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  sources:
    - chart: app-template
      repoURL: https://bjw-s-labs.github.io/helm-charts
      targetRevision: 5.0.1
      helm:
        releaseName: home-assistant
        parameters: []
        valueFiles:
          - $values/argocd/values/home-assistant.yaml
    - repoURL: https://github.com/chealion/homelab
      targetRevision: main
      ref: values
  destination:
    name: in-cluster
    namespace: home-assistant
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

---

## Sources

### Kept
- **Home Assistant Installation page** (https://www.home-assistant.io/installation/) — confirms only HA OS and HA Container installation types; no Kubernetes/Helm path
- **ADR-0013: Home Assistant Container** (https://github.com/home-assistant/architecture/blob/master/adr/0013-home-assistant-container.md) — defines HA Container as OCI-only, no Supervisor, host network/root recommended
- **home-assistant GitHub org** (https://github.com/home-assistant) — 106 repos, no helm-charts repo; confirms no official chart
- **k8s-at-home/charts deprecation issue #1761** (https://github.com/k8s-at-home/charts/issues/1761) — archived Aug 2022, library chart moved to bjw-s
- **bjw-s-labs/helm-charts app-template** (https://github.com/bjw-s-labs/helm-charts/tree/main/charts/other/app-template) — current chart, v5.0.1, with HA example
- **bjw-s HA example** (https://bjw-s-labs.github.io/helm-charts/docs/app-template/examples/home-assistant-codeserver/) — official app-template HA deployment example
- **pajikos/home-assistant-helm-chart** (https://github.com/pajikos/home-assistant-helm-chart) — most popular dedicated community HA chart; uses official image; supports hostNetwork/hostPort
- **HA Container package on GHCR** (https://github.com/orgs/home-assistant/packages/container/package/home-assistant) — official image registry
- **helm/charts stable/home-assistant** (https://github.com/helm/charts/tree/master/stable/home-assistant) — legacy chart, archived Feb 2022
- **Chealion/homelab repo** (https://github.com/Chealion/homelab) — confirmed existing app-template v5.0.1 pattern (readeck, searxng, unpoller)
- **app-template-5.0.1 release** (https://github.com/bjw-s-labs/helm-charts/releases/tag/app-template-5.0.1) — current version, released 2026-05-14

### Dropped
- **gabe565 Helm Charts** (https://charts.gabe565.com/charts/home-assistant/) — uses bjw-s common as dependency; redundant with app-template itself
- **Sironite/helm-home-assistant** (https://github.com/Sironite/helm-home-assistant) — too opinionated (Authentik/ESO integration); not aligned with homelab pattern
- **Wrenix helm-charts** (https://wrenix.eu/docs/helm-charts/home-assistant/) — k3s-focused, less mainstream
- **andrenarchy/helm-charts** (https://github.com/andrenarchy/helm-charts/tree/main/charts/home-assistant) — based on deprecated k8s-at-home chart; low activity
- **blog posts** (virtualcontainer.eu, quadmeup.com) — personal tutorials, not primary sources; mentioned "no official Helm chart" which corroborates findings but not cited as authority

## Gaps

- **Exact current HA stable version:** The research confirmed `ghcr.io/home-assistant/home-assistant:stable` is the recommended tag, but the specific version number at deploy time should be fetched from the [HA releases page](https://github.com/home-assistant/core/releases) or the [GHCR package](https://github.com/orgs/home-assistant/packages/container/package/home-assistant) when implementing. Community reports indicate the `stable` tag can occasionally lag behind the latest release.
- **onedr0p image differences:** The bjw-s example uses `ghcr.io/onedr0p/home-assistant` (a community fork). This research did not deep-dive the differences between onedr0p's image and the official image. The official image is recommended regardless; if onedr0p-specific k8s patches exist, that would be a #157 consideration.
- **hostNetwork compatibility with this cluster's Cilium CNI:** ADR-0013 recommends host networking for HA. Whether hostNetwork works cleanly with this homelab's Cilium setup is a deployment-flavor question for #157, not a chart-landscape question.
