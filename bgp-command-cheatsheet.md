# BGP Command Cheat Sheet for SRE Troubleshooting

## Quick Health Checks

### Check BGP Session Status
```bash
show ip bgp summary
```
**What to look for:**
- State showing a number (e.g., "1") = Established, routes being exchanged
- **(Policy)** = Peered but routes blocked by policy
- **Idle/Active/Connect** = Session broken
- **InQ/OutQ > 0** = Congestion or processing issues
- **MsgRcvd/MsgSent not incrementing** = Dead session

**Example Output:**
```
Neighbor        V    AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt
172.20.0.20     4 65002        53        60        0    0    0 00:00:52            1        2
```

---

## Route Verification

### Check BGP Routing Table
```bash
show ip bgp
```
**What to look for:**
- **\*>** = Valid and best route (installed in routing table)
- **\*** = Valid but not best
- **Next Hop 0.0.0.0** = Locally originated
- **AS Path** = Route's journey through AS numbers

**Example Output:**
```
   Network          Next Hop            Metric LocPrf Weight Path
*> 10.1.0.0/24      0.0.0.0                  0         32768 i
*> 10.2.0.0/24      172.20.0.20              0             0 65002 i
```

### Check Specific Route
```bash
show ip bgp 10.2.0.0/24
```
Shows all paths to a specific network.

### Check What's Installed in Routing Table
```bash
show ip route bgp
# or
show ip route 10.2.0.0/24
```
**Critical:** Route can be in BGP table but NOT installed in routing table due to:
- Better route from another protocol
- Next-hop unreachable

---

## Detailed Neighbor Diagnostics

### Check Specific Neighbor Details
```bash
show ip bgp neighbors 172.20.0.20
```
**Key sections to check:**

**Session State:**
```
BGP state = Established, up for 00:00:52
```
- Established = good
- Active = trying to connect (BAD)
- Idle = not connected (BAD)

**Policy Issues:**
```
For address family: IPv4 Unicast
  Inbound updates discarded due to missing policy
  Outbound updates discarded due to missing policy
```
Indicates neighbor not activated or policy blocking routes.

**Connection Stability:**
```
Connections established 1; dropped 0
Last reset 00:16:38, Reason: Connection reset by peer
```
Shows flapping history and why session last went down.

### Check Routes Received from Neighbor
```bash
show ip bgp neighbors 172.20.0.20 received-routes
```
What routes the neighbor is sending you (may be filtered).

### Check Routes Advertised to Neighbor
```bash
show ip bgp neighbors 172.20.0.20 advertised-routes
```
What routes you're sending to neighbor.

---

## Common Troubleshooting Flows

### Customer Reports VM Unreachable

**Step 1:** Check if route exists
```bash
show ip route <customer-vm-ip>
```

**Step 2:** If no route, check BGP table
```bash
show ip bgp <network>
```

**Step 3:** If not in BGP table, check session status
```bash
show ip bgp summary
```

**Step 4:** If session down, check neighbor details
```bash
show ip bgp neighbors <peer-ip>
```

---

### Routes Not Being Exchanged

**Symptom:** `show ip bgp summary` shows (Policy)

**Check 1:** Verify neighbor activation
```bash
show ip bgp neighbors <peer-ip>
# Look for: "No AFI/SAFI activated for peer"
```

**Fix:**
```bash
configure terminal
router bgp <local-as>
 address-family ipv4 unicast
  neighbor <peer-ip> activate
 exit-address-family
exit
write memory
```

**Check 2:** Verify eBGP policy setting
```bash
show run bgp
# Look for: "bgp ebgp-requires-policy"
```

**Fix (lab only):**
```bash
configure terminal
router bgp <local-as>
 no bgp ebgp-requires-policy
exit
write memory
```

**Clear session to apply changes:**
```bash
clear ip bgp *
```

---

### BGP Session Flapping

**Check flap statistics:**
```bash
show ip bgp neighbors <peer-ip>
# Look for: "Connections established X; dropped Y"
```

**Common causes:**
- Network instability
- Hold timer too aggressive
- High CPU on router
- Firewall intermittently blocking TCP 179

---

## Understanding BGP States

| State | Meaning | Action |
|-------|---------|--------|
| **Idle** | Not attempting connection | Check config, network reachability |
| **Connect** | TCP handshake in progress | Check firewall, TCP 179 access |
| **Active** | Trying to connect but failing | Check peer IP, AS number, reachability |
| **OpenSent** | Sent OPEN message | Wait for peer response |
| **OpenConfirm** | Received OPEN, negotiating | Normal transition state |
| **Established** | Session up, exchanging routes | Healthy! |

---

## Quick Reference: Route Selection

**BGP picks best route using this order:**
1. **Highest Weight** (Cisco/FRR local attribute)
2. **Highest Local Preference**
3. **Locally originated** (routes you advertise)
4. **Shortest AS Path**
5. **Lowest Origin** (IGP > EGP > Incomplete)
6. **Lowest MED** (Multi-Exit Discriminator)
7. **eBGP over iBGP**
8. **Lowest IGP metric to next-hop**
9. **Oldest route** (for stability)

**For SRE troubleshooting:** Usually just care about #3 and #4.

---

## Administrative Distances (Linux/FRR)

Lower = more preferred

| Protocol | AD | Notes |
|----------|-----|-------|
| Connected | 0 | Directly attached |
| Static | 1 | Manually configured |
| eBGP | 20 | External BGP |
| OSPF | 110 | Internal routing |
| iBGP | 200 | Internal BGP |

**Why this matters:** BGP route might exist but static route wins.

---

## Subnet Quick Reference

| CIDR | Addresses | Last Octet Changes | Example Range |
|------|-----------|-------------------|---------------|
| /32 | 1 | None (host route) | 10.1.1.1 |
| /24 | 256 | Yes | 10.1.1.0 - 10.1.1.255 |
| /16 | 65,536 | Last 2 octets | 10.1.0.0 - 10.1.255.255 |
| /8 | 16M | Last 3 octets | 10.0.0.0 - 10.255.255.255 |
| /0 | All | Default route | 0.0.0.0 - 255.255.255.255 |

**Longest Prefix Match:** Most specific route (highest /number) wins.

---

## Production Best Practices

✅ **DO:**
- Check BGP state before assuming route issues
- Document expected routes and next-hops
- Monitor BGP session uptime and flap counts
- Coordinate with network team for routing changes
- Use `show` commands liberally - they're read-only

❌ **DON'T:**
- Clear BGP sessions in production without approval
- Make routing changes without change control
- Assume routes without checking routing table
- Disable eBGP policy in production (lab only)

---

## Emergency Commands

**Session stuck, need to force reset:**
```bash
clear ip bgp <neighbor-ip>
```

**Nuclear option - reset all BGP sessions:**
```bash
clear ip bgp *
```
⚠️ **WARNING:** This drops all BGP sessions. Get approval first.

---

## Related Tools

**Check reachability to BGP peer:**
```bash
ping <peer-ip>
telnet <peer-ip> 179  # Check BGP port access
```

**Check kernel routing:**
```bash
ip route get <destination-ip>  # Shows actual routing decision
```

**Check interface status:**
```bash
show interface <interface-name>
```