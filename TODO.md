# OpenBGPD ECMP Multipath - TODO

Branch: `feature/linux-ecmp-multipath`
Repo: `doxxcorp/openbgpd`

## Status: IPv4 + IPv6 ECMP working on PM1 and PM2

---

## Testing Required

### Withdrawal / Failover
- [ ] Stop CC2 inet-bgpd, verify 207.207.200.0/24 falls back to single-path CC1 cleanly
- [ ] Stop CC1 inet-bgpd, verify traffic shifts entirely to CC2
- [ ] Restart CC2 inet-bgpd, verify ECMP re-establishes with both nexthops
- [ ] Rapid flap: stop/start CC2 inet-bgpd 3x in 10 seconds, verify stable state
- [ ] Withdraw single prefix from CC2 (comment out `network 207.207.200.0/24`), verify that prefix goes single-path while 207.207.201.0/24 stays multipath

### Crash Recovery
- [ ] `kill -9` the inet-bgpd process on PM1, verify systemd restarts it and ECMP re-establishes
- [ ] `kill -9` the inet-bgpd process on PM2, verify same
- [ ] Verify mesh bgpd (port 178) is unaffected during inet-bgpd crash/restart
- [ ] Verify fallback metric 999 default route keeps PM reachable during inet-bgpd downtime

### IPv4 Startup Timing
- [ ] Clean `systemctl restart inet-bgpd` on PM1 - IPv4 ECMP sometimes requires CC2 session bounce to install multipath. IPv6 works from startup. Root cause: `prefix_evaluate` ECMP trigger timing when both CC paths arrive during initial session establishment. Needs investigation.
- [ ] Test on PM2 (simpler config, fewer peers) to see if startup race exists there too

### IPv6 Specific
- [ ] Verify IPv6 ECMP traffic actually splits (tcpdump on both CCs for IPv6 dst)
- [ ] Test IPv6 withdrawal (stop CC2 IPv6 announcements, verify fallback)
- [ ] Verify `RTNH_F_ONLINK` doesn't cause issues with other IPv6 routes

### Multipath Route Correctness
- [ ] Verify `ip route show` never shows duplicate nexthops (same CC twice)
- [ ] Verify non-ECMP prefixes (64.255.47.0/24, 74.126.155.0/24, etc.) remain single-path
- [ ] Verify transit routes (GTT full table, HE full table) are not affected by ECMP code
- [ ] Check mnl_cb_run error rate - should not increase vs pre-ECMP baseline

### Scale / Stability
- [ ] Run for 24h+ with ECMP active, check for memory leaks (bgpd RSS growth)
- [ ] Verify kroute chain integrity after multiple add/remove cycles
- [ ] `bgpctl reload` on PM with ECMP active - verify ECMP survives config reload
- [ ] Add a 3rd CC (future) - verify 3-way ECMP builds correctly

---

## Known Issues

### IPv4 Startup Race (PM1)
After `systemctl restart inet-bgpd`, IPv4 ECMP sometimes installs single-path only. Bouncing the CC2 session (`bgpctl neighbor clear`) triggers correct multipath installation. IPv6 ECMP installs correctly from startup. The `prefix_evaluate` ECMP trigger fires but the main process `kr4_change` may not chain the second nexthop during the initial session convergence burst. Needs deeper investigation.

### GTT Transit Not Routing (PM2)
PM2 announces all prefixes to GTT (AS3257) - confirmed by session prefix counters (4 IPv4, 5 IPv6 sent). However, GTT is not propagating our prefixes directly (`3257 16740` path not seen in global table). GTT only has our prefixes via GTHost (`3257 63023 16740`). User contacted GTT NOC. Not an ECMP issue.

### `show rib out` A Flag Display
The `A` (Announced) flag in `bgpctl show rib out` only appears on self-originated `network` statements, not on iBGP-learned re-exports. Routes ARE being sent to eBGP peers despite lacking the flag. Confirmed by session prefix counters. Cosmetic issue.

### mnl_cb_run Errors
~50-80 per 30 seconds during steady state. Pre-existing from before ECMP work. From transit full table processing. `kroute_insert` fix (skip `send_rtmsg` on multipath append) should reduce these but they still occur during initial convergence.

---

## Code Changes (Session 1 + 2)

### Files Modified (8 files)

| File | Change |
|------|--------|
| `bgpd.h` | `F_ECMP 0x0100` flag, `ACTION_SET_ECMP` enum |
| `rde.h` | `NEXTHOP_ECMP 0x10`, `NEXTHOP_MASK` expanded to `0x1f` |
| `parse.y` | `FIBMULTIPATH` token, `fib-multipath` keyword, grammar rule |
| `rde_filter.c` | `ACTION_SET_ECMP` handler sets `NEXTHOP_ECMP`, display name |
| `rde.c` | `rde_send_kroute()`: dmetric-based `F_ECMP` detection + ECMP sibling kroute loop |
| `rde_decide.c` | `prefix_evaluate()`: ECMP sibling add/remove trigger for FIB update |
| `kroute-linux.c` | `kr4_change`/`kr6_change`: ECMP nexthop chaining. `kroute_remove`: ECMP-aware withdrawal. `kroute_insert`: suppress multipath append netlink. `add_multipath_attr`: IPv6 `RTNH_F_ONLINK` + ifindex resolution |
| `printconf.c` | Prints `fib-multipath` for `ACTION_SET_ECMP` |
| `bgpd_imsg.c` | Empty case for `ACTION_SET_ECMP` serialization |

### Config Syntax
```
match from any prefix X.X.X.X/Y set fib-multipath
```

### Deployment
- Binary: `/usr/sbin/inet-bgpd` (separate from mesh `/usr/sbin/bgpd`)
- Service: `inet-bgpd.service` with `Type=simple`, `TimeoutStartSec=300`
- Both PMs: `fib-multipath` rules for 207.207.200.0/24, 207.207.201.0/24, 2602:f5c1::/48, 2a11:46c0::/48

---

## Future Work

- [ ] Consider `RTNH_F_ONLINK` for IPv4 multipath too (for consistency)
- [ ] wg-server client hashing / CC pinning for VPN traffic
- [ ] Integrate ECMP config generation into `dn-infra.go` `generate_inet_bgpd`
