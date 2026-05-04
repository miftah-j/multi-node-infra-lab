## **🚀** **System Engineering Lab**

### **Linux Networking, Access Control & DNS**

**📌** **Overview**

This lab simulates a production-style environment with multiple Linux nodes communicating over a private network.

Focus:

-   System connectivity
-   Secure access control
-   Firewall enforcement
-   DNS resolution
-   Failure diagnosis & recovery

Applications are treated as black boxes. The emphasis is on **operating and troubleshooting infrastructure**.

**🧱 Architecture**

		            [ lb-1 ]  
                 192.168.100.101  
		               |  
              ---------------------  
              |                   |  
          [ app-1 ]            [ db-1 ]  
       192.168.100.102      192.168.100.103

**⚙️** **Core Setup**

**Static Networking (Netplan)**

``sudo nano /etc/netplan/01-netcfg.yaml``

    sudo nano /etc/netplan/01-netcfg.yaml
    network:
      version: 2
      ethernets:
        enp0s3:
          addresses: [192.168.100.101/24]
          gateway4: 192.168.100.1
          nameservers:
            addresses: [8.8.8.8]


``sudo netplan apply``

**SSH Hardening**

    ssh-keygen -t rsa  
    ssh-copy-id user@192.168.100.101

``sudo nano /etc/ssh/sshd_config``

    PermitRootLogin no  
    PasswordAuthentication no

``sudo systemctl restart ssh``

**Firewall (UFW)**

    sudo apt install ufw -y  
    sudo ufw default deny incoming  
    sudo ufw allow ssh  
    sudo ufw enable

**Local DNS (dnsmasq)**

    sudo apt install dnsmasq -y
    sudo nano /etc/dnsmasq.conf
    
   
   ``address=/app1.lan/192.168.100.102``
   
``sudo systemctl restart dnsmasq``

**✅** **Validation**

    ping 192.168.100.102  
    ping 192.168.100.103  
    dig app.local  
    sudo ufw status

**🔥** **Failure Scenarios Drills (Real Engineering Signal)**

**Network Issue**

ping 8.8.8.8  # fails

**Diagnosis**

    ip a
    ip route

**Fix -**

-   Correct gateway
-   Reapply netplan

**SSH Lockout**

``sudo ufw deny ssh``

**Fix (via console)**

Fix /etc/ssh/sshd_config

``sudo ufw allow ssh``

**Internal Network Blocked**

**Break**

``sudo ufw deny from 192.168.100.0/24``

**Fix**

``sudo ufw delete deny from 192.168.100.0/24``

**DNS Not Resolving**

Stop dnsmasq

``sudo systemctl stop dnsmasq``

    dig app.local  # fails  
    ping app.local  # fails

**Diagnosis**

``systemctl status dnsmasq``

**Fix**

``sudo systemctl restart dnsmasq``

----------

**🧠 Engineering Insights**

-   Connectivity ≠ “ping works” → it’s routing + firewall + DNS
-   SSH hardening can create a total lockout if not tested properly
-   Firewall rules define system reachability boundaries
-   DNS failures often appear as application failures

**📁** **Repository Structure**

    system-engineering-lab/  
    │  
    ├── README.md  
    ├── configs/  
    │  ├── netplan.yaml  
    │  ├── sshd_config  
    │  └── dnsmasq.conf  
    │  
    ├── troubleshooting/  
    │  └── failure-scenarios.md  
    │  
    └── diagrams/  
    └── architecture.png

**📊** **What This Demonstrates**

-   Ability to configure Linux systems in a networked environment
-   Understanding of access control and security hardening
-   Practical firewall management
-   Real-world troubleshooting under failure conditions
