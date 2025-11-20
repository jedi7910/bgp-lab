# BGP Troubleshooting Lab

Hands-on BGP troubleshooting scenarios built with Docker and FRRouting for interview preparation and skill development.

## Overview

This lab demonstrates common BGP issues encountered in production environments, particularly relevant for SRE and network operations roles. Each scenario includes the problem, investigation steps, root cause analysis, and solutions.

## Lab Environment

**Technology Stack:**
- Docker containers (Alpine Linux)
- FRRouting (FRR) 8.4
- 2-router topology with BGP peering

**Network Topology:**
```
[Router1: 172.20.0.10/16 - AS 65001] ←→ [Router2: 172.20.0.20/16 - AS 65002]
        |                                          |
   10.1.0.0/24                                10.2.0.0/24
```

## Scenarios Covered

### 1. [Routes Not Exchanged](scenario-01-no-routes-exchanged.md)
**Problem:** BGP session established but no routes being exchanged  
**Root Cause:** Missing neighbor activation and eBGP policy requirement  
**Key Learning:** Always activate neighbors in address-family configuration

### 2. [Session Stuck in Idle State](scenario-02-session-idle-state.md)
**Problem:** BGP session never establishes, stuck in Idle  
**Root Cause:** AS number mismatch in configuration  
**Key Learning:** Verify AS numbers match between config and actual peer

### 3. [BGP Session Flapping](scenario-03-bgp-flapping.md)
**Problem:** Session repeatedly goes up and down  
**Root Cause:** Network instability and aggressive hold timers  
**Key Learning:** Balance fast failover vs stability when setting timers

### 4. [Route Not Advertised](scenario-04-route-not-advertised.md)
**Problem:** Connected network exists but not in BGP table  
**Root Cause:** Missing BGP network statement  
**Key Learning:** Routes must be explicitly configured in BGP to be advertised

### 5. [iBGP Next-Hop Unreachable](scenario-05-ibgp-nexthop-unreachable.md)
**Problem:** iBGP route in BGP table but not installed in routing table  
**Root Cause:** Next-hop preserved in iBGP and unreachable  
**Key Learning:** Use next-hop-self for iBGP or ensure IGP distributes loopbacks

## Quick Start

### Prerequisites
- Docker installed
- WSL2 (if on Windows)
- Basic networking knowledge

### Setup

1. **Create Docker network:**
```bash
docker network create --subnet=172.20.0.0/16 bgp-net
```

2. **Launch routers:**
```bash
# Router 1
docker run -d --name router1 --network bgp-net --ip 172.20.0.10 \
  --cap-add=NET_ADMIN --privileged frrouting/frr:latest

# Router 2
docker run -d --name router2 --network bgp-net --ip 172.20.0.20 \
  --cap-add=NET_ADMIN --privileged frrouting/frr:latest
```

3. **Enable BGP daemon on both routers:**
```bash
docker exec -it router1 bash
sed -i 's/bgpd=no/bgpd=yes/' /etc/frr/daemons
/etc/init.d/frr restart
exit
```

4. **Configure BGP (see individual scenarios for specific configs)**

### Useful Commands

**Access router:**
```bash
docker exec -it router1 bash
vtysh
```

**Check BGP status:**
```bash
show ip bgp summary
show ip bgp
show ip bgp neighbors <ip>
```

**View routing table:**
```bash
show ip route
show ip route bgp
```

## Reference Materials

- [BGP Command Cheat Sheet](bgp-command-cheatsheet.md) - Quick reference for troubleshooting commands
- [Router Configurations](router-configs.md) - Base configurations for both routers

## Use Cases

**Interview Preparation:**
- Demonstrates hands-on BGP troubleshooting skills
- Shows understanding of eBGP vs iBGP
- Practical experience with common production issues

**Learning:**
- Safe environment to break and fix BGP
- Understand BGP state machines
- Practice systematic troubleshooting methodology

**Reference:**
- Quick lookup for BGP issues
- Command syntax reference
- Root cause analysis examples

## Skills Demonstrated

- BGP protocol fundamentals (eBGP and iBGP)
- Network troubleshooting methodology
- Linux networking and Docker
- Configuration management
- Production-oriented problem solving

## Author

Built for interview preparation and continuous learning in network operations and SRE roles.

## License

Educational use - feel free to use and modify for learning purposes.

---

**Note:** This lab uses simplified configurations for learning. Production BGP deployments require additional security, redundancy, and policy considerations.