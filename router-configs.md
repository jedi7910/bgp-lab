# Router Base Configurations

Standard configurations for the BGP lab routers used across all scenarios.

## Router 1 - Base Configuration

**AS Number:** 65001  
**Router ID:** 1.1.1.1  
**Management IP:** 172.20.0.10/16  
**Loopback:** 10.1.0.1/24

### Complete Config (eBGP)
```
configure terminal

! Enable BGP
router bgp 65001
 bgp router-id 1.1.1.1
 no bgp ebgp-requires-policy
 neighbor 172.20.0.20 remote-as 65002
 !
 address-family ipv4 unicast
  neighbor 172.20.0.20 activate
  network 10.1.0.0/24
 exit-address-family
exit

! Configure loopback
interface lo
 ip address 10.1.0.1/24
exit

write memory
```

### iBGP Variant (Same AS)
```
configure terminal

router bgp 65001
 bgp router-id 1.1.1.1
 no bgp ebgp-requires-policy
 neighbor 172.20.0.20 remote-as 65001
 !
 address-family ipv4 unicast
  neighbor 172.20.0.20 activate
  neighbor 172.20.0.20 next-hop-self
  network 10.1.0.0/24
 exit-address-family
exit

interface lo
 ip address 10.1.0.1/24
exit

write memory
```

---

## Router 2 - Base Configuration

**AS Number:** 65002  
**Router ID:** 2.2.2.2  
**Management IP:** 172.20.0.20/16  
**Loopback:** 10.2.0.1/24

### Complete Config (eBGP)
```
configure terminal

! Enable BGP
router bgp 65002
 bgp router-id 2.2.2.2
 no bgp ebgp-requires-policy
 neighbor 172.20.0.10 remote-as 65001
 !
 address-family ipv4 unicast
  neighbor 172.20.0.10 activate
  network 10.2.0.0/24
 exit-address-family
exit

! Configure loopback
interface lo
 ip address 10.2.0.1/24
exit

write memory
```

### iBGP Variant (Same AS)
```
configure terminal

router bgp 65001
 bgp router-id 2.2.2.2
 no bgp ebgp-requires-policy
 neighbor 172.20.0.10 remote-as 65001
 !
 address-family ipv4 unicast
  neighbor 172.20.0.10 activate
  neighbor 172.20.0.10 next-hop-self
  network 10.2.0.0/24
 exit-address-family
exit

interface lo
 ip address 10.2.0.1/24
exit

write memory
```

---

## Configuration Notes

### no bgp ebgp-requires-policy

FRR 8.x+ requires explicit route-map policies on eBGP sessions by default. The `no bgp ebgp-requires-policy` command disables this requirement for lab/testing purposes.

**Production:** Use explicit route-maps instead of disabling policy checks.

### next-hop-self (iBGP)

For iBGP configurations, `next-hop-self` ensures the advertising router sets itself as the next-hop instead of preserving the original next-hop. This prevents next-hop unreachability issues.

### Network Statements

The `network` statement tells BGP to advertise the specified prefix. The prefix must exist in the routing table (via connected interface, static route, or another protocol) for BGP to advertise it.

---

## Quick Reset Commands

**Remove all BGP config:**
```
configure terminal
no router bgp 65001
exit
write memory
```

**Clear BGP sessions:**
```
clear ip bgp *
```

**Restart FRR:**
```bash
/etc/init.d/frr restart
```

---

## Verification Commands

**Check BGP is running:**
```
show ip bgp summary
```

**Check running config:**
```
show run bgp
```

**Check interface status:**
```
show interface brief
show ip interface
```

**Check routing table:**
```
show ip route
```