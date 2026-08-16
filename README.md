# wgout

Run a single program outside your WireGuard tunnel, while everything else stays tunneled.

```console
$ curl -s ifconfig.me
203.0.113.10          # VPN exit

$ wgout curl -s ifconfig.me
198.51.100.42         # your real address
```

Works with **WireGuard** (`wg`) and **AmneziaWG** (`awg`). Linux only.

## Why

The official WireGuard client on Linux has no per-application split tunneling. `AllowedIPs` splits traffic by destination, which does not help when you want a specific program to bypass the tunnel: a game for latency, a torrent client, a corporate VPN client that conflicts with yours.

The usual workaround is a network namespace, which requires moving the program into a separate netns and dealing with its own resolver and interface setup. `wgout` needs neither: it is a prefix command, like `nice` or `proxychains`.

## How it works

`wg-quick` with `AllowedIPs = 0.0.0.0/0` does not install a default route in the main table. Instead it adds two rules:

```
32764: not from all fwmark 0xca6c lookup 51820    <- unmarked traffic to the tunnel
32766: from all lookup main                       <- marked traffic lands here
```

The fwmark exists so WireGuard's own encrypted packets are not routed back into the tunnel. That is the whole trick: mark an application's traffic with the **same** fwmark, rule 32764 stops matching it, and it falls through to the main table and out via the normal gateway. No additional routing table is created.

The application is identified by **cgroup v2**, not by process name: the process is placed in a dedicated cgroup and nftables matches on `socket cgroupv2`. This survives `fork`/`exec` and covers every child process.

This is the same mechanism [Mullvad uses for split tunneling on Linux](https://deepwiki.com/mullvad/mullvadvpn-app/3.5-split-tunneling).

## Requirements

- Linux with cgroup v2 (default on every current distribution)
- `nftables`
- `wireguard-tools` or `amneziawg-tools`
- `sudo` to install the rule — the program itself runs as your user, not as root

## Install

```sh
sudo install -m 0755 wgout /usr/local/bin/wgout
```

## Usage

```sh
wgout firefox               # run a program outside the tunnel
wgout --status              # show current state
wgout --stop                # remove rules and cgroup
wgout --version
```

The interface is detected automatically. Override it when several tunnels are up:

```sh
WGOUT_IFACE=awg0 wgout qbittorrent
```

Rules persist between invocations so repeated calls are cheap. `--stop` removes them.

## Verify

```sh
curl ifconfig.me            # VPN address
wgout curl ifconfig.me      # your own address
```

If both print the same address, the bypass did not take effect. Check `wgout --status` and the packet counter in the `nftables` output.

## Caveats

**DNS stays in the tunnel.** Queries from the bypassed program are deliberately not marked. The resolver handed out by the tunnel usually lives inside it and is unreachable from outside, so marking port 53 would leave the program unable to resolve anything. The trade-off: DNS queries remain visible to the VPN, and a CDN may return an address optimal for the tunnel exit rather than for the direct path. This rarely matters for games or torrents.

**Only useful for full-tunnel setups.** If `AllowedIPs` is already narrowed, `wg-quick` does not use a fwmark and there is nothing to bypass. `wgout` detects this and says so.

**IPv4 and IPv6 both work**, since the rule lives in an `inet` table and only sets a mark.

**One bypass group.** All programs launched through `wgout` share a single cgroup. There is no per-program policy.

## License

MIT
