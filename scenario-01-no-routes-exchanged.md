# Scenario 1: BGP Peered But No Routes Exchanged

## Symptom
- BGP session shows "Established" state
- `show ip bgp summary` displays "(Policy)" instead of route counts
- Uptime is increasing but no prefixes received/sent

## Investigation

### Check BGP Summary
```
show ip bgp summary
```
**Output:**
```
Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt
172.20.0.20     4      65002        14        15        0    0    0 00:11:30     (Policy) (Policy)
```

### Check BGP Table
```
show ip bgp
```
**Output:** Routes exist in local BGP table but aren't being exchanged

### Check Neighbor Details
```
show ip bgp neighbors 172.20.0.20
```
**Key findings:**
```
Inbound updates discarded due to missing policy
Outbound updates discarded due to missing policy
Last reset 00:16:38, No AFI/SAFI activated for peer
```

## Root Causes

### 1. Neighbor Not Activated in Address Family
BGP requires explicit neighbor activation within each address-family. Without activation, routes won't be sent or received.

### 2. eBGP Policy Requirement (FRR 8.x+)
FRR 8.x+ enforces explicit route-map policies on eBGP sessions by default to prevent accidental route leaking.

## Solution

### Router 1 Configuration
```
configure terminal
router bgp 65001
 no bgp ebgp-requires-policy
 address-family ipv4 unicast
  neighbor 172.20.0.20 activate
 exit-address-family
exit
write memory
```

### Router 2 Configuration
```
configure terminal
router bgp 65002
 no bgp ebgp-requires-policy
 address-family ipv4 unicast
  neighbor 172.20.0.10 activate
 exit-address-family
exit
write memory
```

### Clear BGP Sessions
```
clear ip bgp *
```
Wait 30 seconds for sessions to re-establish.

## Verification
```
show ip bgp summary
```
**Success Output:**
```
Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt
172.20.0.20     4      65002        53        60        0    0    0 00:00:52            1        2
```
```
show ip bgp
```
**Success Output:**
```
   Network          Next Hop            Metric LocPrf Weight Path
*> 10.1.0.0/24      0.0.0.0                  0         32768 i
*> 10.2.0.0/24      172.20.0.20              0             0 65002 i
```

## Key Takeaways
- Always activate neighbors in the address-family configuration
- FRR 8.x+ requires explicit policy or `no bgp ebgp-requires-policy`
- Use `show ip bgp neighbors` to identify policy issues
- Clear BGP sessions after config changes to force re-negotiation

## Production Considerations
- In production, use explicit route-maps instead of disabling policy checks
- Test changes in maintenance windows
- Document all BGP policy requirements