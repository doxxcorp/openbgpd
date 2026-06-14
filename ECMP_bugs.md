# ECMP FIB Install Bug - June 13-14, 2026

## Current State (end of June 13 evening session)

### What's Working
- MIA1: All four servers (pm1, pm2, cc1, cc2) running reverted binary with `peer_blast()` restored
- `/32` service IPs (63.236.138.1, .2) have **automatic ECMP** via `peer_blast()` on both PMs
- `/24` service IPs (207.207.200, .201, 63.236.138.0) have ECMP via **manual `ip route replace`**
- PAO1: Still running old binary with `peer_dump()` always, but ECMP works there (lucky timing)

### What's Broken
- `/24` and `/48` prefixes do NOT get automatic ECMP from `peer_blast()` or the 20s delayed resync
- Only `/32` service IPs get automatic ECMP
- Manual `ip route replace` is required for `/24`s after every inet-bgpd restart

### Binary Changes Deployed (June 13 evening)
1. **Reverted `peer_dump()` always** (commit `d3534a2`): restored original `peer_blast()` / `peer_dump()` selection in `peer_up()`. `peer_blast()` is used when peer address/capabilities haven't changed across a session flap.
2. **Hardcoded socket path** (`bgpd.h`): changed `#ifndef RUNSTATEDIR` to `#undef RUNSTATEDIR` so the socket is always `/var/run/bgpd.sock` regardless of `./configure` prefix. Prevents the `/usr/local/var/run/` mismatch.

### fib-multipath Filter Rules (PM inet-bgpd.conf)

All service prefixes have `set fib-multipath` rules on both PMs:
```
match from any prefix 207.207.200.0/24 prefixlen = 24 set fib-multipath
match from any prefix 207.207.201.0/24 prefixlen = 24 set fib-multipath
match from any prefix 2602:f5c1::/48 prefixlen = 48 set fib-multipath
match from any prefix 2a11:46c0::/48 prefixlen = 48 set fib-multipath
match from any prefix 2602:f5c1:1::10:1/128 prefixlen = 128 set fib-multipath
match from any prefix 2602:f5c1:1::10:2/128 prefixlen = 128 set fib-multipath
match from any prefix 63.236.138.1/32 prefixlen = 32 set fib-multipath
match from any prefix 63.236.138.2/32 prefixlen = 32 set fib-multipath
```

---

## The Remaining Bug: /24 and /48 ECMP FIB Install Failure

### Symptom

After CC inet-bgpd restart + `peer_blast()` to PMs:
- `/32` routes (63.236.138.1/32, .2/32): RIB has ECMP, kernel FIB has ECMP. **Working.**
- `/24` routes (207.207.200.0/24, .201/24, 63.236.138.0/24): RIB has ECMP (`*>` + `*m`), kernel FIB is **single-path**. Broken.
- `/48` routes (2602:f5c1::/48): same pattern, not tested but likely broken.

The 20-second delayed ECMP resync fires ("running delayed ECMP fib resync") and completes
("re-evaluation done for Loc-RIB") but does NOT install multipath for `/24` prefixes.

### Important: /24 and /48 ECMP Worked Before Today

The `/24` and `/48` static `network` routes have always had working ECMP with our ECMP fork patches. This is a regression from today's `peer_dump()` change and the CC inet-bgpd restarts that followed. Even after reverting to `peer_blast()`, the `/24`s aren't getting ECMP automatically - something about the CC restart + reconvergence sequence leaves the `/24` FIB in single-path state. The `/32`s work because `peer_blast()` delivers both CC paths simultaneously and the direct ECMP install path in `rde_decide.c:595-603` catches them. The `/24`s have additional eBGP paths that may interfere with the timing.

### Key Difference Between /32 and /24

**`/32` routes** have exactly 2 paths in the RIB (from cc1 and cc2 only). No eBGP peers announce these specific `/32`s.

**`/24` routes** have 4-6+ paths in the RIB: 2 from CCs (iBGP) + 2-4 from eBGP transit (GTT, HE, EIX route servers). The `from any` in the filter rule means eBGP paths ALSO get `NEXTHOP_ECMP` set, even though they're not ECMP candidates (longer AS path).

### Theory: eBGP Path Interference

The `rde_softreconfig_sync_fib()` function at `rde.c:4425` checks:
```c
ep = TAILQ_NEXT(p, rib_l);   // sibling after best
if (ep != NULL && ep->dmetric == PREFIX_DMETRIC_ECMP &&
    (prefix_nhflags(ep) & NEXTHOP_ECMP))
```

For `/32`s: `TAILQ_NEXT(cc1) = cc2` (the only other path). cc2 has `PREFIX_DMETRIC_ECMP` and `NEXTHOP_ECMP`. Check passes, ECMP installed.

For `/24`s: `TAILQ_NEXT(cc1)` should also be cc2 (same iBGP attributes, sorted before eBGP paths). But with multiple eBGP paths that also have `NEXTHOP_ECMP` set, the prefix ordering or flag state after repeated `prefix_evaluate()` calls during convergence could leave cc2 without the expected `dmetric` or `nhflags`.

**Unverified.** Need debug logging to confirm which check in `rde_softreconfig_sync_fib` fails for `/24` prefixes.

### Alternative Theory: Stabilization No-op During Convergence

During initial convergence after restart, eBGP UPDATEs for `/24` prefixes (from GTT, HE) arrive interleaved with iBGP updates. Each eBGP update triggers `prefix_evaluate()` for the same prefix. The ECMP stabilization code at `rde_decide.c:584-593`:

```c
if (ecmp && ep != NULL &&
    ep->dmetric == PREFIX_DMETRIC_ECMP &&
    new != ep && new != newbest) {
    ; /* no-op for unrelated updates */
}
```

An eBGP update where `new` is not cc1 or cc2 triggers the no-op. If this fires AFTER ECMP was set up in the RIB but BEFORE the kroute was sent to the kernel, the FIB never gets updated. The delayed 20s resync should catch this, but if `rde_softreconfig_sync_fib` also fails (theory above), ECMP is never installed.

`/32` routes don't have this problem because they have NO eBGP paths - the only updates are from cc1 and cc2, so `new == ep || new == newbest` is always true.

---

## Debug Plan for Next Session

### Step 1: Add logging to rde_softreconfig_sync_fib (rde.c:4425)

Log the state for EVERY prefix that has a `fib-multipath` filter match:
```c
static void
rde_softreconfig_sync_fib(struct rib_entry *re, void *bula)
{
    struct prefix *p, *ep;
    struct bgpd_addr addr;

    if ((p = prefix_best(re)) == NULL)
        return;

    ep = TAILQ_NEXT(p, rib_l);

    /* Debug: log ECMP-eligible prefixes */
    if (prefix_nhflags(p) & NEXTHOP_ECMP) {
        pt_getaddr(p->pt, &addr);
        log_info("ECMP-sync %s/%u: best_nh=%s best_dmetric=%d best_nhflags=0x%x "
            "ep=%s ep_dmetric=%d ep_nhflags=0x%x",
            log_addr(&addr), p->pt->prefixlen,
            log_addr(&prefix_nexthop(p)->exit_nexthop),
            p->dmetric, p->nhflags,
            ep ? log_addr(&prefix_nexthop(ep)->exit_nexthop) : "NULL",
            ep ? ep->dmetric : -1,
            ep ? ep->nhflags : 0);
    }
    /* ... existing code ... */
}
```

This will show EXACTLY which check fails for `/24` vs `/32`.

### Step 2: Add logging to rde_send_kroute (rde.c:3575)

Log when ECMP kroutes are sent (or not):
```c
if (type == IMSG_KROUTE_CHANGE && (kf.flags & F_ECMP)) {
    log_info("ECMP kroute: %s/%u nexthop=%s flags=0x%x",
        log_addr(&kf.prefix), kf.prefixlen,
        log_addr(&kf.nexthop), kf.flags);
}
```

### Step 3: Build debug binary, deploy to pm1.mia1 ONLY

```bash
# Build with debug logging
scp openbgpd-doxx.tar.gz blyon@pm1.mia1.doxx.net:/tmp/
ssh blyon@pm1.mia1.doxx.net
cd /tmp && tar xzf openbgpd-doxx.tar.gz && cd openbgpd
./autogen.sh && ./configure && make -j4
sudo mv /usr/sbin/bgpd /usr/sbin/bgpd.bak
sudo mv src/bgpd/bgpd /usr/sbin/bgpd
sudo systemctl restart inet-bgpd
# Wait 25s for resync, then check logs:
sudo journalctl -u inet-bgpd --no-pager | grep ECMP-sync
```

### Step 4: Compare /32 vs /24 log output

The debug logs will reveal which of these is true:
- A) `rde_softreconfig_sync_fib` is never called for `/24` entries
- B) It's called but `ep->dmetric != PREFIX_DMETRIC_ECMP`
- C) It's called but `prefix_nhflags(ep) & NEXTHOP_ECMP` is false
- D) It's called and sends kroutes, but `kr_change4` doesn't install multipath

Each failure mode has a different fix.

---

## History: What We Broke and Fixed on June 13

### Morning: peer_dump() Always Change (commit 764873d)

Changed `peer_up()` in `rde_peer.c` to always call `peer_dump()` instead of `peer_blast()`.
Intended to fix stale Adj-RIB-Out after `bgpctli reload`. Introduced two bugs:

**Bug 1: peer_dump() doesn't send routes to reconnecting peers.**
When a PM reconnects to the CCs, the CC's `peer_dump()` walks Loc-RIB but pm2 receives ZERO routes. cc1 shows routes as `AI*` in `rib out` but nothing reaches the PM. Static `network` entries (207.207.200.0/24) and dynamic routes both missing.

**Bug 2: ECMP broken for routes arriving at different times.**
With `peer_dump()`, routes arrive one CC at a time. The second CC's path hits the stabilization no-op and the delayed resync fails to install multipath.

### Evening: Revert and Recovery

1. **Manual ECMP route fix** on both PMs (`ip route replace ... nexthop ... nexthop ...`) to restore traffic immediately
2. **Reverted peer_dump() change** (commit `d3534a2`) to restore `peer_blast()` behavior
3. **Hardcoded socket path** to `/var/run/bgpd.sock` (was building as `/usr/local/var/run/`)
4. **Deployed reverted binary** to all four MIA1 servers (pm1, pm2, cc1, cc2)
5. **Validated**: `/32` ECMP works automatically via `peer_blast()`, `/24` ECMP required manual fix

### Attempted ECMP fix in rde_decide.c (reverted, NOT deployed)

Added `new != ep` check to distinguish "new ECMP member joining" vs "unrelated iBGP update".
This broke the entire route evaluation path - pm2 received ZERO routes from any peer. Reverted.

---

## Related Files

- `src/bgpd/rde_peer.c` - `peer_up()`, `peer_blast()`, `peer_dump()` session establishment
- `src/bgpd/rde_decide.c:570-619` - ECMP stabilization and FIB update logic
- `src/bgpd/rde.c:4425-4463` - `rde_softreconfig_sync_fib()` delayed ECMP resync
- `src/bgpd/rde.c:3575-3697` - `rde_send_kroute()` F_ECMP flag and sibling loop
- `src/bgpd/kroute-linux.c:440-560` - `kr_change4()` multipath nexthop chaining
- `src/bgpd/bgpd.h:85-88` - `RUNSTATEDIR` / `SOCKET_NAME` hardcoded path fix
- `src/bgpd/rde_filter.c:207-208` - `ACTION_SET_ECMP` sets `NEXTHOP_ECMP` from `fib-multipath`
- wg-server `InetBGPServiceRouteCheck()` in `device.go` - re-injection watchdog for `/32` and `/128` only
- `/24` and `/48` routes are static `network` lines in `/etc/inet-bgpd.conf` on CCs, NOT injected by wg-server
- `01A-Planning/CC-BGPD_announce_issue.md` in doxx.net-private - full P0 postmortem
