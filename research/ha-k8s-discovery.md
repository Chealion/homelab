# HA Discovery and mDNS in Kubernetes

> **Wayfinder ticket #153** — Chealion/homelab
> Resolves: what HA requires for device discovery (mDNS / SSDP / UPnP) when running in Kubernetes, and what options are viable in this homelab's setup (Talos OS + Cilium CNI, LAN 192.168.x, HA Container).

---

## TL;DR / Recommendation

**Discovery approach = `hostNetwork: true` (primary), with static configuration as fallback for integrations that don't need auto-discovery.**

Cilium's multicast support is cluster-internal only (beta, pod-to-pod via VXLAN) and cannot bridge mDNS/SSDP multicast between pods and the LAN — Cilium actively drops traffic to `224.0.0.251` (mDNS) and `239.255.255.250` (SSDP). MetalLB/LoadBalancer Services do not forward multicast or broadcast traffic. An mDNS repeater alone is insufficient because it does not cover SSDP/UPnP or HomeKit's same-subnet requirement. HA Container's own architecture decision (ADR-0013) states that host network is the only supported deployment mode. `hostNetwork: true` simultaneously solves mDNS and SSDP, requires no Talos machine-config changes, and is the simplest path on this cluster. If network isolation is later deemed critical, Multus CNI with an ipvlan secondary interface is the documented upgrade path — but it is a larger infrastructure change.

**Blocked ticket #158 (networking decision) should confirm this choice.** Storage, auth, and chart selection are out of scope for this ticket.

---

## 1. What HA Actually Needs for Discovery

Home Assistant uses four discovery mechanisms. Two of them — mDNS/Zeroconf and SSDP/UPnP — rely on multicast/broadcast traffic on the LAN and are the ones that break in Kubernetes.

### 1.1 mDNS / Zeroconf (UDP 224.0.0.251:5353)

- HA uses the `python-zeroconf` library to send and receive mDNS multicast on `224.0.0.251:5353`.
- The `zeroconf` integration is enabled by default (part of `default_config`).
- It both **discovers** devices (scans for advertised services) and **advertises** HA itself (makes HA discoverable to other services like Apple HomeKit).
- **90+ integrations** are auto-discovered via zeroconf, including: Apple TV, ESPHome, Google Cast, HomeKit Bridge, HomeKit Device, Hue, Matter, Nanoleaf, Samsung Smart TV, Shelly, Sonos, WLED, ZHA, Z-Wave, and many more. ([zeroconf integration list](https://www.home-assistant.io/integrations/zeroconf/))
- The `network` integration controls which interfaces mDNS broadcasts on, auto-detecting based on the routing next-hop for `224.0.0.251`. ([Network integration](https://www.home-assistant.io/integrations/network/))

### 1.2 SSDP / UPnP (UDP 239.255.255.250:1900)

- The `ssdp` integration scans the network for UPnP devices via multicast to `239.255.255.250:1900`.
- Also enabled by default (part of `default_config`).
- **38+ integrations** are auto-discovered via SSDP, including: Samsung Smart TV, Sonos, LG webOS TV, Denon AVR, Roku, Synology DSM, UniFi Network, UPnP/IGD, Logitech Harmony, deCONZ, and more. ([SSDP integration list](https://www.home-assistant.io/integrations/ssdp/))
- SSDP uses a **different multicast group and port** than mDNS — a solution that only bridges mDNS will not cover SSDP.

### 1.3 DHCP discovery

- HA listens to DHCP traffic to discover devices by hostname/MAC. This requires the pod to see DHCP broadcast traffic on the LAN — same overlay-network problem as mDNS/SSDP, but lower impact (fewer integrations, most can be configured manually).

### 1.4 USB discovery

- Discovers USB devices (Zigbee dongles, Z-Wave sticks, etc.). Requires device passthrough, not multicast. Not affected by the k8s networking problem (separate concern — USB passthrough via `hostPath` or `devPath`).

### 1.5 HomeKit same-subnet requirement

- HomeKit integration has an additional constraint: HA and the HomeKit device must be on the **same /24 subnet**. mDNS alone is not sufficient — the IP packets reaching the device must have a source address in the same subnet. ([swrm.io](https://swrm.io/posts/homeassistant_kubernetes/), [HA issue #95413](https://github.com/home-assistant/core/issues/95413))
- Samsung TVs have a similar same-subnet networking requirement.
- This means even an mDNS repeater that bridges queries will not fully solve HomeKit — HA needs a LAN IP on the correct subnet.

### 1.6 Which integrations can use static IP / manual configuration?

Many integrations support **manual configuration** as an alternative to auto-discovery. For example:
- **Shelly**: "If it wasn't discovered automatically, don't worry! You can set up a manual integration entry." ([Shelly docs](https://www.home-assistant.io/integrations/shelly/))
- **Sonos**: Can be configured by IP. ([Sonos docs](https://www.home-assistant.io/integrations/sonos/))
- **Hue**: Requires the bridge IP, discovered via SSDP but can be entered manually.
- **ESPHome**: Can be configured by static IP.
- **Most network-based integrations**: Accept a host/IP in their config flow.

However, some integrations **require** mDNS to function, not just for discovery:
- **HomeKit Bridge**: Must advertise via mDNS (`_hap._tcp`) for Apple devices to find it. Cannot work with static IP alone.
- **Matter**: Uses mDNS for commissioning and operational discovery.
- **Google Cast local fulfillment**: Requires mDNS to advertise the local fulfillment path.
- **Google Cast discovery**: Uses mDNS to find Chromecast devices.

**Bottom line:** Static-only configuration works for many integrations but **not all**. HomeKit, Matter, and Cast are the notable hard requirements for mDNS.

---

## 2. Why Kubernetes Breaks Discovery

Pods in Kubernetes live in an overlay/pod network (e.g., 10.42.x.x) that is separate from the LAN (192.168.x). Multicast and broadcast traffic does not traverse this boundary by default:

- **mDNS** (224.0.0.251:5353) — link-local multicast, scoped to the broadcast domain. Pod overlay networks are separate broadcast domains.
- **SSDP** (239.255.255.250:1900) — multicast, same problem.
- **DHCP** — broadcast, same problem.

HA running in a pod on the pod network will never see mDNS/SSDP traffic from LAN devices, and LAN devices will never see HA's mDNS advertisements. Discovery silently fails. ([swrm.io](https://swrm.io/posts/homeassistant_kubernetes/), [meshlaneous.dev](https://meshlaneous.dev/blog/hass-in-k8s))

---

## 3. Option Analysis

### 3.1 `hostNetwork: true`

**What it gives:**
- The pod shares the host node's network namespace directly. It gets a LAN IP, can send/receive mDNS and SSDP multicast, and is on the correct subnet for HomeKit.
- Both mDNS and SSDP work immediately with no additional components.
- HA's own ADR-0013 states: *"The only supported way to run the container is on the host network as root with full privileges."* ([HA ADR-0013](https://github.com/home-assistant/architecture/blob/master/adr/0013-home-assistant-container.md))
- Simplest configuration: one line in the pod spec.

**What it costs:**
- **One pod per node**: port 8123 binds to the node's IP, so only one HA pod can run per node. Use `nodeSelector` or `podAntiAffinity` to pin to a specific node.
- **`dnsPolicy: ClusterFirstWithHostNet`** required to still use cluster DNS for service resolution.
- **Reduced isolation**: the pod has full access to the host network stack. The HA Container ADR already accepts this (Container runs as root with full privileges).
- **Ingress/service interaction**: with `hostNetwork: true`, a `ClusterIP` Service still works but the pod is directly on the LAN. An Ingress/Gateway can still route to it.
- **Rollout strategy**: use `Recreate` (not `RollingUpdate`) to avoid port conflicts during updates.

**Example pod spec:**
```yaml
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet
  nodeSelector:
    kubernetes.io/hostname: <specific-node>
  containers:
    - name: home-assistant
      image: ghcr.io/home-assistant/home-assistant:stable
      ports:
        - containerPort: 8123
```

**Sources:** [HA ADR-0013](https://github.com/home-assistant/architecture/blob/master/adr/0013-home-assistant-container.md), [meshlaneous.dev](https://meshlaneous.dev/blog/hass-in-k8s), [Warren Amphlett](https://blog.warrenamphlett.co.uk/homelab/home-assistant-in-kubernetes), [drpump/k3s-home-assistant](https://github.com/drpump/k3s-home-assistant)

### 3.2 LoadBalancer / MetalLB / Gateway

**Does it expose discovery? No.**

- Kubernetes `type: LoadBalancer` (MetalLB or any cloud LB) operates at L4 and forwards **unicast** TCP/UDP traffic to pod endpoints. It does **not** forward multicast or broadcast traffic.
- MetalLB issue [#344](https://github.com/metallb/metallb/issues/344): *"Kubernetes's LoadBalancer logic (which MetalLB relies on) does not support forwarding broadcast traffic into pods, so this simply will not work in k8s."*
- MetalLB issue [#253](https://github.com/google/metallb/issues/253): layer2 mode doesn't receive broadcast packets unless promiscuous mode is enabled, and even then it's not a supported multicast path.
- A Service + Ingress/Gateway is fine for **publishing HA's web UI** (TCP 8123) to the LAN, but it does **nothing for discovery**.

**What does work behind an LB:** The web UI, REST API, and WebSocket traffic — all unicast TCP. Discovery (mDNS/SSDP) must be solved separately.

**Sources:** [MetalLB #344](https://github.com/metallb/metallb/issues/344), [MetalLB #253](https://github.com/google/metallb/issues/253), [Varac: Kubernetes limitations](https://www.varac.net/docs/kubernetes/limitations.html)

### 3.3 mDNS Repeater / Reflector

**What it is:** A sidecar container or separate DaemonSet that bridges mDNS multicast between the LAN and the pod network. The repeater must itself be on the LAN (via `hostNetwork: true` or Multus) and reflect mDNS packets to the pod's overlay interface.

**Options:**
- **Avahi daemon** (reflector mode): Can be run as a DaemonSet with `hostNetwork: true`. The `enable-reflector=yes` setting bridges mDNS between interfaces. Warren Amphlett's approach: install Avahi on each node, configure reflector to bridge LAN → pod network. ([Warren Amphlett blog](https://blog.warrenamphlett.co.uk/homelab/home-assistant-in-kubernetes))
- **mdns-repeater** (e.g., `rauchg/mdns-repeater`, `jdelker/docker-mdns-repeater`): Lightweight container that repeats mDNS between specified interfaces. ([jdelker/docker-mdns-repeater](https://github.com/jdelker/docker-mdns-repeater))
- **mdns-reflector** (`vfreex/mdns-reflector`): Reflects mDNS queries and responses among multiple LANs. ([vfreex/mdns-reflector](https://github.com/vfreex/mdns-reflector))
- **Router-level mDNS repeater**: Some routers (UniFi, OPNsense) have built-in mDNS repeater/reflector for cross-VLAN. This solves cross-VLAN but not the pod-to-LAN boundary.

**Limitations:**
- **Only solves mDNS, not SSDP/UPnP.** mDNS repeaters bridge `224.0.0.251:5353`. SSDP uses `239.255.255.250:1900` — a different multicast group and port. A separate SSDP/UPnP relay would be needed (less common, more complex).
- **Does not solve HomeKit's same-/24 requirement.** Even if mDNS queries are reflected, HomeKit devices require HA's IP to be on the same subnet. A repeater doesn't give HA a LAN IP.
- **Does not solve HA advertising itself.** HA needs to be discoverable (e.g., for Apple HomeKit, Google Cast local fulfillment). A repeater reflecting from pod to LAN may advertise HA's pod IP, not a reachable LAN IP.
- **On Talos OS**, installing Avahi as a system daemon is not straightforward (immutable OS). It would need to be a container DaemonSet with `hostNetwork: true` — which itself requires the same network access that `hostNetwork: true` on the HA pod would provide.

**Typical k8s deployment:**
```yaml
# DaemonSet with hostNetwork, bridging mDNS between host LAN interface and pod CIDR
spec:
  hostNetwork: true
  containers:
    - name: mdns-repeater
      image: jdelker/docker-mdns-repeater:latest
      args: ["eth0", "cilium_host"]  # LAN interface, pod interface
```

**Verdict:** Insufficient as a standalone solution. Only covers mDNS, not SSDP or HomeKit subnet requirements. Adds operational complexity. The repeater itself needs LAN access (hostNetwork), so you're already paying the `hostNetwork` cost for a component that provides less than just putting HA on hostNetwork.

**Sources:** [Warren Amphlett](https://blog.warrenamphlett.co.uk/homelab/home-assistant-in-kubernetes), [jdelker/docker-mdns-repeater](https://github.com/jdelker/docker-mdns-repeater), [vfreex/mdns-reflector](https://github.com/vfreex/mdns-reflector), [Reddit: mDNS in k8s pods](https://www.reddit.com/r/kubernetes/comments/10iung5/how_do_you_get_mdns_working_in_kubernetes_pods/)

### 3.4 Multus CNI (Secondary Interface)

**What it is:** Multus is a CNI meta-plugin that attaches additional network interfaces to pods. A secondary interface (macvlan or ipvlan) bridged to the LAN gives the pod a LAN IP without full `hostNetwork: true`.

**How it works:**
- Pod gets two interfaces: `eth0` (cluster/pod network via Cilium) + `net1` (LAN via macvlan/ipvlan attached to the host's LAN interface).
- mDNS/SSDP traffic goes out `net1` directly to the LAN.
- HA's `network` integration must be configured to broadcast on the `net1` interface (Settings → System → Network).
- Cluster DNS still works via `eth0`.

**What it costs:**
- **Requires node-level configuration**: bridge/VLAN interface must exist on the node. On Talos, this means custom machine config (`machine.network.interfaces` with VLAN/bridge settings).
- **NetworkAttachmentDefinition** resources must be created.
- **IPAM**: must manage LAN IP allocation (Whereabouts or static IPs).
- **Rollout strategy**: `Recreate` (can't have two pods with the same LAN IP during rollout).
- **Node affinity**: pods must be pinned to nodes with the matching LAN interface.
- **macvlan vs ipvlan**: macvlan gives each pod a virtual MAC (some managed switches don't like multiple MACs on one port); ipvlan shares the host MAC (more compatible, but pod can't communicate with host on same interface).

**Sources:** [swrm.io](https://swrm.io/posts/homeassistant_kubernetes/), [meshlaneous.dev](https://meshlaneous.dev/blog/hass-in-k8s), [bjw-s: Run a Pod in a VLAN](https://bjw-s.github.io/home-ops/notes-ramblings/howto/pod-multihome.html)

**Verdict:** Best isolation, but significant infrastructure change. Recommended as an upgrade path if `hostNetwork` security posture is unacceptable. Requires coordination with Talos machine config (separate from Argo CD app-of-apps).

### 3.5 Cilium Multicast (NOT viable for LAN discovery)

**Cilium has a multicast feature (beta), but it is cluster-internal only.**

Key facts from Cilium docs and issues:
- Cilium's `multicast-enabled` feature provides **pod-to-pod multicast fanout within the cluster** via VXLAN replication. It does **not** egress multicast to the LAN. ([Cilium multicast docs](https://docs.cilium.io/en/stable/network/multicast/))
- The feature requires **VXLAN mode** and kernel ≥ 5.10 (AMD64) / ≥ 6.0 (AArch64). Talos Linux ≥ 1.5 is supported by Cilium and ships kernel 6.x, so the kernel requirement is met.
- **Cilium drops mDNS and SSDP multicast traffic.** Issue [#30586](https://github.com/cilium/cilium/issues/30586): Cilium drops IPv4 multicast traffic to `224.0.0.22` and `239.255.255.250` (SSDP) with no clear drop reason. mDNS (`224.0.0.251`) is similarly not passed through.
- **Multicast egress to external networks is not implemented.** CFP [#45836](https://github.com/cilium/cilium/issues/45836) proposes adding multicast egress as a `CiliumEgressGatewayPolicy` extension, but this is a proposal, not a released feature. Today, `CiliumEgressGatewayPolicy` only matches unicast destinations — multicast destinations are silently dropped at the lxc egress hook.
- Configuration is manual per-node (`cilium-dbg bpf multicast group add`), not automatic. Designed for "data feeds to multiple consumers in the Kubernetes cluster," not for LAN discovery.
- Does not work with IPsec encryption between Cilium-managed pods.

**Verdict:** Cilium multicast cannot solve HA's LAN discovery problem. It is designed for intra-cluster multicast (e.g., media streaming between pods), not for bridging pod multicast to the physical LAN. Do not rely on this feature for HA discovery.

**Sources:** [Cilium multicast docs](https://docs.cilium.io/en/stable/network/multicast/), [Cilium #30586](https://github.com/cilium/cilium/issues/30586), [Cilium CFP #45836](https://github.com/cilium/cilium/issues/45836), [Cilium #13239](https://github.com/cilium/cilium/issues/13239)

### 3.6 Static-Only Configuration (No Discovery)

**What it is:** Disable zeroconf/SSDP discovery and configure every integration manually with a static IP or hostname.

**What works:**
- Most network-based integrations (Shelly, Sonos, Hue, ESPHome, MQTT, etc.) accept a manual host/IP in their config flow.
- Integrations using cloud APIs or local HTTP APIs work fine without discovery.
- Zigbee2MQTT, Z-Wave JS over TCP, MQTT-based integrations — all work without mDNS.

**What does NOT work without mDNS:**
- **HomeKit Bridge**: Must advertise via mDNS (`_hap._tcp`). Cannot function without mDNS. Apple devices will not find the bridge.
- **Matter**: Uses mDNS for commissioning and operational discovery. Cannot be commissioned without mDNS.
- **Google Cast**: Discovers Chromecast devices via mDNS. Local fulfillment requires mDNS advertising.
- **Apple TV**: Discovers via mDNS. Manual configuration may work for some models but is unreliable.
- **Auto-discovery of new devices**: Any device added to the network won't be auto-discovered; must be manually added each time.

**Verdict:** Viable as a **fallback** for integrations that support manual configuration, but **not sufficient as the sole approach** if HomeKit, Matter, or Cast is needed. Many users report that even integrations that "support" manual config are more reliable with discovery working.

---

## 4. Cilium Specifics (This Homelab's CNI)

This homelab uses Cilium CNI on Talos Linux. Key findings:

| Aspect | Status |
|---|---|
| Cilium multicast feature | Beta, **cluster-internal only** (pod-to-pod via VXLAN) |
| Multicast egress to LAN | **Not implemented** (CFP #45836 is a proposal, not released) |
| mDNS (224.0.0.251) passthrough | **Dropped** by Cilium (issue #30586) |
| SSDP (239.255.255.250) passthrough | **Dropped** by Cilium (issue #30586) |
| Kernel requirement | ≥ 5.10 (AMD64) / ≥ 6.0 (AArch64) — Talos ≥ 1.5 ships kernel 6.x, **requirement met** |
| VXLAN mode required | Yes, for multicast feature (but irrelevant since egress isn't supported) |
| IPsec compatibility | Multicast feature does not work with IPsec |

**Conclusion:** Cilium cannot bridge mDNS/SSDP between pods and the LAN. The multicast feature is irrelevant for HA discovery. HA's discovery traffic will not traverse Cilium's pod network to reach LAN devices. The solution must be at the pod networking layer (hostNetwork or Multus), not at the CNI layer.

---

## 5. SSDP / UPnP — Same Shape as mDNS

SSDP/UPnP has the same k8s problem as mDNS but with different protocol details:

| Property | mDNS/Zeroconf | SSDP/UPnP |
|---|---|---|
| Multicast group | 224.0.0.251 | 239.255.255.250 |
| Port | UDP 5353 | UDP 1900 |
| Protocol | mDNS (DNS over multicast) | SSDP (HTTP over UDP multicast) |
| HA integration | `zeroconf` (default_config) | `ssdp` (default_config) |
| # of integrations discovered | 90+ | 38+ |
| Breaks in k8s overlay? | Yes | Yes |
| Cilium drops it? | Yes | Yes |
| Solvable with hostNetwork? | Yes | Yes |
| Solvable with mDNS repeater? | Yes (mDNS only) | **No** (separate relay needed) |
| Solvable with Multus? | Yes | Yes |
| Solvable with MetalLB/LB? | No | No |

**Key insight:** `hostNetwork: true` and Multus solve **both** mDNS and SSDP simultaneously because they give the pod direct LAN access. An mDNS-only repeater leaves SSDP broken. This is a strong argument against the repeater-only approach.

---

## 6. Recommendation

### Discovery approach = `hostNetwork: true` (primary) + static configuration (fallback)

**Rationale:**

1. **Cilium cannot do it.** Cilium's multicast is cluster-internal only; it drops mDNS/SSDP traffic and has no egress path to the LAN. No amount of Cilium configuration will solve this.
2. **MetalLB/LoadBalancer cannot do it.** Kubernetes Services do not forward multicast/broadcast.
3. **mDNS repeater is insufficient.** It only covers mDNS (not SSDP), doesn't solve HomeKit's same-/24 requirement, doesn't let HA advertise itself correctly, and on Talos the repeater itself needs hostNetwork anyway.
4. **HA officially supports hostNetwork.** ADR-0013: "The only supported way to run the container is on the host network as root with full privileges."
5. **Solves both mDNS and SSDP.** One configuration change covers all discovery protocols.
6. **Simplest on Talos + Cilium.** No Talos machine-config changes, no Multus, no NetworkAttachmentDefinition, no DaemonSet. Just `hostNetwork: true` + `dnsPolicy: ClusterFirstWithHostNet` + `nodeSelector`.
7. **HomeKit/Matter work.** HA gets a LAN IP on the correct subnet.

**Tradeoffs accepted:**
- One HA pod per node (pin with `nodeSelector`).
- Reduced network isolation (HA Container already runs as root per ADR-0013).
- Use `Recreate` rollout strategy.
- Ingress/Gateway still works for the web UI (route to the node's LAN IP or use a Service).

**If isolation becomes a priority later:** Upgrade to Multus CNI with an ipvlan secondary interface (gives HA a LAN IP without full hostNetwork). This requires Talos machine config for the bridge/VLAN interface and NetworkAttachmentDefinition resources. This is a larger change that touches the Talos layer, not just Argo CD.

**What about the chart?** The `hostNetwork` setting is a standard value in HA Helm charts (e.g., pajikos/home-assistant-helm-chart, bjw-s app-template). Chart selection is a separate ticket.

---

## 7. Implementation Checklist (for ticket #158 / networking decision)

- [ ] Confirm `hostNetwork: true` + `dnsPolicy: ClusterFirstWithHostNet` in the HA deployment spec.
- [ ] Pin HA to a specific node via `nodeSelector` or `nodeAffinity`.
- [ ] Set deployment `strategy: Recreate` (not RollingUpdate) to avoid port conflicts.
- [ ] Ensure the target node's LAN interface is on 192.168.x.
- [ ] Verify mDNS discovery works: check Settings → System → Network → Zeroconf Browser after deploy.
- [ ] Verify SSDP discovery works: check Settings → System → Network → SSDP/UPnP Browser.
- [ ] Configure HA's `network` integration to broadcast on the correct interface (auto-detection should work with hostNetwork).
- [ ] For integrations that don't need discovery, configure with static IPs as a reliability measure.
- [ ] If HomeKit/Matter is needed, verify HA's LAN IP is on the same /24 as the HomeKit devices.

---

## 8. Sources

### Primary / Official
- [HA Developer Docs: Networking and discovery](https://developers.home-assistant.io/docs/network_discovery/) — mDNS/Zeroconf, SSDP, DHCP, USB discovery APIs
- [HA Zeroconf integration](https://www.home-assistant.io/integrations/zeroconf/) — list of 90+ mDNS-discovered integrations
- [HA SSDP integration](https://www.home-assistant.io/integrations/ssdp/) — list of 38+ SSDP-discovered integrations
- [HA Network integration](https://www.home-assistant.io/integrations/network/) — interface auto-detection via mDNS next-hop
- [HA ADR-0013: Home Assistant Container](https://github.com/home-assistant/architecture/blob/master/adr/0013-home-assistant-container.md) — "The only supported way to run the container is on the host network"
- [Cilium Multicast Support (Beta)](https://docs.cilium.io/en/stable/network/multicast/) — cluster-internal only, VXLAN required, kernel ≥ 5.10/6.0
- [Cilium CFP #45836: Multicast egress as CiliumEgressGatewayPolicy extension](https://github.com/cilium/cilium/issues/45836) — multicast egress to LAN is not implemented
- [Cilium #30586: Dropping IPv4 multicast traffic](https://github.com/cilium/cilium/issues/30586) — mDNS/SSDP multicast dropped with no reason
- [Cilium System Requirements](https://docs.cilium.io/en/latest/operations/system_requirements/) — Talos Linux ≥ 1.5 supported
- [MetalLB #344: Listen to UDP broadcast traffic](https://github.com/metallb/metallb/issues/344) — LB does not forward broadcast/multicast
- [MetalLB #253: layer2 mode doesn't receive broadcast packets](https://github.com/google/metallb/issues/253)

### Community / Experience Reports
- [swrm.io: Running Home-Assistant in Kubernetes](https://swrm.io/posts/homeassistant_kubernetes/) — Multus CNI approach, HomeKit same-/24 requirement, HA network interface config
- [meshlaneous.dev: HASS on K8s: Never say never](https://meshlaneous.dev/blog/hass-in-k8s) — hostNetwork vs Multus vs Avahi reflector comparison, Matter Hub hostNetwork
- [Warren Amphlett: Cross-VLAN Home Assistant inside K8S](https://blog.warrenamphlett.co.uk/homelab/home-assistant-in-kubernetes) — Avahi daemon reflector on K3s nodes, moved away from hostNetwork for security
- [drpump/k3s-home-assistant](https://github.com/drpump/k3s-home-assistant) — k3s deployment with hostNetwork: true
- [Reddit: How do you get mdns working in kubernetes pods?](https://www.reddit.com/r/kubernetes/comments/10iung5/how_do_you_get_mdns_working_in_kubernetes_pods/) — community options summary
- [Varac: Kubernetes limitations](https://www.varac.net/docs/kubernetes/limitations.html) — MetalLB does not support broadcast traffic

### mDNS Repeater / Reflector Tools
- [jdelker/docker-mdns-repeater](https://github.com/jdelker/docker-mdns-repeater) — Docker mDNS repeater (Darell Tan's mdns-repeater)
- [vfreex/mdns-reflector](https://github.com/vfreex/mdns-reflector) — Lightweight mDNS reflector with IPv6 support
- [cbrand/mdnsforwarder](https://github.com/cbrand/mdnsforwarder) — mDNS forwarder for VPN/container networks

### Multus / Pod Multihoming
- [bjw-s: Run a Pod in a VLAN](https://bjw-s.github.io/home-ops/notes-ramblings/howto/pod-multihome.html) — Multus + macvlan/ipvlan for HA discovery (404 at time of research; referenced in search results)
- [Intel Multus CNI](https://github.com/k8snetworkplumbingwg/multus-cni) — meta-plugin for secondary interfaces

### HomeKit / Same-Subnet
- [HA issue #95413: Apple TV not getting connected](https://github.com/home-assistant/core/issues/95413) — "IP packets reaching the Apple TV has an address belonging to the same network"
- [HA issue #132288: HomeKit not working with multiple interfaces](https://github.com/home-assistant/core/issues/132288) — mDNS on multiple interfaces
- [HA Community: Homekit x K8s](https://community.home-assistant.io/t/homekit-x-k8s-kubernetes/872498) — HomeKit discovery issues in k8s
