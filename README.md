# 🔌 Docker Networking

An interactive Reveal.js presentation on Docker networking — bridge, host, overlay, macvlan drivers, DNS resolution, port mapping, network isolation, and production patterns.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/Docker_Networking/)

## 📄 [Markdown Version](presentation.md)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Topics | Drivers, communication, security, operations |
| 02 | Docker Networking Overview | Namespaces, veth pairs, pluggable drivers |
| 03 | Default Bridge Network | docker0 interface, 172.17.0.0/16, limitations |
| 04 | User-Defined Bridge Networks | Automatic DNS and per-network isolation |
| 05 | Host Networking | Sharing the host network namespace directly |
| 06 | Overlay Networks | Multi-host communication via VXLAN tunnels |
| 07 | Macvlan & IPvlan Drivers | Assigning real LAN IPs to containers |
| 08 | Network Driver Comparison | Scope, IP, DNS, isolation, best fit |
| 09 | DNS Resolution in Docker | Embedded DNS at 127.0.0.11 and aliases |
| 10 | Port Mapping & Publishing | iptables DNAT, -p flag variants |
| 11 | Container-to-Container Communication | Same-network and cross-network patterns |
| 12 | Network Isolation & Security | Internal networks, ICC, encryption |
| 13 | Network Commands Cheat Sheet | ls, create, inspect, connect, prune |
| 14 | Network Troubleshooting | netshoot toolkit and diagnostic checklist |
| 15 | Network Plugins & CNI | Calico, Cilium, Flannel, Weave |
| 16 | Production Networking Patterns | Reverse proxy and sidecar patterns |
| 17 | Summary & Further Reading | Key takeaways and decision flowchart |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## References

[Docker Networking Documentation](https://docs.docker.com/network/) · [Network Drivers Overview](https://docs.docker.com/engine/network/drivers/) · [Compose Networking](https://docs.docker.com/compose/networking/) · [Docker and iptables](https://docs.docker.com/engine/network/packet-filtering-firewalls/)

## License

Educational use. Code examples provided as-is.
