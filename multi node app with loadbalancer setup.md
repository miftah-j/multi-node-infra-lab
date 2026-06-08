
## The Multi App Infra Lab step by Step

**What I did , what problem I faced all the troubleshooting and the fixations**


## Lab Setup (Your “Data Center”)

3 machines that can talk to each other.

Create 3 VMs:

-   lloadbalance1 → 192.168.100.101
-   appserver1 → 192.168.100.102
-   database1 → 192.168.100.103

**Commands**

Shown for loadbalancer ; same procedure for the other vms

```hostnamectl set-hostname loadbalancer1```

Set static IP (Ubuntu example):

```sudo nano /etc/netplan/01-netcfg.yaml```

```
network:
  version: 2
  renderer: networkd
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.100.101/24
      routes:
        - to: default
          via: 192.168.100.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
```

```sudo netplan apply```

Tested all the server can communicate with each other.

## Access Control (SSH)

**On the base machine**

    ssh-keygen -t rsa  
    ssh-copy-id user@192.168.100.101

**Disable password login:**

```sudo nano /etc/ssh/sshd_config```

Set:
> PermitRootLogin no   
> PasswordAuthentication no

```sudo systemctl restart ssh```

##  Firewall Thinking

On appserver1:

    sudo apt install ufw -y  
    sudo ufw default deny incoming  
    sudo ufw allow ssh  
    sudo ufw enable


##  Networking Between VMs

```ping 192.168.100.101``` from app server

**Broke It**

```sudo ufw deny from 192.168.100.0/24```


```sudo ufw delete deny from 192.168.56.0/24```

This is where the first issue occured

Ping (ICMP) traffic was still allowed from the same subnet.


**Root Cause**

-   UFW allows ICMP by default via /etc/ufw/before.rules
-   ICMP ACCEPT rules were placed **before** user-defined rules
-   iptables processes rules **top-down (first match wins)**


Example: ufw status showed

> ACCEPT icmp -- 0.0.0.0/0 icmptype 8
> DROP  icmp -- 192.168.100.0/24 icmptype 8


👉 Traffic matched ACCEPT first, so DROP never applied.


**Persistent Fix (UFW)**

Edit:
``sudo nano /etc/ufw/before.rules``

Add BEFORE existing ICMP ACCEPT rules:

    -A ufw-before-input -p icmp --icmp-type echo-request -s 192.168.100.0/24 -j DROP

`sudo ufw reload`

## DNS Setup (Local Resolution)

    sudo apt install dnsmasq -y  
    sudo nano /etc/dnsmasq.conf

Add:
```address=/app.lan/192.168.56.11```

```sudo systemctl restart dnsmasq```


```systemctl status dnsmasq  ```
```dig app.local```


### **Issues**

**Issue 1 — dnsmasq failed to start**
failed to create listening socket for port 53: Address already in use

**Issue 2 — DNS resolution failure**
ping app.lan → Name or service not known

**Issue 3 — Missing tools**
dig / nslookup not found


### **Root Causes**

**Port Conflict** -   systemd-resolved was using port 53 (127.0.0.53)

**Resolver Misconfiguration**
-   /etc/resolv.conf pointed to external DNS (8.8.8.8)
-   dnsmasq was not being used

**Minimal System** -   DNS tools not installed


### **Solutions**

**Step 1 — Install tools**
sudo apt install dnsutils -y

**Step 2 — Disable systemd-resolved**
sudo systemctl disable systemd-resolved
sudo systemctl stop systemd-resolved

**Fix resolver:**
sudo rm /etc/resolv.conf
echo "nameserver 192.168.100.102" | sudo tee /etc/resolv.conf


**Step 3 — Configure dnsmasq (on App Server)**

``sudo nano /etc/dnsmasq.conf``

    #  
    === DNSMASQ LAB CONFIG ===
    
    domain-needed
    bogus-priv
    
    server=8.8.8.8
    server=1.1.1.1
    
    local=/lan/
    
    address=/lb.lan/192.168.100.101
    address=/app.lan/192.168.100.102
    address=/db.lan/192.168.100.103
    
    listen-address=127.0.0.1
    listen-address=192.168.100.102
    
    local-ttl=30
    log-queries




# === DNSMASQ LAB CONFIG ===

# Basic sanity
domain-needed
bogus-priv

# Upstream DNS
server=8.8.8.8
server=1.1.1.1

# Local domain
local=/lan/

# Static DNS records

address=/lb.lan/192.168.100.101
address=/app1.lan/192.168.100.102
address=/db.lan/192.168.100.103
address=/app2.lan/192.168.100.104

# Optional: multiple app nodes (simulate scaling)
# address=/app2.lan/192.168.100.104

# Optional round-robin entry
address=/app.lan/192.168.100.102
address=/app.lan/192.168.100.104


# Bind safely
listen-address=127.0.0.1
listen-address=192.168.100.102

# TTL for fast testing
local-ttl=30

# Logging (important for debugging)
log-queries
log-facility=/var/log/dnsmasq.log



**Step 4 — Restart dnsmasq**
``sudo systemctl restart dnsmasq``

**Step 5 — Configure all servers**
On Load Balancer and DB Server:
``echo "nameserver 192.168.100.102" | sudo tee /etc/resolv.conf``

**Validation**

    dig app.lan @127.0.0.1
    ping app.lan
    ping db.lan
    ping lb.lan

Expected:
-   All hostnames resolve correctly
-   Inter-server communication works

DNS only works when **clients actually use the intended resolver**



But **DNS resolution failed from client nodes**

From Load Balancer:
> ping app1.lan → Temporary failure in name resolution

**Root Cause**

-   Clients were still using external DNS (8.8.8.8)
-   Internal DNS server was not consistently applied across nodes
-   Inconsistent resolver configuration

**Resolution**

-   Updated /etc/resolv.conf on all nodes
-   Forced all servers to use internal DNS: 192.168.100.102
-   Removed systemd-resolved interference
-   Standardized DNS dependency on internal resolver

All nodes successfully resolved internal domain names.


### **🚨**  ** DNS Server Reachability Failure (Port 53 Timeout)**

``dig app1.lan @192.168.100.102``  

> ;; communications error: timed out

**🔍** **Root Cause**

-   UFW firewall blocking UDP/TCP port 53
-   Incorrect deny rules applied to subnet

The firewall rules we applied backfired. System was reaching the app server. But firewall dropped the request.

**Removed incorrect rules:**

    sudo ufw delete deny from 192.168.100.0/24  

**Allowed DNS traffic:**

    sudo ufw allow from 192.168.100.0/24 to any port 53 proto udp  
    sudo ufw allow from 192.168.100.0/24 to any port 53 proto tcp  
    sudo ufw reload

-   DNS server became reachable from all nodes
-   Query flow restored

**Test from each server**

ping app.lan  
ping db.lan  
ping lb.lan





## Role-based configs (real system behavior)**

Now we go beyond DNS.

**🟢 LOAD BALANCER (lb.lan)**

``sudo apt install nginx -y``

Basic config:

``sudo nano /etc/nginx/sites-available/default``


    upstream app_backend {  
    server app.lan:80;  
    }  
      
    server {  
    listen 80;  
      
    location / {  
    proxy_pass http://app_backend;  
    }  
    }  

   or

    server { 
    listen 80;
    server_name lb.lan;
    
    location / {
    proxy_pass http://192.168.100.102;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    
    }
    
    }

Restart:

sudo systemctl restart nginx


**🔵** **APP SERVER (app.lan)**

Install:

``sudo apt install nginx -y``

Simple test page:

``echo "APP SERVER" | sudo tee /var/www/html/index.html``


**🟣 DB SERVER (db.lan)**

we can simulate:
``sudo apt install mysql-server -y``

Or just test connectivity:
``ping db.lan``

----------

 **End-to-end test (this is where it becomes real)**

From any machine:
``curl http://lb.lan``

 Flow:

> Client → DNS → lb.lan → nginx → app.lan → response


But I was not satisfied with the db test. So I decided to take it furthermore.

**Installed MySQL on DB Server (192.168.100.103)**

    sudo apt update  
    sudo apt install mysql-server -y
    
``sudo systemctl enable mysql``
``sudo systemctl start mysql``

Check status:
``sudo systemctl status mysql``

**Secured MySQL (important even in lab)**

``sudo mysql_secure_installation``

**Recommended answers:**
> -   VALIDATE PASSWORD: optional (lab → can skip strict)
> -   Remove anonymous users: ✔ yes
> -   Disallow root remote login: ✔ yes
> -   Remove test DB: ✔ yes
> -   Reload privileges: ✔ yes

**🌐** **STEP 3 — Enable remote access (CRITICAL STEP)**

Edit MySQL config:

``sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf``

Find:
> bind-address = 127.0.0.1

Change to:
> bind-address = 0.0.0.0

Restart MySQL:
``sudo systemctl restart mysql``

**Created test database user**

Login:
``sudo mysql``

Run:

    CREATE DATABASE labdb;  
    CREATE USER 'labuser'@'192.168.100.%' IDENTIFIED BY 'Lab@1234';  
    GRANT ALL PRIVILEGES ON labdb.* TO 'labuser'@'192.168.100.%';  
    FLUSH PRIVILEGES;  
    EXIT;

**Firewall rules applied**
Allow only app/loadbalancer access:

    sudo ufw allow from 192.168.100.101 to any port 3306 proto tcp  
    sudo ufw allow from 192.168.100.102 to any port 3306 proto tcp  
    sudo ufw reload

**Tested MySQL connectivity (REAL VALIDATION)**

Install client on app server:
``sudo apt install mysql-client -y``

Connect:
``mysql -h db.lan -u labuser -p``

Enter password:
``Lab@1234``

> Result: Welcome to the MySQL monitor


## Loadbalancing test

Cloned the Appserver1 VM to Appserver2 

Changed IP through Netplan

**Firewall rules hampered ssh from base machine**

Added the base machine ip to ssh through port 22

Faced a local hostname resolution failure

sudo: unable to resolve host appserver2: Name or service not known

### Troubleshooting
hostname
cat /etc/hosts

Fix (Edit hosts file):

sudo hostnamectl set-hostname appserver2

or
sudo nano /etc/hosts
127.0.1.1 appserver2




I thought Cloning Will help me to resolve the DNS easily But I was wrong


### FIX — Single Source of Truth

Use ONLY one DNS server

-   appserver1 = DNS master
-   appserver2 = ONLY app node

 On appserver2:
```
sudo systemctl disable dnsmasqsudo 
systemctl stop dnsmasq

**✅** **Fix resolver (on App server 2)**

    sudo systemctl disable systemd-resolved  
    sudo systemctl stop systemd-resolved 

    sudo rm /etc/resolv.conf  
    echo "nameserver 192.168.100.102" | sudo tee /etc/resolv.conf
----------

**🔍** **Verify**

``cat /etc/resolv.conf``

Must show:
> nameserver 192.168.100.102



**ULTIMATE NGINX LOAD BALANCER CONFIG**

Edit on **Load Balancer (192.168.100.101)**:
``sudo nano /etc/nginx/sites-available/default``


**✅** **FINAL CONFIG (production-style)**

```
# Upstream backend cluster

```
upstream app_cluster {
    server 192.168.100.102 max_fails=3 fail_timeout=10s;
    server 192.168.100.103 max_fails=3 fail_timeout=10s;
}

server {
    listen 80;
    server_name lb.lan;

    # Logging (VERY IMPORTANT for debugging)
    access_log /var/log/nginx/lb_access.log;
    error_log /var/log/nginx/lb_error.log;

    location / {
        proxy_pass http://app_cluster;

        # Preserve client identity
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Stability tuning (prevents hanging like you saw earlier)
        proxy_connect_timeout 3s;
        proxy_send_timeout 10s;
        proxy_read_timeout 10s;

        # Avoid buffering confusion during lab testing
        proxy_buffering off;
    }
}
```


** Why this is “ultimate” **

**1. Real load balancing (NOT DNS guessing)**
-Nginx distributes traffic at request level

**2. Built-in failure handling**
max_fails=3 fail_timeout=10s;
→ automatically stops sending traffic to broken node

**3. Proper client traceability**
X-Forwarded-For
→ You can now see real user IPs in backend logs
**4. Debug visibility (critical for labs)**
/var/log/nginx/lb_error.log
**5. Prevents your earlier “hang issue”**
These 2 lines fix your previous pain:

proxy_connect_timeout 3s;  
proxy_read_timeout 10s;

* **Apply configuration**

    sudo nginx -t  
    sudo systemctl reload nginx

----------

**🧪 TESTING (must do in order)**

**1. Check upstream directly**

curl http://192.168.100.102  
curl http://192.168.100.104

**2. Test LB**

curl http://lb.lan  
curl http://lb.lan  
curl http://lb.lan

**3. Validate distribution**

You should see alternating responses:

-   APP SERVER 1
-   APP SERVER 2

----------

Initially we were focussing on the app.lan

but now the server runs on lb.lan


But After a System reboot I was getting only response from only App Server 1

Started troubleshooting & Found the Appserver 2 (The cloned VM) still holding the old static IP active in runtime state or another persistent config file is applying it.

Though We changed:
192.168.100.102 → 192.168.100.104
inside one Netplan file, BUT:
•	another Netplan file still contains .102
OR 
•	runtime interface state still has .102
OR 
•	cloud-init regenerated old config during boot 

1. Check ALL Netplan Files
Run:
ls -l /etc/netplan/
2. Search for OLD IP Everywhere
This is the MOST IMPORTANT command now:
sudo grep -R "192.168.100.102" /etc

3. Check Generated Runtime Config
Run:
cat /run/systemd/network/10-netplan-ens33.network
If you still see:
192.168.100.102
then Netplan generation is still pulling old config from somewhere.
 4. Check for Cloud-Init (VERY COMMON IN CLONED VMs)
Run:
cloud-init status
If installed, check:
ls /etc/cloud/cloud.cfg.d/
Cloud-init may regenerate old network config at boot.
5. Disable Cloud-Init Network Management (if needed)
Create:
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
Put:
network: {config: disabled}

6. Delete Old Generated Netplan State
sudo rm -rf /run/systemd/network/*

 7. Apply Clean Config
Now use ONLY ONE Netplan file:
Example:

`` sudo nano /etc/netplan/01-network-manager-all.yaml``

    network:
      version: 2
      renderer: networkd
    
      ethernets:
        ens33:
          dhcp4: no
          addresses:
            - 192.168.100.104/24
    
          routes:
            - to: default
              via: 192.168.100.1
    
          nameservers:
            addresses:
              - 8.8.8.8
              - 1.1.1.1

 8. Regenerate Everything
sudo netplan generate
sudo netplan apply
sudo systemctl restart systemd-networkd

 9. Reboot
sudo reboot


🧾 FINAL FIX (Exact Steps)
________________________________________
🔴 STEP 1 — Edit the REAL Active Config
Open:
sudo nano /etc/cloud/cloud.cfg.d/90-installer-network.cfg

Final clean config should look like:
# This is the network config written by 'subiquity'

    network:
      version: 2
    
      ethernets:
        ens33:
          addresses:
            - 192.168.100.104/24
    
          routes:
            - to: default
              via: 192.168.100.1
    
          nameservers:
            addresses:
              - 8.8.8.8
              - 4.2.2.2

🔴 STEP 3 — Apply Netplan
Now regenerate:
sudo netplan generate
sudo netplan apply

🔴 STEP 4 — Restart Networking
sudo systemctl restart systemd-networkd

🔴 STEP 5 — Verify
ip a
Expected:
192.168.100.104
NOT:
192.168.100.102

🔴 STEP 6 — Reboot Validation
sudo reboot
After reboot:
ip a
The .104 IP should persist permanently.

🧠 WHY THIS HAPPENED
Ubuntu Server installer created:
90-installer-network.cfg
And cloud-init/subiquity treats it as authoritative.
So:
•	editing another Netplan file had no effect 
•	reboot regenerated old network state 

🔥 Optional (Production-Grade Cleanup)
If this is a pure lab VM and you don’t want cloud-init managing networking anymore:
Create:
sudo nano /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg
Put:
network: {config: disabled}
Then manage networking ONLY via /etc/netplan/*.yaml.

