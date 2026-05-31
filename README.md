# Talos Linux Kubernetes Homelab

Single-node Kubernetes cluster running on a Lenovo ThinkCentre M710q Tiny,
built as part of a cybersecurity career transition portfolio.

## Hardware

| Component | Detail |
|---|---|
| Node | Lenovo ThinkCentre M710q Tiny |
| CPU | Intel Core i5-6400T (4 core, VT-x enabled) |
| RAM | 7.6 GiB DDR4 |
| Storage | 240GB SSD (FCS-240GB) |
| Network | Intel I219-LM (enp0s31f6) |
| Router | EE Smart Hub 6 Plus (192.168.1.254) |

## Stack

| Component | Version |
|---|---|
| Talos Linux | 1.13.3 |
| Kubernetes | 1.35.0 |
| MetalLB | 0.14.9 |
| Pi-hole | 6.x (latest) |
| Homepage | latest |

## Cluster architecture

EE Smart Hub 6 Plus (192.168.1.254)
└── ThinkCentre M710q (192.168.1.10)
├── Talos Linux 1.13.3
└── Kubernetes 1.35.0
├── MetalLB (L2 mode, pool: 192.168.1.200-210)
├── Pi-hole (DNS: 192.168.1.200, Web: 192.168.1.200/admin)
└── Homepage (Dashboard: 192.168.1.201:3000)

## What this builds

- Talos Linux installed to bare metal via USB
- Single-node Kubernetes cluster (control plane scheduling enabled)
- MetalLB in L2 mode providing LoadBalancer IPs from the home network range
- Pi-hole deployed as a Kubernetes workload with persistent storage via hostPath
- Network-wide DNS filtering served from 192.168.1.200
- EE Hub primary DNS pointed at Pi-hole
- Homepage dashboard providing a single-pane view of cluster services

## Key configuration decisions

**Talos over standard Linux:** Talos is an immutable, API-driven OS designed
specifically for Kubernetes. No SSH, no shell all configuration is declarative
YAML applied via talosctl. Chosen to evidence infrastructure-as-code practice
and align with production Kubernetes patterns.

**MetalLB L2 mode:** Provides LoadBalancer-type services on a home network
without a cloud provider. Requires a secondary IP (192.168.1.200/32) assigned
to the physical interface so the host kernel accepts inbound traffic before
nftables forwarding rules apply.

**Pi-hole in Kubernetes:** Rather than running Pi-hole on a dedicated device,
it runs as a Kubernetes Deployment. This demonstrates workload management,
namespace isolation, PodSecurity policy handling, and service exposure via
LoadBalancer all on the same node.

**Homepage dashboard:** Deployed as a Kubernetes workload rather than a
standalone container. Provides live cluster metrics via the Kubernetes API,
service links, and a datetime widget. ConfigMap-driven configuration means
all dashboard state is version-controlled in this repo.

talos-k8s-homelab/
├── homepage/                    # Homepage dashboard manifests
│   ├── configmap.yaml           # Dashboard configuration (services, widgets, bookmarks)
│   ├── deployment.yaml          # Homepage Deployment
│   ├── namespace.yaml           # homepage namespace
│   ├── rbac.yaml                # ServiceAccount and ClusterRoleBinding
│   └── service.yaml             # LoadBalancer service (192.168.1.201:3000)
├── pihole/                      # Pi-hole DNS filtering manifests
│   ├── pihole.yaml              # Pi-hole Deployment and LoadBalancer Services
│   ├── pihole-dns-patch.yaml    # Annotation patch for DNS service IP sharing
│   └── pihole-web-patch.yaml    # Annotation patch for web service IP sharing
├── controlplane-redacted.yaml   # Talos machine config with secrets removed
├── metallb-pool.yaml            # MetalLB IP address pool and L2 advertisement
├── metallb-speaker-patch.yaml   # hostNetwork patch for MetalLB speaker
├── interface-patch.yaml         # NIC configuration patch
├── patch.json                   # Additional Talos machine config patch
├── port53-patch.yaml            # Talos firewall patch for DNS port
└── talos-firewall-patch.yaml    # Talos firewall configuration
## Repository structure

## Deployment order

1. Flash Talos ISO to USB, boot ThinkCentre, apply controlplane.yaml
2. Bootstrap etcd: `talosctl bootstrap`
3. Pull kubeconfig: `talosctl kubeconfig`
4. Deploy MetalLB: `kubectl apply -f metallb-pool.yaml`
5. Patch MetalLB speaker: `kubectl patch daemonset -n metallb-system speaker --patch-file metallb-speaker-patch.yaml`
6. Deploy Pi-hole: `kubectl apply -f pihole/pihole.yaml`
7. Label pihole namespace privileged for PodSecurity
8. Apply Pi-hole patches: `kubectl apply -f pihole/pihole-dns-patch.yaml -f pihole/pihole-web-patch.yaml`
9. Point router DNS at 192.168.1.200
10. Deploy Homepage: `kubectl apply -f homepage/`

## Security+ domains evidenced

| Domain | Evidence |
|---|---|
| 2.0 Threats, Vulnerabilities and Mitigations | Network segmentation, DNS filtering, service isolation via namespaces |
| 3.0 Security Architecture | Defence-in-depth network design, infrastructure as code, least-privilege RBAC |
| 4.0 Security Operations | Service monitoring via Homepage dashboard, log analysis via kubectl, workload health visibility |

## Skills demonstrated

- Bare metal Linux deployment
- Kubernetes cluster administration
- Infrastructure as code (declarative YAML, version-controlled configuration)
- Network services (DNS, DHCP interaction, L2 ARP)
- Container workload management and namespace isolation
- RBAC and ServiceAccount configuration
- Dashboard and observability tooling
- Troubleshooting: certificate mismatches, nftables forwarding, PodSecurity
  policies, read-only ConfigMap mount constraints

## Next steps

- Phase 2: Wazuh SIEM deployed in VMware Workstation Pro on MSI Stealth 16
- Network monitoring lab: Zeek or Suricata on ThinkCentre
- GitHub Actions: validate YAML manifests on push
- Pi-hole widget: re-enable once fresh app password is generated
