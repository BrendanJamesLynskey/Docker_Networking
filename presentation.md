# Docker Networking Deep Dive

**Connecting Containers to the World**

`Networking` | `Bridge` | `Overlay` | `DNS`

Container → veth → Bridge → iptables → Host NIC

Understanding how Docker containers communicate — with each other, the host, and the outside world — through bridges, overlays, DNS, and network policies.

---

## 01 — Topics

### Network Drivers
- Bridge networks (default & user-defined)
- Host networking
- Overlay networks (Swarm / multi-host)
- Macvlan & IPvlan drivers

### Communication & DNS
- Container-to-container communication
- Docker embedded DNS server
- Port mapping & publishing
- Service discovery patterns

### Security & Isolation
- Network isolation fundamentals
- Inter-container firewall rules
- Encrypting overlay traffic
- Network policies & best practices

### Operations & Production
- Network commands cheat sheet
- Troubleshooting techniques
- Network plugins & CNI
- Production networking patterns

---

## 02 — Docker Networking Overview

Docker uses a pluggable networking architecture built on top of Linux kernel primitives. Every container gets its own **network namespace**, with virtual ethernet pairs connecting it to a network driver.

### Network Namespace
Each container has an isolated network stack: its own interfaces, routing table, iptables rules, and /proc/net entries.

### veth Pairs
Virtual ethernet device pairs act as tunnels between the container namespace and the host/bridge namespace.

### Network Drivers
Pluggable drivers (bridge, host, overlay, macvlan, none) determine how traffic is routed and isolated.

**Data path:** Container eth0 → veth pair → docker0 bridge → iptables NAT → eth0 (host)

```bash
# List all Docker networks
docker network ls

# Output:
# NETWORK ID     NAME      DRIVER    SCOPE
# a1b2c3d4e5f6   bridge    bridge    local
# f6e5d4c3b2a1   host      host      local
# 9a8b7c6d5e4f   none      null      local
```

---

## 03 — Default Bridge Network

Every Docker installation creates a default `bridge` network (the **docker0** interface). Containers connect here automatically unless you specify otherwise.

### How It Works
- Linux bridge `docker0` created at daemon start
- Default subnet: `172.17.0.0/16`
- Containers get IPs via IPAM (172.17.0.x)
- Host acts as gateway at `172.17.0.1`
- NAT via iptables for outbound traffic

### Limitations
- No automatic DNS resolution between containers
- Must use `--link` (deprecated) or IP addresses
- All containers share one bridge — no isolation
- Cannot be used with Docker Compose service names

```bash
# Run a container on the default bridge
docker run -d --name web nginx

# Inspect the network
docker network inspect bridge

# Check the container's IP
docker inspect -f \
  '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' \
  web
# 172.17.0.2

# Containers can ping by IP (not name)
docker exec web ping 172.17.0.3
```

```bash
# View the bridge on the host
ip addr show docker0
# docker0: <BROADCAST,MULTICAST,UP>
#   inet 172.17.0.1/16 scope global docker0

brctl show docker0
# bridge    interfaces
# docker0   veth3a1b2c
```

---

## 04 — User-Defined Bridge Networks

**User-defined bridges** are the recommended way to run containers. They provide automatic DNS, better isolation, and can be connected/disconnected on the fly.

### Advantages over Default Bridge
- **Automatic DNS:** containers resolve each other by name
- **Better isolation:** only containers on the same network communicate
- **Hot-pluggable:** connect/disconnect containers without restart
- **Custom subnets:** define your own IP ranges and gateways
- **Per-network config:** MTU, driver options, labels

```bash
# Create a user-defined bridge
docker network create my-app

# Run containers on it
docker run -d --name api \
  --network my-app node:20

docker run -d --name db \
  --network my-app postgres:16

# DNS works automatically!
docker exec api ping db
# PING db (172.18.0.3): 56 bytes

# Connect a running container
docker network connect my-app web

# Disconnect
docker network disconnect my-app web
```

```bash
# Custom subnet and gateway
docker network create \
  --subnet 10.10.0.0/24 \
  --gateway 10.10.0.1 \
  --ip-range 10.10.0.128/25 \
  my-custom-net
```

```yaml
# docker-compose.yml (automatic user-defined bridge)
services:
  web:
    image: nginx
    networks: [frontend]
  api:
    image: node:20
    networks: [frontend, backend]
  db:
    image: postgres:16
    networks: [backend]

networks:
  frontend:
  backend:
```

---

## 05 — Host Networking

With `--network host`, the container shares the host's network namespace directly. No network isolation, no NAT — the container **is** the host from a networking perspective.

### When to Use Host Mode
- Performance-critical apps (eliminates NAT overhead)
- Applications binding to many dynamic ports
- Network monitoring / packet capture tools
- Legacy apps that must bind to specific host interfaces

### Drawbacks
- No port isolation — port conflicts between containers
- Reduced security (full host network access)
- Only works on Linux (not Docker Desktop Mac/Win)
- Cannot use port mapping (`-p` flag is ignored)

```bash
# Run with host networking
docker run -d --network host --name monitor \
  nicolaka/netshoot

# The container sees all host interfaces
docker exec monitor ip addr
# 1: lo ...
# 2: eth0: 192.168.1.100/24
# 3: docker0: 172.17.0.1/16

# nginx on host mode binds directly to port 80
docker run -d --network host nginx
curl http://localhost:80  # Works!
```

```yaml
# docker-compose.yml
services:
  prometheus:
    image: prom/prometheus
    network_mode: host
    # No ports: mapping needed
```

---

## 06 — Overlay Networks

Overlay networks enable **multi-host communication** by encapsulating container traffic in VXLAN tunnels. Essential for Docker Swarm and multi-node deployments.

### How Overlay Works
- Uses VXLAN encapsulation (UDP port 4789)
- Creates a distributed virtual network across hosts
- Each node gets a VTEP (VXLAN Tunnel Endpoint)
- Built-in service discovery and load balancing
- Optional IPsec encryption with `--opt encrypted`

### Requirements
- Docker Swarm mode initialised (`docker swarm init`)
- Ports 2377 (mgmt), 7946 (gossip), 4789 (VXLAN) open
- All nodes must be able to reach each other
- Use `--attachable` for standalone container access

```bash
# Initialise Swarm
docker swarm init --advertise-addr 10.0.1.10

# Create an overlay network
docker network create \
  --driver overlay \
  --attachable \
  --opt encrypted \
  my-overlay

# Deploy a service across nodes
docker service create \
  --name web \
  --network my-overlay \
  --replicas 3 \
  nginx

# Standalone container on overlay
docker run -d --network my-overlay \
  --name debug nicolaka/netshoot
```

---

## 07 — Macvlan & IPvlan Drivers

These drivers assign **real IP addresses** from the physical network to containers, making them appear as physical hosts on the LAN. No NAT, no bridge.

### Macvlan
- Each container gets its own MAC address
- Appears as a distinct physical device on the network
- Requires the host NIC in promiscuous mode
- Best for: legacy apps needing direct LAN access
- Supports 802.1Q VLAN trunking

### IPvlan
- Containers share the parent's MAC address
- L2 mode: same subnet, like macvlan but single MAC
- L3 mode: routed, each container a separate subnet
- Works where promiscuous mode is blocked (cloud VMs)
- Lower overhead than macvlan

```bash
# Create a macvlan network
docker network create -d macvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  -o parent=eth0 \
  my-macvlan

# Run container with LAN IP
docker run -d --network my-macvlan \
  --ip 192.168.1.50 \
  --name lan-server nginx
```

```bash
# IPvlan L2 mode
docker network create -d ipvlan \
  --subnet 192.168.1.0/24 \
  --gateway 192.168.1.1 \
  -o parent=eth0 \
  -o ipvlan_mode=l2 \
  my-ipvlan

# IPvlan L3 mode (routed)
docker network create -d ipvlan \
  --subnet 10.10.0.0/24 \
  -o parent=eth0 \
  -o ipvlan_mode=l3 \
  my-ipvlan-l3
```

---

## 08 — Network Driver Comparison

| Driver | Scope | Container IP | DNS | Isolation | Best For |
|--------|-------|-------------|-----|-----------|----------|
| **bridge** (default) | Single host | Private (172.17.x.x) | No | Low | Quick testing |
| **bridge** (user-defined) | Single host | Private (custom) | Yes | Medium | Most workloads |
| **host** | Single host | Host IP | Host | None | Performance, monitoring |
| **overlay** | Multi-host | Private (10.0.x.x) | Yes | High | Swarm services |
| **macvlan** | Single host | LAN IP (unique MAC) | No | High | LAN-direct access |
| **ipvlan** | Single host | LAN IP (shared MAC) | No | High | Cloud VMs, L3 routing |
| **none** | Single host | None | No | Complete | Batch jobs, security |

**Rule of thumb:** Use user-defined bridges for single-host apps, overlay for multi-host, and host only when you need raw performance or host-level network access.

---

## 09 — DNS Resolution in Docker

Docker runs an **embedded DNS server** at `127.0.0.11` inside every user-defined network. This enables automatic service discovery by container name.

### How Docker DNS Works
- Embedded DNS at `127.0.0.11:53`
- Resolves container names → IP addresses
- Resolves network aliases and service names
- Falls back to host DNS for external domains
- **Only works on user-defined networks**

### Network Aliases
- Multiple containers can share an alias
- Docker round-robins DNS responses
- Useful for simple load balancing

```bash
# DNS in action
docker network create app-net

docker run -d --name db \
  --network app-net postgres:16

docker run -d --name api \
  --network app-net node:20

# Automatic resolution
docker exec api nslookup db
# Server:  127.0.0.11
# Name:    db
# Address: 172.19.0.2

docker exec api ping db
# PING db (172.19.0.2): 56 bytes
```

```bash
# Network aliases for round-robin
docker run -d --network my-app \
  --network-alias api worker1
docker run -d --network my-app \
  --network-alias api worker2
# "api" resolves to both IPs
```

```bash
# Custom DNS configuration
docker run -d \
  --dns 8.8.8.8 \
  --dns-search example.com \
  --hostname myapp.local \
  --name api my-image
```

---

## 10 — Port Mapping & Publishing

Port mapping uses **iptables DNAT rules** to forward traffic from a host port to a container port. This is how external clients reach containerised services.

| Flag | Meaning | Example |
|------|---------|---------|
| `-p 8080:80` | Host 8080 → Container 80 | All interfaces |
| `-p 127.0.0.1:8080:80` | Localhost only | Dev safety |
| `-p 8080:80/udp` | UDP protocol | DNS, QUIC |
| `-p 8080-8090:80-90` | Port range | Multi-port apps |
| `-P` | Publish all EXPOSE | Random host ports |

### Security Warning
Docker port mappings **bypass UFW/firewalld** by default. Use `127.0.0.1:` binding or configure `DOCKER_IPTABLES=false` and manage rules manually.

```bash
# Basic port mapping
docker run -d -p 8080:80 nginx

# Bind to localhost only (secure)
docker run -d -p 127.0.0.1:3000:3000 \
  my-api

# Multiple port mappings
docker run -d \
  -p 80:80 \
  -p 443:443 \
  -p 127.0.0.1:8443:8443 \
  my-proxy

# Check published ports
docker port my-proxy
# 80/tcp  -> 0.0.0.0:80
# 443/tcp -> 0.0.0.0:443
# 8443/tcp -> 127.0.0.1:8443
```

```yaml
# docker-compose.yml
services:
  web:
    image: nginx
    ports:
      - "80:80"
      - "127.0.0.1:8080:8080"
```

---

## 11 — Container-to-Container Communication

Containers on the **same user-defined network** can communicate freely using container names. Cross-network communication requires connecting containers to multiple networks.

### Same Network
- Direct communication via container name
- No port publishing needed
- All ports are accessible between containers
- Uses Docker's embedded DNS

### Cross-Network Communication
- Containers on different networks are isolated
- Connect container to multiple networks
- Use a "gateway" container on both networks
- Or route via published ports on the host

```yaml
# Multi-network Compose pattern
services:
  nginx:
    image: nginx
    networks: [frontend]
    ports: ["80:80"]

  api:
    image: node:20
    networks: [frontend, backend]
    # api talks to both nginx and db

  db:
    image: postgres:16
    networks: [backend]
    # db is isolated from nginx

networks:
  frontend:
    name: app-frontend
  backend:
    name: app-backend
    internal: true  # No external access
```

**Architecture:** nginx ↔ api ↔ db (frontend | backend internal)

---

## 12 — Network Isolation & Security

### Isolation Strategies
- **Separate networks:** each tier on its own network
- **Internal networks:** `--internal` blocks external access
- **No network:** `--network none` for complete isolation
- **ICC:** disable inter-container communication on bridge
- **Read-only:** combine with read-only filesystem

### Common Mistakes
- Running everything on the default bridge
- Exposing database ports to `0.0.0.0`
- Not using internal networks for backends
- Ignoring Docker's iptables bypass of host firewall

```bash
# Internal network (no external access)
docker network create --internal backend

# Disable inter-container communication
dockerd --icc=false

# No network at all
docker run --network none alpine

# Encrypt overlay traffic
docker network create --driver overlay \
  --opt encrypted secure-overlay
```

```json
// /etc/docker/daemon.json
{
  "icc": false,
  "iptables": true,
  "default-address-pools": [
    {"base": "10.10.0.0/16", "size": 24}
  ]
}
```

---

## 13 — Network Commands Cheat Sheet

| Command | Description |
|---------|-------------|
| `docker network ls` | List all networks |
| `docker network create <name>` | Create a bridge network (default driver) |
| `docker network create -d overlay <name>` | Create an overlay network |
| `docker network inspect <name>` | Show network details (subnet, containers, config) |
| `docker network connect <net> <container>` | Attach a running container to a network |
| `docker network disconnect <net> <container>` | Detach a container from a network |
| `docker network rm <name>` | Remove a network (must have no containers) |
| `docker network prune` | Remove all unused networks |
| `docker port <container>` | Show port mappings for a container |
| `docker inspect --format '...' <ctr>` | Extract networking info from a container |

```bash
# Useful inspect format strings
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' my-ctr
docker inspect -f '{{json .NetworkSettings.Ports}}' my-ctr | jq .
docker inspect -f '{{.NetworkSettings.Gateway}}' my-ctr
```

---

## 14 — Network Troubleshooting

When containers cannot communicate, use these tools and techniques to diagnose the problem systematically.

### The netshoot Toolkit
`nicolaka/netshoot` is a container packed with networking tools: curl, dig, nslookup, tcpdump, iperf, nmap, and more.

```bash
# Attach netshoot to a network
docker run --rm -it \
  --network my-app \
  nicolaka/netshoot

# Debug from inside a container's namespace
docker run --rm -it \
  --network container:my-api \
  nicolaka/netshoot

# Capture traffic on docker0
docker run --rm -it --net host \
  nicolaka/netshoot \
  tcpdump -i docker0 -nn port 80
```

### Troubleshooting Checklist
- Are containers on the **same network**? Check with `docker network inspect`
- Is it a **user-defined** network? (DNS needs it)
- Can you **ping** the target by IP? (rules out DNS)
- Is the **service listening** on the right port and interface (`0.0.0.0` not `127.0.0.1`)?
- Check **iptables** rules: `iptables -L -n -t nat`
- Check **docker logs** for connection errors
- Verify **firewall** allows Docker-managed ports

```bash
# Quick diagnostics from host
docker exec my-api cat /etc/resolv.conf
docker exec my-api nslookup db
docker exec my-api curl -v http://db:5432
iptables -L DOCKER -n -v
```

---

## 15 — Network Plugins & CNI

Docker supports third-party network plugins for advanced use cases. In Kubernetes, the **Container Network Interface (CNI)** standard replaces Docker's built-in networking.

### Calico
- L3 routing with BGP
- Advanced network policies
- High performance at scale
- Used in major K8s deployments

### Cilium
- eBPF-based networking
- L7 (HTTP/gRPC) policies
- Transparent encryption
- Observability with Hubble

### Flannel
- Simple overlay networking
- VXLAN or host-gw backend
- Minimal configuration
- Good for small clusters

### Weave Net
- Mesh overlay with encryption
- Automatic peer discovery
- DNS-based service discovery
- Multicast support

### Docker Plugin API
- Install: `docker plugin install <name>`
- Plugins implement the libnetwork remote driver API
- Vendors: Infoblox, Contiv, Kuryr (OpenStack)
- Moving toward CNI standard in most ecosystems

---

## 16 — Production Networking Patterns

### Reverse Proxy Pattern
- Traefik / Nginx / Caddy as edge proxy
- Only proxy exposes ports to the host
- Backend containers on internal networks
- TLS termination at the proxy
- Automatic service discovery via Docker labels

### Sidecar Pattern
- Share network namespace: `--network container:main`
- Log shippers, metrics collectors, service mesh proxies
- Containers communicate over `localhost`
- Used by Istio/Envoy, Datadog Agent

```yaml
# Reverse proxy with Traefik
services:
  traefik:
    image: traefik:v3.0
    ports: ["80:80", "443:443"]
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks: [proxy]

  api:
    image: my-api:latest
    labels:
      - "traefik.http.routers.api.rule=Host(`api.example.com`)"
    networks: [proxy, backend]

  db:
    image: postgres:16
    networks: [backend]

networks:
  proxy:
  backend:
    internal: true
```

```bash
# Sidecar: envoy shares api's network
docker run -d --name api my-api
docker run -d --name envoy \
  --network container:api \
  envoyproxy/envoy
```

---

## 17 — Summary & Further Reading

### Key Takeaways
- **Always** use user-defined bridge networks over the default
- Leverage Docker DNS — connect by name, not IP
- Isolate tiers with separate networks and `--internal`
- Bind ports to `127.0.0.1` in development
- Use overlay networks for multi-host deployments
- Macvlan/IPvlan for direct LAN integration
- Always consider Docker's iptables implications

### Decision Flowchart
- Single host, multiple containers? → **User-defined bridge**
- Multi-host cluster? → **Overlay**
- Need LAN IP? → **Macvlan/IPvlan**
- Max performance? → **Host**
- Total isolation? → **None**

### Further Reading
- [Docker Networking Documentation](https://docs.docker.com/network/)
- [Network Driver Overview](https://docs.docker.com/engine/network/drivers/)
- [Networking in Compose](https://docs.docker.com/compose/networking/)
- [netshoot: Docker Network Troubleshooting](https://github.com/nicolaka/netshoot)
- [Docker and iptables](https://docs.docker.com/engine/network/packet-filtering-firewalls/)

### Hands-On Exercises
- Create a multi-tier app with isolated networks
- Set up Traefik with automatic HTTPS
- Debug connectivity with netshoot
- Test overlay networking with Docker Swarm
- Configure macvlan for LAN-accessible containers
