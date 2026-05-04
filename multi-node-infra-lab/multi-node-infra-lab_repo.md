
multi-node-infra-lab/

├── README.md

├── docs/

│   ├── architecture.md

│   ├── troubleshooting-report.md

│   ├── scenarios/

│   │   ├── scenario-01-icmp.md

│   │   ├── scenario-02-dnsmasq-port-53.md

│   │   ├── scenario-03-dns-resolution.md

│   │   ├── scenario-04-dns-unreachable.md

│   │   ├── scenario-05-dns-post-fix.md

│

├── configs/

│   ├── dns/

│   │   ├── dnsmasq.conf

│   │   ├── resolv.conf

│   │

│   ├── firewall/

│   │   ├── ufw-rules.txt

│   │   ├── iptables-rules.txt

│   │

│   ├── netplan/

│   │   ├── loadbalancer.yaml

│   │   ├── appserver.yaml

│   │   ├── dbserver.yaml

│

├── scripts/

│   ├── setup-dns.sh

│   ├── setup-firewall.sh

│   ├── setup-network.sh

│   ├── validate-lab.sh

│

├── diagrams/

│   ├── architecture.png

│   ├── architecture.drawio

│   ├── traffic-flow.png

│

└── logs/

│    ├── dns-debug.log
    
│    ├── firewall-debug.log
    

