    multi-node-infra-lab/
    │
    ├── README.md
    │
    ├── docs/
    │   ├── architecture.md
    │   ├── troubleshooting-report.md
    │   │
    │   └── scenarios/
    │       ├── 01-icmp-blocking.md
    │       ├── 02-dnsmasq-port-conflict.md
    │       ├── 03-dns-resolution-failure.md
    │       ├── 04-dns-server-unreachable.md
    │       └── 05-dns-recovery-validation.md
    │
    ├── configs/
    │   ├── dns/
    │   │   ├── dnsmasq.conf
    │   │   └── resolv.conf
    │   │
    │   ├── firewall/
    │   │   ├── ufw-rules.txt
    │   │   └── iptables-rules.txt
    │   │
    │   └── netplan/
    │       ├── lb-1.yaml
    │       ├── app-1.yaml
    │       └── db-1.yaml
    │
    ├── scripts/
    │   ├── setup-network.sh
    │   ├── setup-firewall.sh
    │   ├── setup-dns.sh
    │   └── validate-lab.sh
    │
    ├── diagrams/
    │   ├── architecture.png
    │   ├── architecture.drawio
    │   └── traffic-flow.png
    │
    ├── logs/
    │   ├── dns-debug.log
    │   └── firewall-debug.log
    │
    └── .gitignore

