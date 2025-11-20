# Networking Cheat Sheet for SRE/Network Operations

Quick reference for networking concepts, troubleshooting commands, and interview talking points.

---

## 1. Load Balancing Concepts

### Types of Load Balancing

**Layer 4 (Transport Layer)**
- Balances based on: IP address, port
- Fast, simple decision
- No visibility into application data
- Examples: HAProxy (TCP mode), NetScaler (basic LB)

**Layer 7 (Application Layer)**
- Balances based on: HTTP headers, cookies, URL path, content
- Can route to different backends based on content
- SSL termination possible
- Examples: HAProxy (HTTP mode), NetScaler (content switching), NGINX

### Load Balancing Algorithms

| Algorithm | How it Works | Use Case |
|-----------|-------------|----------|
| Round Robin | Distributes requests sequentially | Equal capacity servers |
| Least Connections | Sends to server with fewest active connections | Varying request duration |
| IP Hash | Hash source IP to determine backend | Session persistence without cookies |
| Weighted Round Robin | Like round robin but with capacity weighting | Mixed server capacities |
| Least Response Time | Sends to fastest responding server | Performance-sensitive apps |

### Health Checks

**Active Health Checks:**
- LB actively probes backend servers
- HTTP GET, TCP connect, custom scripts
- Determines if server is healthy

**Passive Health Checks:**
- Monitor actual traffic for failures
- Faster detection of issues
- Combined with active for best results

**Example Health Check Types:**
```bash
# HTTP health check
curl -f http://backend:8080/health

# TCP health check
nc -zv backend 8080

# Custom script
/usr/local/bin/check_app.sh
```

---

## 2. BGP Fundamentals

### BGP Basics

**What is BGP?**
- Border Gateway Protocol
- Path-vector routing protocol
- Routes traffic between Autonomous Systems (AS)
- Used for internet routing and data center fabrics

**Key Concepts:**
- **AS Number:** Unique identifier for network (e.g., AS 65001)
- **BGP Peer/Neighbor:** Router you exchange routes with
- **eBGP:** Between different AS numbers
- **iBGP:** Within same AS number
- **Route Advertisement:** Announcing networks you can reach

### BGP Session States

```
Idle → Connect → OpenSent → OpenConfirm → Established
```

| State | Meaning |
|-------|---------|
| Idle | No peering attempt |
| Connect | TCP connection attempt |
| OpenSent | Sent OPEN message, waiting for response |
| OpenConfirm | Received OPEN, sent KEEPALIVE |
| Established | Peering up, exchanging routes ✅ |

### eBGP vs iBGP

| Feature | eBGP | iBGP |
|---------|------|------|
| AS relationship | Different AS | Same AS |
| Next-hop | Changed to self | Preserved |
| TTL | 1 (directly connected) | 255 (multi-hop) |
| Use case | Between organizations | Within organization |
| Policy requirement | Yes (FRR default) | No |

**Key Difference for Troubleshooting:**
- eBGP: Changes next-hop to itself
- iBGP: Preserves original next-hop (can cause unreachable routes!)

### Common BGP Issues

**Routes not exchanged:**
- Neighbor not activated in address-family
- eBGP policy missing (`no bgp ebgp-requires-policy`)
- Routes not being advertised (`network` statement missing)

**Session won't establish:**
- AS number mismatch
- Firewall blocking TCP 179
- IP address typo in config
- Wrong remote-as configured

**Routes in BGP table but not routing table:**
- iBGP next-hop unreachable (need `next-hop-self` or IGP)
- Better route from another protocol
- Route rejected by filter/policy

---

## 3. TCP/IP Troubleshooting Commands

### Connectivity Testing

```bash
# Basic reachability
ping 8.8.8.8
ping -c 4 google.com

# Trace route path
traceroute 8.8.8.8
mtr 8.8.8.8  # Better than traceroute, shows packet loss

# DNS resolution
nslookup google.com
dig google.com
host google.com

# Test specific port
telnet hostname 80
nc -zv hostname 443  # Netcat port check
curl -v http://hostname:8080
```

### Network Interface Commands

```bash
# View interfaces
ip addr show
ip link show
ifconfig  # Older, but still common

# View routing table
ip route show
route -n
netstat -rn

# View ARP table (IP to MAC mapping)
ip neigh show
arp -a

# Interface statistics
ip -s link show eth0
ifconfig eth0

# Bring interface up/down
sudo ip link set eth0 up
sudo ip link set eth0 down
```

### Network Statistics

```bash
# Active connections
ss -tunap  # Modern, faster
netstat -tunap  # Older but still widely used

# Listening ports
ss -tlnp
netstat -tlnp
lsof -i -P -n  # List open files/ports

# Connection counts by state
ss -ant | awk '{print $1}' | sort | uniq -c

# Top connections
netstat -ntu | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -n
```

### Packet Capture

```bash
# Capture on interface
sudo tcpdump -i eth0
sudo tcpdump -i any  # All interfaces

# Capture specific traffic
sudo tcpdump -i eth0 port 80
sudo tcpdump -i eth0 host 192.168.1.10
sudo tcpdump -i eth0 tcp and port 443

# Save to file
sudo tcpdump -i eth0 -w capture.pcap

# Read from file
tcpdump -r capture.pcap

# Verbose output with timestamps
sudo tcpdump -i eth0 -tttt -vv

# Common filter examples
sudo tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0'  # SYN packets
sudo tcpdump -i eth0 icmp  # Ping packets
```

### Firewall/iptables

```bash
# List rules
sudo iptables -L -n -v
sudo iptables -t nat -L -n -v  # NAT table

# Check if port is blocked
sudo iptables -L -n | grep <port>

# Temporarily allow port (for testing)
sudo iptables -I INPUT -p tcp --dport 8080 -j ACCEPT

# View firewalld status (RHEL/CentOS)
sudo firewall-cmd --list-all
sudo firewall-cmd --state
```

---

## 4. TCP/IP Troubleshooting Flow

### Layer-by-Layer Approach

**Start at Layer 3 (Network), work up:**

**1. Can you ping?**
```bash
ping <destination>
```
- Yes → Layer 3 works, move to Layer 4
- No → Check routing, IP config, firewall

**2. Can you reach the port?**
```bash
telnet <destination> <port>
nc -zv <destination> <port>
```
- Yes → Port is open, move to Layer 7
- No → Check firewall, service not listening

**3. Is the service responding correctly?**
```bash
curl -v http://<destination>:<port>
```
- Check HTTP response codes
- Check response time
- Check response content

**4. DNS resolution working?**
```bash
nslookup <hostname>
dig <hostname>
```

### Common Troubleshooting Scenarios

**Scenario: Can't connect to web service**

```bash
# Step 1: Can you ping the host?
ping webserver.example.com

# Step 2: Can you reach the IP directly?
ping 10.1.2.3

# Step 3: Is DNS working?
nslookup webserver.example.com

# Step 4: Is the port open?
telnet 10.1.2.3 80
nc -zv 10.1.2.3 80

# Step 5: What's the TCP handshake showing?
sudo tcpdump -i eth0 -n host 10.1.2.3 and port 80

# Step 6: Is the service listening?
# (On the server)
ss -tlnp | grep :80
```

**Scenario: Slow network performance**

```bash
# Check for packet loss
ping -c 100 destination
mtr destination  # Shows loss per hop

# Check bandwidth
iperf3 -c destination  # Requires iperf3 server running

# Check interface errors
ip -s link show eth0

# Look for retransmits
ss -ti  # Shows TCP info including retransmits

# Check MTU issues
ping -M do -s 1472 destination  # Test if fragmentation needed
```

**Scenario: Intermittent connectivity**

```bash
# Continuous ping with timestamps
ping destination | while read line; do echo "$(date): $line"; done

# Monitor connection state changes
watch -n 1 'ss -tn | grep destination'

# Check for interface flapping
dmesg -T | grep eth0

# Monitor DNS resolution
watch -n 5 'dig +short destination'
```

---

## 5. Network Debugging Tools

### Essential Tools

| Tool | Purpose | Example |
|------|---------|---------|
| `ping` | Basic connectivity | `ping 8.8.8.8` |
| `traceroute` | Path to destination | `traceroute google.com` |
| `mtr` | Better traceroute | `mtr google.com` |
| `ss` | Socket statistics | `ss -tunap` |
| `netstat` | Network stats (older) | `netstat -tunap` |
| `tcpdump` | Packet capture | `tcpdump -i eth0 port 80` |
| `nmap` | Port scanning | `nmap -p 1-1000 host` |
| `curl` | HTTP testing | `curl -v http://example.com` |
| `dig` | DNS queries | `dig google.com` |
| `iperf3` | Bandwidth testing | `iperf3 -c server` |

### Quick Network Health Check

```bash
#!/bin/bash
# Quick network health script

echo "=== Network Interfaces ==="
ip addr show

echo -e "\n=== Routing Table ==="
ip route show

echo -e "\n=== DNS Resolution ==="
dig +short google.com

echo -e "\n=== Internet Connectivity ==="
ping -c 3 8.8.8.8

echo -e "\n=== Listening Ports ==="
ss -tlnp

echo -e "\n=== Active Connections ==="
ss -tn | wc -l

echo -e "\n=== Interface Stats ==="
ip -s link show
```

---

## 6. Load Balancing Troubleshooting

### Common Load Balancer Issues

**Backend servers not receiving traffic:**
```bash
# Check health check status
curl http://lb-ip/health-status

# Check backend connectivity from LB
# (from LB host)
curl http://backend-ip:8080/health

# Verify backend is listening
# (from backend)
ss -tlnp | grep :8080

# Check LB logs
tail -f /var/log/haproxy.log
# or for NetScaler
ssh nsroot@netscaler "show lb vserver <name>"
```

**Session persistence not working:**
```bash
# Check if cookies are being set
curl -v http://lb-ip | grep Set-Cookie

# Check IP hash consistency
# Multiple requests should hit same backend
for i in {1..10}; do curl -s http://lb-ip | grep server; done

# Check LB config for persistence method
# HAProxy: "stick-table" or "cookie"
# NetScaler: "persistenceType"
```

**Uneven traffic distribution:**
```bash
# Check backend connection counts
for backend in backend1 backend2 backend3; do
  echo -n "$backend: "
  ss -tn | grep $backend | wc -l
done

# Check LB weights/ratios
# Are all backends weighted equally?

# Check if one backend is slow (causing backup)
# Slow backends get fewer connections with least-conn
```

---

## 7. Interview Preparation - Common Questions

### Load Balancing Experience

**Question:** "Tell me about your load balancing experience."

**Key Points to Cover:**
- Types of load balancers managed (hardware/software, L4/L7)
- Scale of deployment (number of instances, traffic volume)
- Automation and configuration management approaches
- Health check strategies and failover mechanisms
- Challenges faced and solutions implemented

**Production Experience Topics:**
- Health checks are critical - passive + active gives best results
- Session persistence needs to match application requirements
- BGP failover timing - balance between fast failover and stability
- At scale, automation is essential - manual changes don't scale
- Monitoring and alerting for load balancer health

### BGP Experience

**Question:** "What's your BGP experience?"

**Key Concepts to Discuss:**
- eBGP vs iBGP differences and use cases
- BGP states and peering troubleshooting
- Route advertisement and withdrawal
- Common BGP issues (AS mismatch, policy requirements, next-hop)
- BGP in production (if applicable): failover, traffic engineering, multi-site

**Lab/Learning Experience:**
- Docker-based BGP lab for practicing scenarios
- AS number mismatches and debugging
- iBGP next-hop issues and solutions
- FRRouting or similar open-source BGP implementations

### Troubleshooting Methodology

**Question:** "Walk me through troubleshooting a network connectivity issue."

**Layer-by-Layer Approach:**

**Layer 3 (Network):** 
- Can I ping? If no, check routing, IP config, DNS

**Layer 4 (Transport):** 
- Can I reach the port? Use telnet or nc
- If no, check firewall and if service is listening

**Layer 7 (Application):** 
- Is the service responding correctly? 
- Use curl to test actual application response

**Additional Checks:**
- Recent changes (deployments, config changes)
- Monitoring/alerts for related issues
- Logs on both client and server side
- Network path with traceroute or mtr
- Packet capture with tcpdump for intermittent issues

### When to Escalate

**Escalate to Network Team:**
- BGP routing issues affecting multiple hosts
- Switch/router hardware failures
- ISP/WAN circuit issues
- Core network infrastructure changes needed
- Firewall ACL changes (if outside your access)

**Handle at SRE/System Level:**
- Application not listening on port (app team)
- Host firewall rules (within your control)
- DNS issues (check /etc/resolv.conf first)
- Single host networking (interface, routing table)
- Service-level issues (restart, config)

---

## 8. Quick Command Reference

### Most Used Commands (SRE Focus)

```bash
# Connectivity
ping <host>
curl -v http://<host>:<port>
telnet <host> <port>

# DNS
dig <hostname>
nslookup <hostname>

# Listening ports
ss -tlnp
netstat -tlnp

# Active connections
ss -tunap
netstat -tunap

# Routing
ip route show
traceroute <destination>

# Packet capture
sudo tcpdump -i any -n port <port>

# Interface info
ip addr show
ip -s link show eth0

# Firewall
sudo iptables -L -n -v
```

### One-Liners for Common Tasks

```bash
# Find which process is using a port
sudo lsof -i :8080
sudo ss -tlnp | grep :8080

# Count connections by state
ss -ant | awk '{print $1}' | sort | uniq -c

# Top talkers (most connections)
netstat -ntu | awk '{print $5}' | cut -d: -f1 | sort | uniq -c | sort -rn | head

# Test if port is open
timeout 2 bash -c "</dev/tcp/host/port" && echo "Open" || echo "Closed"

# Continuous connection monitoring
watch -n 1 'ss -s'

# Check for packet loss over time
ping <host> | ts '[%Y-%m-%d %H:%M:%S]'
```

---

## 9. Managing Infrastructure at Scale

### Large-Scale Load Balancer Deployments

**Common Challenges:**
- Configuration drift across instances
- Consistent health checks and failover behavior
- Centralized monitoring and alerting
- Automated deployments without downtime
- Version control and change management

**Best Practices:**
- Configuration management tools (Ansible, Chef, Puppet) for consistency
- Automated testing before production deployment
- Phased rollouts with verification gates
- Centralized logging and metrics collection
- Infrastructure as Code for reproducibility

### BGP-Based High Availability

**Common Patterns:**
- Multiple sites announcing same VIP via BGP
- Active site has preferred route (lower AS-path or higher local-pref)
- On failover: withdraw BGP route from active, announce from standby
- Traffic automatically flows to secondary site via BGP convergence
- Health verification before enabling secondary

**Implementation Considerations:**
- BGP convergence time vs application SLAs
- Graceful draining of active connections
- Monitoring BGP session state
- Automated vs manual failover triggers
- Rollback procedures if secondary is unhealthy

---

## 10. Additional Resources

- **BGP:** RFC 4271 - BGP-4 specification
- **TCP:** RFC 793 - Transmission Control Protocol
- **Load Balancing:** HAProxy docs, NGINX docs
- **Troubleshooting:** Brendan Gregg's Linux Performance Tools
- **Packet Analysis:** Wireshark User Guide

---

## SRE/Network Operations Skills Summary

**Core Competencies:**
- Production experience with networking infrastructure at scale
- Systematic troubleshooting methodology (layer-by-layer)
- BGP fundamentals and practical implementation
- Load balancing concepts and troubleshooting
- Knowing when to escalate vs solve independently
- Automation and infrastructure-as-code mindset

**Practical Skills:**
- Layer 3-7 network troubleshooting
- TCP/IP packet analysis with tcpdump
- Load balancer configuration and health checks
- BGP peering and route troubleshooting
- Linux networking commands and tools
- Scripting for automation and diagnostics

**Not Required to Memorize:**
- Every tcpdump flag and filter syntax
- All BGP attributes and path selection rules
- Exact iptables/firewalld command syntax

**Required Knowledge:**
- How to systematically troubleshoot connectivity issues
- When to use different tools (ping vs tcpdump vs ss)
- Where to look for information (man pages, docs, logs)
- Understanding of protocols at conceptual level
- Experience with production systems at scale

---

**For SRE and Network Operations Roles**
