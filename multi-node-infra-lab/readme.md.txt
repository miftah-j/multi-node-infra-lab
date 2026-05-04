# 🧪 Multi-Node Infrastructure Lab

A production-style simulated infrastructure environment built using Linux virtual machines to demonstrate:

- DNS-based service discovery
- Firewall-controlled communication
- Static IP architecture
- Multi-node networking (Load Balancer, App Server, DB Server)

---

## 🏗️ Architecture

- Load Balancer → 192.168.100.101  
- DNS / App Server → 192.168.100.102  
- Database Server → 192.168.100.103  

Internal DNS Domain: `*.lan`

---

## ⚙️ Features

- Centralized DNS using dnsmasq
- UFW-based firewall segmentation
- Static IP-based service mapping
- Internal service discovery system
- Real-world troubleshooting scenarios

---

## 🚀 Quick Setup

```bash
bash scripts/setup-network.sh
bash scripts/setup-dns.sh
bash scripts/setup-firewall.sh