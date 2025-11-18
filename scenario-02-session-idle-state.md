# Scenario 2: BGP Session Stuck in Idle State

## Symptom
- BGP session shows "Idle" state
- No routes being exchanged
- Session never progresses beyond Idle

## Investigation

### Check BGP Summary
```
show ip bgp summary
```
**Output:**
```
Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt
172.20.0.20     4      65003         0         0        0    0    0    never       Idle        Idle
```

**Key indicators:**
- State = **Idle**
- Up/Down = **never** (session never established)
- MsgRcvd/MsgSent = **0** (no BGP messages exchanged)

### Check Neighbor Details
```
show ip bgp neighbors 172.20.0.20
```

**Look for:**
- AS number configuration
- Connection attempts
- Error messages

## Root Cause

**AS Number Mismatch**

Router1 configured to expect AS 65003, but Router2 is actually AS 65002.

BGP requires exact AS number match in the `neighbor remote-as` configuration. When there's a mismatch:
- Router won't attempt to establish connection
- Session remains in Idle state
- No BGP messages exchanged

### How the Misconfiguration Happened
```
router bgp 65001
 neighbor 172.20.0.20 remote-as 65003  ← Wrong! Should be 65002
```

## Solution

### Fix the AS Number
```
configure terminal
router bgp 65001
 neighbor 172.20.0.20 remote-as 65002
exit
write memory
```

### Optional: Clear Session to Speed Recovery
```
clear ip bgp 172.20.0.20
```

## Verification
```
show ip bgp summary
```

**Success Output:**
```
Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt
172.20.0.20     4      65002         3         4        0    0    0 00:00:15            1        2
```

**Key changes:**
- State now shows route count (Established)
- Up/Down shows session uptime
- MsgRcvd/MsgSent incrementing

## Key Takeaways

- **Idle state** = BGP won't even try to connect
- Common causes of Idle:
  - AS number mismatch (this scenario)
  - Neighbor IP unreachable
  - Interface down
  - Peer not configured
- Always verify AS numbers in documentation/IaC
- `show ip bgp neighbors` reveals configuration mismatches

## Production Troubleshooting Flow

1. Check session state: **Idle** = config or reachability issue
2. Verify AS numbers match documentation
3. Check network reachability: `ping <peer-ip>`
4. Verify peer is configured on remote side
5. Check interface status

## Related States

| State | Meaning | Common Causes |
|-------|---------|---------------|
| **Idle** | Not attempting connection | AS mismatch, interface down, peer not configured |
| **Active** | Trying but failing to connect | Firewall blocking TCP 179, wrong IP |
| **Connect** | TCP handshake in progress | Normal transition (should be brief) |