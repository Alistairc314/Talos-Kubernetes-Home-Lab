# Talos Kubernetes Home Lab
> Last updated: June 2026

## Objective

This lab builds a production-pattern Kubernetes environment on dedicated bare-metal hardware, used as the
platform for security tooling deployment and network traffic monitoring. Current services running in the
cluster include network-level DNS filtering with full query logging across all home network clients.
Wazuh SIEM deployment follows the Security+ exam in July 2026 and will run on this infrastructure.

The lab covers Security Architecture and Security Operations domains from CompTIA Security+ SY0-701,
with direct application to SOC and GRC environments: namespace isolation, RBAC, infrastructure as code,
and observable workload state.

---

## Environment

| Component    | Detail                                      |
|--------------|---------------------------------------------|
| Host         | Lenovo ThinkCentre M710q Tiny               |
| CPU          | Intel Core i5-6400T (4 core, VT-x enabled)  |
| RAM          | 7.6 GiB DDR4                                |
| Storage      | 240 GB SSD (FCS-240GB)                      |
| NIC          | Intel I219-LM (enp0s31f6)                   |
| OS           | Talos Linux 1.13.3                          |
| Kubernetes   | 1.35.0                                      |
| MetalLB      | 0.14.9 (L2 mode)                            |
| Router       | EE Smart Hub 6 Plus (192.168.1.254)         |

---

## Architecture

```
EE Smart Hub 6 Plus (192.168.1.254)
└── ThinkCentre M710q — 192.168.1.10
    ├── Talos Linux 1.13.3
    └── Kubernetes 1.35.0
        ├── MetalLB (L2 mode, pool: 192.168.1.200–210)
        ├── pihole namespace
        │   └── Pi-hole (DNS: 192.168.1.200, admin: 192.168.1.200/admin)
        │       └── hostPath volume: /var/lib/pihole
        └── homepage namespace
            └── Homepage dashboard (192.168.1.201:3000)
```

All home network clients route DNS through Pi-hole at `192.168.1.200`. Query logs are
visible in the Pi-hole admin interface, providing a baseline view of DNS-layer traffic
across the network.

---

## Key configuration decisions

**Talos Linux over standard Linux**
Talos is an immutable, API-driven OS with no SSH and no interactive shell. All configuration
is declarative YAML applied via `talosctl`. This reflects production Kubernetes security
practice: no persistent remote access, no configuration drift, no manual state.

**MetalLB in L2 mode**
Provides `LoadBalancer`-type services on a home network without a cloud provider. A secondary
IP address (`192.168.1.200/32`) is assigned to the physical interface so the host kernel
accepts inbound traffic before `nftables` forwarding rules apply. This resolved a non-obvious
routing issue during initial deployment.

**Pi-hole in Kubernetes rather than on dedicated hardware**
Running Pi-hole as a Kubernetes Deployment rather than a standalone device demonstrates
workload management, namespace isolation, PodSecurity policy handling, `hostNetwork` mode,
and `LoadBalancer` service exposure — all on the same node. DNS queries from all network
clients are visible in the query log, with source IP preserved via `hostNetwork: true`.

**hostNetwork and DNSMASQ_LISTENING configuration**
Without `hostNetwork: true`, all DNS queries appear to originate from the pod gateway
(`10.244.0.1`), masking real client IPs. The correct configuration requires `hostNetwork: true`,
`dnsPolicy: ClusterFirstWithHostNet`, and `DNSMASQ_LISTENING: all` in combination. This was
identified through log analysis and corrected by patching the deployment.

**ConfigMap mount constraints**
Homepage reads configuration from a read-only ConfigMap mount. The ConfigMap must declare
`docker.yaml`, `custom.css`, and `custom.js` as explicit keys — even when empty — or the pod
crashes on startup. This was debugged from pod logs and resolved by updating the ConfigMap
manifest.

---

## What I did

1. Flashed Talos ISO to USB and booted ThinkCentre from bare metal
2. Applied `controlplane.yaml` via `talosctl apply-config` with disk selector targeting the FCS-240GB SSD
3. Bootstrapped etcd and pulled kubeconfig: `talosctl bootstrap` → `talosctl kubeconfig`
4. Applied MetalLB manifests and patched the speaker DaemonSet with `hostNetwork: true`
5. Configured MetalLB IP pool (`192.168.1.200–210`) and L2 advertisement
6. Deployed Pi-hole to `pihole` namespace with `hostNetwork: true`, `hostPath` persistent volume, and two `LoadBalancer` services sharing `192.168.1.200` via shared IP annotations
7. Labelled `pihole` namespace privileged to satisfy PodSecurity admission
8. Applied firewall patch to open port 53 on the Talos host
9. Set EE Smart Hub primary DNS to `192.168.1.200`
10. Deployed Homepage to `homepage` namespace with Kubernetes API RBAC, ConfigMap-driven configuration, and `LoadBalancer` service at `192.168.1.201:3000`

---

## Key findings and observations

- Source IP preservation in Pi-hole requires `hostNetwork: true` at the pod level. The default CNI gateway masking is not obvious from the Pi-hole documentation and was identified by comparing query log source IPs against known client addresses.
- Talos `nftables` rules block port 53 by default at the host level. DNS was unreachable from the network until a dedicated firewall patch was applied via `talosctl`.
- MetalLB speaker requires `hostNetwork: true` to respond to ARP requests from the router for IPs in the assigned pool. Without this, `LoadBalancer` services receive an IP but are unreachable from the network.
- Homepage ConfigMap crashes are silent at the deployment level — the pod log is the only place the missing key error appears.

---

## Screenshots

<!-- Pi-hole query log showing network-wide DNS traffic -->
<!-- Homepage dashboard showing cluster service status -->

---

## Lessons learned

The most instructive part of this build was not the installation — it was the debugging. Three separate issues (MetalLB ARP, Pi-hole source IP masking, and the ConfigMap crash) all required reading logs, forming a hypothesis, and testing a targeted fix. Each one maps to a pattern that appears in SOC work: an observable symptom, a root cause that isn't where you first look, and a configuration change that resolves it cleanly.

Talos's lack of a shell makes troubleshooting more deliberate. There is no option to log in and poke around. Everything has to be reasoned from `kubectl` output, pod logs, and `talosctl dmesg`.

---

## Security+ domains evidenced

| Domain                                        | Weight | How this lab covers it                                                                                      |
|-----------------------------------------------|--------|--------------------------------------------------------------------------------------------------------------|
| 2.0 Threats, Vulnerabilities & Mitigations    | 22%    | DNS filtering blocks known malicious domains at the network layer; namespace isolation limits blast radius   |
| 3.0 Security Architecture                     | 18%    | Defence-in-depth network design; infrastructure as code; least-privilege RBAC; immutable OS (Talos)         |
| 4.0 Security Operations                       | 28%    | Query log monitoring; workload health visibility; log-driven troubleshooting; observable cluster state       |

---

## Repository structure

```
talos-k8s-homelab/
├── homepage/
│   ├── configmap.yaml          # Dashboard config (services, widgets, bookmarks)
│   ├── deployment.yaml         # Homepage Deployment
│   ├── namespace.yaml          # homepage namespace
│   ├── rbac.yaml               # ServiceAccount and ClusterRoleBinding
│   └── service.yaml            # LoadBalancer service (192.168.1.201:3000)
├── pihole/
│   ├── pihole.yaml             # Pi-hole Deployment and LoadBalancer services
│   ├── pihole-dns-patch.yaml   # Annotation patch for DNS service IP sharing
│   └── pihole-web-patch.yaml   # Annotation patch for web service IP sharing
├── controlplane-redacted.yaml  # Talos machine config (secrets removed)
├── metallb-pool.yaml           # MetalLB IP pool and L2 advertisement
├── metallb-speaker-patch.yaml  # hostNetwork patch for MetalLB speaker
├── interface-patch.yaml        # NIC configuration patch
├── patch.json                  # Additional Talos machine config patch
├── port53-patch.yaml           # Talos firewall patch for DNS
└── talos-firewall-patch.yaml   # Talos firewall configuration
```

---

## Deployment order

1. Flash Talos ISO to USB, boot ThinkCentre, apply `controlplane.yaml`
2. Bootstrap etcd: `talosctl bootstrap`
3. Pull kubeconfig: `talosctl kubeconfig`
4. Deploy MetalLB: `kubectl apply -f metallb-pool.yaml`
5. Patch MetalLB speaker: `kubectl patch daemonset -n metallb-system speaker --patch-file metallb-speaker-patch.yaml`
6. Deploy Pi-hole: `kubectl apply -f pihole/pihole.yaml`
7. Label pihole namespace privileged for PodSecurity
8. Apply Pi-hole patches: `kubectl apply -f pihole/pihole-dns-patch.yaml -f pihole/pihole-web-patch.yaml`
9. Apply port 53 firewall patch: `talosctl patch mc --patch @port53-patch.yaml`
10. Set router DNS to `192.168.1.200`
11. Deploy Homepage: `kubectl apply -f homepage/`

---

## Next steps

- Deploy Wazuh SIEM on MSI Stealth 16 (VMware) — agents to ship logs from ThinkCentre and home network
- Network monitoring: Zeek or Suricata on ThinkCentre for traffic analysis alongside DNS filtering
- GitHub Actions: YAML manifest validation on push
- Re-enable Pi-hole Homepage widget once fresh v6 app password is generated
