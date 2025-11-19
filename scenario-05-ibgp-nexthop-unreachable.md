# Scenario 5: iBGP Next-Hop Unreachable

## Concept: iBGP vs eBGP Next-Hop Behavior

### eBGP (External BGP)
- Between different AS numbers
- **Next-hop CHANGES to advertising router**
- Example: Router1 advertises with next-hop 172.20.0.10 (itself)

### iBGP (Internal BGP)
- Between routers in same AS
- **Next-hop PRESERVED from original advertisement**
- Example: Router1 advertises with next-hop 10.1.0.1 (loopback), stays 10.1.0.1
- **This causes problems if next-hop unreachable!**

## Lab Setup
```
Topology:
[10.1.0.1/24] -- [Router1 AS 65001] --iBGP-- [Router2 AS 65001] -- [10.2.0.1/24]
                 172.20.0.10                  172.20.0.20
```

**Both routers in same AS (65001) = iBGP peering**

## The Problem

### Initial State - Working
Router1 advertises 10.1.0.0/24 with next-hop 172.20.0.10 (directly connected), route installs.

### Broken State - Next-Hop Unreachable
Router1 configured to advertise with next-hop 10.1.0.1 (loopback), which Router2 cannot reach.

## Investigation

### Check Router2's BGP Table
```
show ip bgp
```

**Broken Output:**
```
   Network          Next Hop            Metric LocPrf Weight Path
  i10.1.0.0/24      10.1.0.1                 0    100      0 i
*> 10.2.0.0/24      0.0.0.0                  0         32768 i
```

**Critical observations:**
- Route shows as `i10.1.0.0/24` (just "i", NOT "*>i")
- Missing `*` = **Route is INVALID**
- Missing `>` = Not best route (can't be best if invalid)
- Next-hop is 10.1.0.1 (Router1's loopback)

**Compare to working iBGP route:**
```
*>i10.1.0.0/24      172.20.0.10              0    100      0 i
```
Shows `*>i` = valid, best, iBGP

### Check Routing Table (Router2)
```
show ip route bgp
```

**Output:** Empty (no BGP routes installed)

**The route is in BGP table but NOT installed in routing table!**

### Check Reachability to Next-Hop (Router2)
```
show ip route 10.1.0.1
```

**Output:**
```
Routing entry for 0.0.0.0/0
  Known via "kernel", distance 0, metric 0, best
  Last update 01:22:12 ago
  * 172.20.0.1, via eth0
```

**Only shows default route** - Router2 has NO specific route to 10.1.0.1

**This is the problem:** Router2 cannot reach the next-hop, so route stays invalid.

## Root Cause

**iBGP Preserves Next-Hop**

When Router1 advertises 10.1.0.0/24 via iBGP:
1. Router1 sets next-hop to 10.1.0.1 (its loopback)
2. Router2 receives route with next-hop still 10.1.0.1
3. Router2 checks: "Can I reach 10.1.0.1?"
4. Answer: NO (not in routing table)
5. Route marked invalid, not installed

**Why eBGP didn't have this problem:**
- eBGP automatically changes next-hop to advertising router
- Router1 would use 172.20.0.10 as next-hop
- Router2 CAN reach 172.20.0.10 (directly connected)
- Route installs successfully

## Solution

### Primary Solution: next-hop-self

**Configure Router1 to change next-hop to itself:**
```
configure terminal
router bgp 65001
 address-family ipv4 unicast
  neighbor 172.20.0.20 next-hop-self
 exit-address-family
exit
write memory
clear ip bgp *
```

**What this does:**
- Forces Router1 to set next-hop to 172.20.0.10 (its transit IP)
- Router2 receives route with reachable next-hop
- Route installs successfully

### Alternative Solution: IGP Route Distribution

**Add route on Router2 to reach loopback:**
```
configure terminal
ip route 10.1.0.0/24 172.20.0.10
exit
write memory
```

**In production:** Use OSPF/ISIS to distribute loopback routes internally so all routers can reach each other's loopbacks.

**Note:** This doesn't scale - better to use next-hop-self.

## Verification After Fix

### Check Router2's BGP Table
```
show ip bgp
```

**Success Output:**
```
   Network          Next Hop            Metric LocPrf Weight Path
*>i10.1.0.0/24      172.20.0.10              0    100      0 i
*> 10.2.0.0/24      0.0.0.0                  0         32768 i
```

**Key changes:**
- Now shows `*>i` (valid, best, iBGP) ✅
- Next-hop changed to 172.20.0.10 (reachable) ✅

### Check Routing Table (Router2)
```
show ip route bgp
```

**Success Output:**
```
B>* 10.1.0.0/24 [200/0] via 172.20.0.10, eth0, weight 1, 00:00:18
```

**Route is installed!**
- [200/0] = iBGP administrative distance
- via 172.20.0.10 = reachable next-hop
- Installed in routing table ✅

### Test Connectivity (Router2)
```
ping 10.1.0.1 -c 3
```

Should work now that route is installed.

## Key Takeaways

**iBGP Next-Hop Rules:**
- iBGP preserves next-hop (doesn't change it)
- Next-hop must be reachable via IGP or directly connected
- If next-hop unreachable, route invalid and won't install

**Detection in Production:**
- Route in BGP table but missing `*` (invalid)
- Route NOT in routing table (`show ip route bgp` doesn't show it)
- Check reachability to next-hop
- Common symptom: "Customer reports network unreachable but route is advertised"

**Standard Solutions:**
1. **next-hop-self** - Force iBGP speaker to use itself as next-hop
2. **IGP distribution** - Ensure loopbacks reachable via OSPF/ISIS
3. **Route reflectors** - In larger deployments, configure properly

**Interview Talking Points:**
- "iBGP doesn't change next-hop like eBGP does"
- "Route can be in BGP table but invalid if next-hop unreachable"
- "I'd check if the route shows `*` in BGP table - missing asterisk means invalid"
- "next-hop-self is standard practice for iBGP route reflectors"
- "Would verify with `show ip route <next-hop>` to see if it's reachable"

## Production Considerations

**When to use next-hop-self:**
- iBGP peers that aren't directly connected
- Route reflectors (always use next-hop-self)
- When IGP doesn't distribute all loopbacks

**When NOT to use next-hop-self:**
- eBGP (next-hop changes automatically)
- When you specifically want to preserve next-hop for traffic engineering

**Common Mistake:**
Forgetting next-hop-self on route reflectors, causing all clients to have invalid routes.

## Comparison: eBGP vs iBGP

| Aspect | eBGP | iBGP |
|--------|------|------|
| AS Numbers | Different | Same |
| Next-Hop | Changes to advertising router | Preserved |
| AS Path | Modified (prepends AS) | Unchanged |
| Admin Distance | 20 | 200 |
| Local Preference | Not used | Used for path selection |
| Next-Hop-Self | Not needed | Often required |
| Full Mesh | Not required | Required (or use route reflectors) |

## Related Issues

### Issue: Route Reflector Without next-hop-self

**Scenario:**
- Route Reflector (RR) receives route from Client A
- RR reflects to Client B
- Next-hop preserved as Client A's IP
- Client B can't reach Client A directly
- Route invalid on Client B

**Solution:**
```
router bgp 65001
 address-family ipv4 unicast
  neighbor <clients> next-hop-self
  neighbor <clients> route-reflector-client
```

### Issue: IGP Not Distributing Loopbacks

**Symptom:**
- All iBGP routes invalid
- None can reach each other's next-hops

**Solution:**
- Ensure OSPF/ISIS includes loopback networks
- Or use next-hop-self everywhere

## Troubleshooting Checklist

When iBGP route not installing:

- [ ] Is route in BGP table? (`show ip bgp <network>`)
- [ ] Does route show `*` (valid)?
- [ ] What's the next-hop?
- [ ] Can I reach next-hop? (`show ip route <next-hop>`)
- [ ] Is next-hop-self configured on advertising router?
- [ ] Is IGP distributing loopback routes?
- [ ] Check for route-maps modifying next-hop
- [ ] Verify iBGP session established

## Advanced: Why This Matters at Vultr

**Vultr's control plane likely uses iBGP:**
- Multiple routers in same AS
- Routes distributed internally via iBGP
- Customer VM routes need to propagate across data center

**Real scenario:**
1. Customer spins up VM at 10.5.0.50
2. Compute node advertises via iBGP
3. Without next-hop-self, other compute nodes can't reach next-hop
4. Customer VM unreachable from rest of infrastructure
5. **Your job:** Detect this is next-hop issue, apply next-hop-self

**This is why they ask about iBGP specifically!**