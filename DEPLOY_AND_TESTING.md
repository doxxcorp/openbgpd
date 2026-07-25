# OpenBGPd - Build, Test, and Deploy Guide

doxx.net OpenBGPd (`doxxcorp/openbgpd`) with ECMP multipath FIB support for Linux.

## Quick Reference

| Item | Value |
|---|---|
| Source repo | `github.com:doxxcorp/openbgpd.git` |
| Binary path on servers | `/usr/sbin/bgpd` (both mesh bgpd and inet-bgpd use the same binary) |
| Control sockets | `/usr/local/var/run/bgpd.sock.0` (mesh), `/usr/local/var/run/bgpd.sock.1` (inet) |
| Control binaries | `bgpctl` (mesh, port 178), `bgpctli` (inet, port 179) - both are symlinks or copies of the same `bgpctl` binary |
| Deploy packages on ops servers | `/var/lib/doxx/openbgpd-doxx.tar.gz` (source), `/var/lib/doxx/openbgpd-doxx-bins.tar.gz` (pre-built) |
| Ops servers | `ops1.phx1.doxx.net`, `ops2.mia1.doxx.net` |
| Infra API endpoint | `serve_openbgpd_source`, `serve_openbgpd_bins` |

## Building

### Prerequisites (Debian/Ubuntu)

```bash
apt install build-essential automake autoconf libtool bison pkg-config libmnl-dev
```

### Build from source

```bash
cd /tmp
tar xzf openbgpd-doxx.tar.gz
cd openbgpd
./autogen.sh
./configure
make -j4
```

Binaries are at:
- `src/bgpd/bgpd` - the BGP daemon
- `src/bgpctl/bgpctl` - the control client

### Cross-build note

The binary is architecture-specific (x86-64 ELF). Build on the same
architecture as the target servers. All doxx.net servers are x86-64 Linux.
Building on any PM or CC server works.

## Testing a Patch

### 1. Build on a server

Pick any server with build tools (PMs are good since they have the deps):

```bash
scp openbgpd-doxx.tar.gz blyon@pm1.pao1.doxx.net:/tmp/
ssh blyon@pm1.pao1.doxx.net
cd /tmp && tar xzf openbgpd-doxx.tar.gz && cd openbgpd
./autogen.sh && ./configure && make -j4
```

### 2. Deploy to a single server (use mv, not cp)

The binary is typically running. Use `mv` to atomically replace it - `cp` will
fail with "Text file busy":

```bash
sudo mv /usr/sbin/bgpd /usr/sbin/bgpd.bak
sudo mv /tmp/openbgpd/src/bgpd/bgpd /usr/sbin/bgpd
sudo chmod +x /usr/sbin/bgpd
```

### 3. Restart and validate

Both `bgpd` (mesh) and `inet-bgpd` (internet) use the same binary. Restart
both after replacing:

```bash
sudo mkdir -p /usr/local/var/run    # required for control socket
sudo systemctl restart inet-bgpd
sudo systemctl restart bgpd
```

Wait 10-15 seconds for BGP sessions to re-establish, then validate:

```bash
# Check inet-bgpd peers (internet peering)
sudo bgpctli show neighbor | grep -E 'neighbor is|state'

# Check mesh bgpd peers (backbone WireGuard)
sudo bgpctl show neighbor | grep -E 'neighbor is|state'

# Verify routes are being announced (on CC servers)
sudo bgpctli show rib out | head -20

# Verify routes received from CCs (on PM servers)
sudo bgpctli show rib neighbor <CC_IP> in
```

### 4. Rollback if needed

```bash
sudo systemctl stop inet-bgpd bgpd
sudo mv /usr/sbin/bgpd.bak /usr/sbin/bgpd
sudo systemctl start bgpd inet-bgpd
```

## Deploy Order

When rolling out a new binary across a site, deploy in this order to minimize
disruption. Never deploy to all servers in parallel - do them sequentially and
validate peers between each:

1. **PM servers first** (pm1, pm2) - they receive routes, less risk
2. **CC servers second** (cc1, cc2) - they originate service routes

After replacing bgpd on a CC, also restart `wg-server` to re-inject the
dynamic service routes into inet-bgpd (the wg-server watchdog will also catch
missing routes within 30 seconds).

## Creating the Bootstrap Deploy Package

The bootstrap system (`bootstrap_server`) fetches OpenBGPd from the infra API.
Two tarballs are maintained on both ops servers:

### Source tarball (`openbgpd-doxx.tar.gz`)

From your local checkout (excludes .git and build artifacts):

```bash
cd ~/go_projects
tar czf /tmp/openbgpd-doxx.tar.gz \
    --exclude='.git' --exclude='*.o' --exclude='*.lo' \
    --exclude='.libs' --exclude='.deps' \
    --exclude='autom4te.cache' --exclude='config.status' \
    --exclude='config.log' --exclude='Makefile' \
    --exclude='libtool' --exclude='stamp-h1' \
    openbgpd/
```

Structure: `openbgpd/` directory containing full source tree.

### Binary tarball (`openbgpd-doxx-bins.tar.gz`)

Flat tarball (no directory prefix) containing:

```
bgpd            # the daemon binary (x86-64 ELF, not stripped)
bgpctl          # the control client binary
RELEASE         # version string, e.g. "9.2-doxx3"
VERSION         # version string, e.g. "OpenBGPD 9.2"
checksums.txt   # sha256 checksums of bgpd and bgpctl
```

To create from a built binary:

```bash
mkdir -p /tmp/openbgpd-bins && cd /tmp/openbgpd-bins
# Copy binaries from build server or local build
scp blyon@pm1.pao1.doxx.net:/usr/sbin/bgpd .
scp blyon@pm1.pao1.doxx.net:/usr/sbin/bgpctl .
# Or from local build:
# cp /tmp/openbgpd/src/bgpd/bgpd .
# cp /tmp/openbgpd/src/bgpctl/bgpctl .

echo "9.2-doxx3" > RELEASE          # bump version on each release
echo "OpenBGPD 9.2" > VERSION
sha256sum bgpd bgpctl > checksums.txt

tar czf /tmp/openbgpd-doxx-bins.tar.gz bgpd bgpctl RELEASE VERSION checksums.txt
```

### Upload to ops servers

Both ops servers must have identical copies:

```bash
scp /tmp/openbgpd-doxx.tar.gz blyon@ops1.phx1.doxx.net:/tmp/
scp /tmp/openbgpd-doxx-bins.tar.gz blyon@ops1.phx1.doxx.net:/tmp/
ssh blyon@ops1.phx1.doxx.net "sudo mv /var/lib/doxx/openbgpd-doxx.tar.gz /var/lib/doxx/openbgpd-doxx.tar.gz.bak && \
    sudo mv /var/lib/doxx/openbgpd-doxx-bins.tar.gz /var/lib/doxx/openbgpd-doxx-bins.tar.gz.bak && \
    sudo mv /tmp/openbgpd-doxx.tar.gz /var/lib/doxx/ && \
    sudo mv /tmp/openbgpd-doxx-bins.tar.gz /var/lib/doxx/"

# Repeat for ops2.mia1.doxx.net
```

The infra API serves these via:
- `https://infra.doxx.net/?serve_openbgpd_source` - source tarball
- `https://infra.doxx.net/?serve_openbgpd_bins` - pre-built binaries

## Key Patches (doxx.net fork)

### peer_up() Adj-RIB-Out resync (a23bb98, 9.2-doxx4)

**Files:** `src/bgpd/rde_peer.c`, `src/bgpd/rde_adjout.c`, `src/bgpd/rde.h`

Fixes the silent re-announcement bug (pao1 June 11, mia1 July 22 P0s):
when a peer daemon restarts, the session re-establishes with changed
parameters (typically the graceful restart capability), which sent
`peer_up()` down the `peer_dump()` path. `peer_dump()` generates deltas
against the surviving Adj-RIB-Out, so a populated tree meant almost
nothing was re-sent to a peer whose table was actually empty.

The fix: on the force_sync path, silently flush the Adj-RIB-Out first
(new `adjout_prefix_flush()`, no wire withdraws), then dump the full
Loc-RIB per AID. An `adjout_dirty` flag persists across interrupted
resyncs. `peer_blast()` remains the fast path for unchanged flaps.
Every establishment now logs "replaying Adj-RIB-Out" (blast) or
"full Adj-RIB-Out resync (<reason>)" (flush+dump).

NOTE: an earlier attempt (764873d, June 13) forced `peer_dump()` for
EVERY reconnect - that made every flap silent and was reverted. Do not
reintroduce it.

### Graceful listener binds (d7f62d5)

`IP_FREEBIND`/`IPV6_FREEBIND` on listeners + non-fatal partial bind at
startup AND reload (upstream reload silently discarded the whole config
on one failed bind). Allows configs with listen addresses that are not
yet up (dark carrier ports, v6 discovery issues).

### ECMP multipath FIB support

Multiple patches for Linux ECMP multipath routing - see git log for details.

## Server Roles and BGP

Each site has PM (Packet Mover) and CC (Central Computer) servers:

- **PM servers** run inet-bgpd for internet peering (AS16740) and mesh bgpd
  for backbone WireGuard tunnels. They are the internet gateways.
- **CC servers** run inet-bgpd to announce service prefixes (DNS anycast,
  VPN pool, SNAT IPs) to PMs via iBGP. They also run mesh bgpd and wg-server.

inet-bgpd on CCs has two types of routes:
- **Static** (`S` flag in `bgpctli show network`): from `/etc/inet-bgpd.conf`
  `network` lines. Survive reloads.
- **Dynamic** (no `S` flag): injected by wg-server via `bgpctli network add`.
  Lost on inet-bgpd restart. The wg-server watchdog re-injects within 30s.
