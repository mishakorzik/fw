# fw
Easily block malicious traffic before it even reaches conntrack — and set per-port limits.

`fw` is a single-file, conntrack-free `nftables` firewall manager for Linux.
Instead of tracking every connection (`ct state established`), it drops junk
statelessly in the `raw` table with `notrack`: bad TCP flag combos, fragments,
short UDP, SYN floods — so a flood never fills the conntrack table or burns
CPU. On top of that: per-port/ICMP rate limits, timed blacklist/whitelist,
DNAT redirects and masquerade for VPN-gateway setups, plus automatic NIC
ring-buffer tuning for flood resilience.

- Requires: Linux with `nf_tables`, Python 3.11+, `nftables`
- Version: `1.0.8`
- Run as root: `fw` shells out to `nft` and `sysctl`

## Install
```
wget https://github.com/mishakorzik/fw/releases/download/1.0.8/fw-1.0.8.deb
sudo apt install ./fw-1.0.8.deb -y
rm fw-1.0.8.deb
```

## Quick start
```bash
fw allow 22/tcp --limit 50 --comment "ssh"
fw allow 80/tcp
fw allow 443/tcp
fw enable # load the ruleset into the kernel
fw status # show what's active
```

The box now accepts SSH/HTTP/HTTPS and drops everything else on input —
with SYN-flood rate limiting and zero connection-tracking overhead.

More:
```bash
fw allow icmp --limit 5            # let the box be pinged, rate-limited
fw deny 23/tcp                     # explicit drop (wins over allow)
fw blacklist add 1.2.3.4 --time 1h # timed block, enforced at ingress
fw check 8080                      # ALLOW / DENY / REDIRECT / DROP?
fw redirect add 8080/tcp --to 192.168.1.50:80
fw masquerade add --iface eth0 --from 10.8.0.0/24
```

## How it works
- `EARLY_DROP` (`prerouting`, priority `raw`): malformed/scan drops, blacklist,
  per-port SYN/packet limits, then `notrack` for everything except the narrow
  NAT/redirect flows that kernel-wise require conntrack.
- `EARLY_DROP_OUT` (`output`, priority `raw`): same `notrack` for locally
  generated traffic, so outbound floods/pings/DNS build no state either.
- `FW_IN` (`input`, policy `drop`): stateless admission — loopback, ICMP errors
  + echo-reply, DHCP, replies from `53/80/443`, then your allowed ports.
- NAT/forward chains appear only when you configure `redirect`/`masquerade`.
- `ethtool-ring-tune` sets NIC RX/TX rings to `min(driver max, 4096)` on every
  physical NIC at boot and on link-up, so bursts die in no queue before `nft`.

## Screenshots
<img width="99.9%" src="https://raw.githubusercontent.com/mishakorzik/fw/refs/heads/main/images/Screenshot_2025-08-19-11-51-55-806_com.termux-edit.jpg"/>
<img width="99.9%" src="https://raw.githubusercontent.com/mishakorzik/fw/refs/heads/main/images/Screenshot_2025-08-19-11-52-23-516_com.termux-edit.jpg"/>

## Donate

**If you want to donate, click on the button**
<a href="https://www.buymeacoffee.com/misakorzik"><img title="Donate" src="https://img.shields.io/badge/Donate-Firewall-red?style=for-the-badge&logo=github"></a>
