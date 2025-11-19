# Scenario 3: BGP Session Flapping

## Symptom
- BGP session repeatedly goes up and down
- Customer reports intermittent connectivity loss
- Routes appear and disappear from routing table
- Monitoring shows session state changes

## Lab Setup

**Simulating flapping conditions:**
- Aggressive hold timers (3s keepalive, 9s hold)
- 60% packet loss on both routers
- Bidirectional network instability

## Investigation

### Check Connection Statistics (Router2)
```
show ip bgp neighbors 172.20.0.10
```

**Key section - Connection Stats:**
```
Connections established 8; dropped 7
Last reset 00:00:08, Notification received (Hold Timer Expired)
```

**Analysis:**
- **8 established** - Session has come up 8 times
- **7 dropped** - Session has dropped 7 times
- **Hold Timer Expired** - Router didn't receive keepalives in time

### Check BGP Summary (Router2)
```
show ip bgp summary
```

**Output:**
```
Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt
172.20.0.10     4      65001       243       261        0    0    0 00:00:37            1        2
```

**Key indicators:**
- **Up/Down: 00:00:37** - Current session only 37 seconds old (unstable)
- **MsgRcvd: 243** - Many messages exchanged across multiple sessions
- Session is currently established but won't last

### Observing State Transitions (Router1)

**Session cycling through states over time:**

**T+0 seconds:**
```
172.20.0.20     4      65002       250       278        0    0    0 00:00:13         Idle        0
```
Session down (Idle state)

**T+20 seconds:**
```
172.20.0.20     4      65002       251       282        0    0    0 00:00:34  OpenConfirm        0
```
Attempting to establish (OpenConfirm)

**T+33 seconds:**
```
172.20.0.20     4      65002       251       286        0    0    0 00:00:44     OpenSent        0
```
Still attempting (OpenSent)

**T+43 seconds:**
```
172.20.0.20     4      65002       254       289        0    0    0 00:00:54         Idle        0
```
Failed again, back to Idle

**Pattern:** Session can't stay established, cycles through connection attempts

### Check Timer Configuration
```
show ip bgp neighbors 172.20.0.10
```

**Router1 Configuration (Aggressive):**
```
Configured hold time is 9 seconds, keepalive interval is 3 seconds
```

**Router2 Configuration (Default):**
```
Configured hold time is 180 seconds, keepalive interval is 60 seconds
```

**The Problem:**
- Router1 has aggressive 9-second hold timer
- With 60% packet loss, keepalives don't arrive in time
- Router1 times out and drops session
- Timer mismatch between routers compounds instability

## Root Causes

### 1. Network Instability
- High packet loss (60% in this lab)
- In production: Bad cables, switch issues, ISP problems, congestion

### 2. Aggressive Hold Timers
- 9-second hold timer too sensitive for unstable network
- Default 180s would be more forgiving
- Production: Balance responsiveness vs stability

### 3. Asymmetric Timer Configuration
- Router1: 9s hold time
- Router2: 180s hold time
- BGP uses the LOWER of the two (9s)
- Both sides affected by aggressive setting

## Impact in Production

**When BGP flaps:**
1. **Routes withdrawn** - Traffic blackholing during down time
2. **Routing instability** - Other routers react to changes
3. **Customer impact** - VMs lose connectivity intermittently
4. **Alert fatigue** - Constant notifications mask real issues

## Solution

### Immediate: Remove Network Instability

**Router1:**
```bash
tc qdisc del dev eth0 root
```

**Router2:**
```bash
tc qdisc del dev eth0 root
```

**Verify session stabilizes:**
```
show ip bgp summary
```

Session should stay established, Up/Down time should increase.

### Short-term: Adjust Hold Timers

**If network can't be fixed immediately, increase hold time tolerance:**

**Router1:**
```
configure terminal
router bgp 65001
 neighbor 172.20.0.20 timers 30 90
exit
write memory
```

**Values:**
- Keepalive: 30s (was 3s)
- Hold: 90s (was 9s)
- More tolerant of packet loss

### Long-term: Fix Root Cause

**Production troubleshooting steps:**
1. **Identify network path** - Which links are involved?
2. **Check interface errors** - `show interface eth0` for CRC, drops
3. **Monitor latency/loss** - Run continuous ping/mtr
4. **Check hardware** - Bad NICs, cables, switch ports
5. **Review logs** - System logs, kernel messages
6. **Engage network team** - For deeper infrastructure investigation

## Verification

**After fixes, check connection stats:**
```
show ip bgp neighbors 172.20.0.10
```

**Success indicators:**
```
Connections established 1; dropped 0
Last reset never
```
- Only 1 established connection (no reconnects)
- Zero drops
- Up/Down time steadily increasing

**Check BGP summary:**
```
Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt
172.20.0.10     4      65001       150       155        0    0    0 01:15:23            1        2
```
- Up/Down showing hours, not seconds/minutes
- Stable route counts

## Key Takeaways

**Detection:**
- Check "Connections established vs dropped" ratio
- Short Up/Down times indicate instability  
- "Hold Timer Expired" in last reset reason
- BGP state cycling (Idle → OpenSent → Idle)

**Common Causes in Production:**
- Fiber degradation or dirty optics
- Switch port flapping
- ISP circuit issues
- MTU problems causing fragmentation
- High CPU on router (can't process keepalives)

**Prevention:**
- Monitor BGP session metrics (dropped count, uptime)
- Alert on flap rate, not just down state
- Use appropriate hold timers for network stability
- Implement BFD (Bidirectional Forwarding Detection) for faster detection

**Interview Talking Points:**
- "I'd check connection statistics to see dropped count"
- "Hold Timer Expired indicates keepalives not received"
- "Balance fast failover vs stability when setting timers"
- "Would work with network team to investigate physical layer"
- "In my NetScaler work, BGP flapping during failover would blackhole traffic"

## Production Considerations

**When investigating flapping:**
1. Don't immediately blame BGP - it's often the symptom, not cause
2. Check both sides of peering (one side may see different stats)
3. Look at historical data - is this new or ongoing?
4. Consider maintenance windows - was work done recently?
5. Check for asymmetric issues - one direction worse than other

**Timers best practices:**
- Default (60s/180s) for stable production networks
- Shorter (10s/30s) for fast failover where stability proven
- Never go below 3s/9s (too sensitive)
- Match timers on both sides when possible
- Use BFD if sub-second detection needed