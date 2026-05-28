\# Talos Linux Kubernetes Homelab



Single-node Kubernetes cluster running on a Lenovo ThinkCentre M710q Tiny,

built as part of a cybersecurity career transition portfolio.



\## Hardware



| Component | Detail |

|---|---|

| Node | Lenovo ThinkCentre M710q Tiny |

| CPU | Intel Core i5-6400T (4 core, VT-x enabled) |

| RAM | 7.6 GiB DDR4 |

| Storage | 240GB SSD (FCS-240GB) |

| Network | Intel I219-LM (enp0s31f6) |

| Router | EE Smart Hub 6 Plus (192.168.1.254) |



\## Stack



| Component | Version |

|---|---|

| Talos Linux | 1.13.3 |

| Kubernetes | 1.35.0 |

| MetalLB | 0.14.9 |

| Pi-hole | 6.x (latest) |



\## Cluster architecture

EE Smart Hub 6 Plus (192.168.1.254)

└── ThinkCentre M710q (192.168.1.10)

├── Talos Linux 1.13.3

└── Kubernetes 1.35.0

├── MetalLB (L2 mode, pool: 192.168.1.200-210)

└── Pi-hole (DNS: 192.168.1.200, Web: 192.168.1.200/admin)



\## What this builds



\- Talos Linux installed to bare metal via USB

\- Single-node Kubernetes cluster (control plane scheduling enabled)

\- MetalLB in L2 mode providing LoadBalancer IPs from the home network range

\- Pi-hole deployed as a Kubernetes workload with persistent storage via hostPath

\- Network-wide DNS filtering served from 192.168.1.200

\- EE Hub primary DNS pointed at Pi-hole



\## Key configuration decisions



\*\*Talos over standard Linux:\*\* Talos is an immutable, API-driven OS designed

specifically for Kubernetes. No SSH, no shell — all configuration is declarative

YAML applied via talosctl. Chosen to evidence infrastructure-as-code practice

and align with production Kubernetes patterns.



\*\*MetalLB L2 mode:\*\* Provides LoadBalancer-type services on a home network

without a cloud provider. Requires a secondary IP (192.168.1.200/32) assigned

to the physical interface so the host kernel accepts inbound traffic before

nftables forwarding rules apply.



\*\*Pi-hole in Kubernetes:\*\* Rather than running Pi-hole on a dedicated device,

it runs as a Kubernetes Deployment. This demonstrates workload management,

namespace isolation, PodSecurity policy handling, and service exposure via

LoadBalancer — all on the same node.



\## Files in this repo



| File | Purpose |

|---|---|

| `controlplane-redacted.yaml` | Talos machine config with secrets removed |

| `metallb-pool.yaml` | MetalLB IP address pool and L2 advertisement |

| `pihole.yaml` | Pi-hole Deployment and LoadBalancer Services |

| `metallb-speaker-patch.yaml` | hostNetwork patch for MetalLB speaker |

| `pihole-dns-patch.yaml` | Annotation patch for DNS service IP sharing |

| `pihole-web-patch.yaml` | Annotation patch for web service IP sharing |



\## Deployment order



1\. Flash Talos ISO to USB, boot ThinkCentre, apply controlplane.yaml

2\. Bootstrap etcd: `talosctl bootstrap`

3\. Pull kubeconfig: `talosctl kubeconfig`

4\. Deploy MetalLB: `kubectl apply -f metallb-pool.yaml`

5\. Deploy Pi-hole: `kubectl apply -f pihole.yaml`

6\. Label pihole namespace privileged for PodSecurity

7\. Point router DNS at 192.168.1.200



\## Security+ domains evidenced



| Domain | Evidence |

|---|---|

| 2.0 Threats, Vulnerabilities and Mitigations | Network segmentation, DNS filtering |

| 3.0 Security Architecture | Defence-in-depth network design, infrastructure as code |

| 4.0 Security Operations | Service monitoring, log analysis via kubectl |



\## Skills demonstrated



\- Bare metal Linux deployment

\- Kubernetes cluster administration

\- Infrastructure as code (declarative YAML configuration)

\- Network services (DNS, DHCP interaction, L2 ARP)

\- Container workload management

\- Troubleshooting: certificate mismatches, nftables forwarding, PodSecurity policies



\## Next steps



\- Phase 2: Wazuh SIEM deployed in VMware Workstation Pro on MSI Stealth 16

\- Network monitoring lab: Zeek/Suricata on ThinkCentre

\- GitHub Actions: validate YAML manifests on push

