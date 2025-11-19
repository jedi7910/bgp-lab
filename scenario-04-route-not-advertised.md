# Scenario 4: Route Exists But Not Advertised

## Symptom
- Connected network exists on router (interface is up)
- Route is NOT in BGP table
- Neighbor does not receive the route
- Customer reports network unreachable

## Lab Setup

**Configuration issue:** Router1 has 10.1.0.0/24 configured on loopback interface but removed from BGP network statement.

## Investigation

### Check Local BGP Table (Router1)
```
show ip bgp
```

**Output:**
```
   Network          Next Hop            Metric LocPrf Weight Path
*> 10.2.0.0/24      172.20.0.20              0             0 65002 i
```

**Problem identified:** Only sees 10.2.0.0/24 (from Router2). Our own 10.1.0.0/24 is missing!

### Check What We're Advertising (Router1)
```
show ip bgp neighbors 172.20.0.20 advertised-routes
```

**Output:**
```
   Network          Next Hop            Metric LocPrf Weight Path
*> 10.2.0.0/24      0.0.0.0                                0 65002 i
```

**Problem confirmed:** Not advertising 10.1.0.0/24 to neighbor.

### Check Neighbor's View (Router2)
```
show ip bgp
```

**Output:**
```
   Network          Next Hop            Metric LocPrf Weight Path
*> 10.2.0.0/24      0.0.0.0                  0         32768 i
```

**Router2 only sees its own route.** It never received 10.1.0.0/24 from Router1.

### Verify Interface Still Exists (Router1)
```
show ip route 10.1.0.0/24
```

**Output should show:**
```
C>* 10.1.0.0/24 is directly connected, lo
```

**The route exists as a connected interface but is NOT in BGP!**

## Root Cause

**Missing BGP Network Statement**

The interface exists and is up, but BGP doesn't know to advertise it.

### Check BGP Configuration (Router1)
```
show run bgp
```

**What's missing:**
```
router bgp 65001
 bgp router-id 1.1.1.1
 neighbor 172.20.0.20 remote-as 65002
 !
 address-family ipv4 unicast
  neighbor 172.20.0.20 activate
  ! network 10.1.0.0/24 ← MISSING!
 exit-address-family
exit
```

**The `network 10.1.0.0/24` statement is not configured.**

## How This Happens in Production

**Common scenarios:**
1. **Configuration change error** - Accidentally removed during edit
2. **Copy-paste mistake** - Config template missing network statement
3. **After router reload** - Config not saved, reverted to old
4. **Automation failure** - Ansible/Chef/Puppet didn't apply full config
5. **Manual troubleshooting** - Someone removed it to test and forgot to add back

## Solution

### Add Network Statement (Router1)
```
configure terminal
router bgp 65001
 address-family ipv4 unicast
  network 10.1.0.0/24
 exit-address-family
exit
write memory
```

**BGP will immediately advertise the route** (no need to clear session).

## Verification

### Check Local BGP Table (Router1)
```
show ip bgp
```

**Success output:**
```
   Network          Next Hop            Metric LocPrf Weight Path
*> 10.1.0.0/24      0.0.0.0                  0         32768 i
*> 10.2.0.0/24      172.20.0.20              0             0 65002 i
```

Now both routes are present!

### Check Advertisements (Router1)
```
show ip bgp neighbors 172.20.0.20 advertised-routes
```

**Success output:**
```
   Network          Next Hop            Metric LocPrf Weight Path
*> 10.1.0.0/24      0.0.0.0                  0         32768 i
*> 10.2.0.0/24      0.0.0.0                                0 65002 i
```

Now advertising 10.1.0.0/24!

### Check Neighbor Received It (Router2)
```
show ip bgp
```

**Success output:**
```
   Network          Next Hop            Metric LocPrf Weight Path
*> 10.1.0.0/24      172.20.0.10              0             0 65001 i
*> 10.2.0.0/24      0.0.0.0                  0         32768 i
```

Router2 now sees 10.1.0.0/24 from Router1!

## Key Takeaways

**Troubleshooting flow:**
1. Customer reports network X unreachable
2. Check: Is route in our BGP table? (`show ip bgp X`)
3. If not: Is interface up? (`show ip route X`)
4. If interface up but not in BGP: Check config (`show run bgp`)
5. Fix: Add network statement

**The gotcha:** Route can be connected/up but NOT in BGP if network statement missing.

**Prevention in production:**
- Configuration management (Ansible/Chef validates network statements)
- Always `write memory` after changes
- Config backup and diff checking
- Peer review for BGP changes

## Related Issues

### Network Statement with Wrong Mask
```
router bgp 65001
 address-family ipv4 unicast
  network 10.1.0.0/16  ← Wrong! Interface is /24
```

BGP won't advertise because exact match required.

**Solution:** Match the actual interface mask:
```
network 10.1.0.0/24
```

### Route-Map Filtering
Even with network statement, route-map might filter it:
```
router bgp 65001
 neighbor 172.20.0.20 route-map BLOCK-10 out
!
route-map BLOCK-10 deny 10
 match ip address prefix-list BLOCK_PREFIX
!
ip prefix-list BLOCK_PREFIX permit 10.1.0.0/24
```

**Check for route-maps:**
```
show ip bgp neighbors 172.20.0.20
# Look for: "Route map for outgoing advertisements"
```

## Production Troubleshooting Checklist

When route not being advertised:

- [ ] Is interface up? (`show interface lo`)
- [ ] Is route in routing table? (`show ip route 10.1.0.0/24`)
- [ ] Is route in BGP table? (`show ip bgp 10.1.0.0/24`)
- [ ] Is network statement configured? (`show run bgp`)
- [ ] Does network statement match exact mask?
- [ ] Is neighbor activated? (`show ip bgp neighbors X`)
- [ ] Are there route-maps filtering? (`show run | include route-map`)
- [ ] Is route being suppressed? (check for aggregate statements)

## Interview Talking Points

**Question:** "Customer reports their network 10.5.0.0/24 is unreachable. How do you troubleshoot?"

**Answer:**
1. "First, I'd check if the route is in our BGP table with `show ip bgp 10.5.0.0/24`"
2. "If not in BGP, I'd verify the interface is up and route is connected"
3. "Then check the BGP config for the network statement - it might be missing or have wrong mask"
4. "I'd also check for route-maps that might be filtering the advertisement"
5. "Once I add the network statement, the route should advertise immediately"
6. "Finally, verify the neighbor received it by checking their BGP table"

**Question:** "What's the difference between route in routing table vs BGP table?"

**Answer:**
- "Routing table contains ALL routes (connected, static, BGP, OSPF, etc.)"
- "BGP table only contains routes BGP knows about"
- "A route can be connected but not in BGP if there's no network statement"
- "Conversely, a route can be in BGP table but not installed in routing table if there's a better route from another protocol"