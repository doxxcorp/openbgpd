# OpenBGPd - doxx.net

doxx.net OpenBGPd with ECMP multipath FIB support for Linux.

Repository: `github.com:doxxcorp/openbgpd.git`

## Platform

Linux x86-64 (Debian/Ubuntu). All doxx.net servers.

## Build

```bash
apt install build-essential automake autoconf libtool bison pkg-config libmnl-dev
./autogen.sh && ./configure && make -j4
```

Binaries: `src/bgpd/bgpd` (daemon), `src/bgpctl/bgpctl` (control client).

## Documentation

- [DEPLOY_AND_TESTING.md](DEPLOY_AND_TESTING.md) - build, test, deploy guide
- [ECMP_bugs.md](ECMP_bugs.md) - ECMP FIB install bug tracking and debug plan
