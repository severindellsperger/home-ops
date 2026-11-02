# VyOS (LAN Router)

VyOS runs as a single privileged pod with `hostNetwork: true` on `dels-cp-0` and is the router for the LAN (`10.0.97.0/24`, Wi-Fi access points).
It provides DHCP, DNS forwarding, NAT, the WAN firewall (IPv4 + IPv6) and NTP.

It is **not** managed by Flux: the LAN has no internet before VyOS runs, so it is applied manually during the cluster setup (see [`talos/README.md`](../../talos/README.md#11-vyos-lan-router)).

| File | Content |
|---|---|
| `cm.yaml` | ConfigMap `vyos-config` with the VyOS `config.boot` (the complete router configuration) |
| `vyos.yaml` | PVC `vyos-state` and Deployment `vyos` |

## Who does what

| Talos (machine config) | VyOS (`config.boot`) |
|---|---|
| Interfaces, bridges and IP addresses: `br-lan` (`enp3s0` + `enp4s0`, `10.0.97.1/24`), `br-mgmt` (`172.16.254.1/24`) | DHCP server for the LAN (`10.0.97.50-150`, static leases for the APs `.10`/`.11`) |
| WAN address via DHCP (`enp7s0f1np1`), IPv6 via SLAAC | DNS forwarding on `10.0.97.1` for the LAN |
| IP forwarding (`SysctlConfig`) | NAT masquerade `10.0.97.0/24` → WAN |
| Kernel modules (`nf_tables`, `nft_nat`, `nft_masq`, `bridge`, ...) | IPv4/IPv6 firewall for traffic from the WAN |
| | NTP for the LAN, SSH on `10.0.97.1` |

VyOS doesn't configure addresses on interfaces. The interfaces in `config.boot` only carry descriptions, the addresses come from Talos.
Interface names in `config.boot` must match the Talos interface names (they can change when PCIe devices are added, check with `talosctl get links`).

## How it works

- **Config (`cm.yaml`)**: Only the `config.boot` key of the ConfigMap is mounted (`subPath`, read-only) at `/opt/vyatta/etc/config/config.boot`, where VyOS loads its boot config from.
  It is loaded **once at pod start** and is the only source of truth.
  Changes made live with `configure` / `commit` work immediately, but are lost on the next pod restart (`save` fails as the file is read-only).
- **State (`vyos-state` PVC)**: Mounted at `/opt/vyatta/etc/config` (local-path storage). Holds runtime state, e.g. DHCP leases (`dhcp/`) and the commit archive (`archive/`).
  At boot VyOS bind-mounts `/opt/vyatta/etc/config` onto `/config`, so the directory itself must be writable: if it were read-only (e.g. the whole ConfigMap mounted there), `service dhcp-server` and `system config-management` fail to commit and are silently missing from the running config.
- **Host network**: VyOS works directly in the node's network namespace. Its nftables tables (`ip vyos_filter`, `ip6 vyos_filter`, `ip vyos_nat`, ...) live next to the Talos table (`inet talos`) and Cilium's rules.
- **Init container `flush-vyos-nft`**: Because the nftables tables belong to the host, they survive a pod restart. The init container deletes the old `vyos_*` tables before VyOS starts, so VyOS can recreate them cleanly from `config.boot`.
  Consequence: if VyOS fails to load the config, the node has **no** VyOS firewall/NAT at all. Always verify after a restart.
- **`postStart` hook**: Replaces `modprobe` / `rmmod` with `/bin/true`. Kernel modules can't be loaded from the container; Talos loads them (`KernelModuleConfig`).
- **`hostAliases`**: With `hostNetwork` the pod has the node hostname `dels-cp-0`, which is added to `/etc/hosts` (otherwise `sudo` complains `unable to resolve host`).

## Firewall and NAT

| Rule set | Hooked into | Rules |
|---|---|---|
| `ipv4 name WAN-IN` | `input` + `forward` filter for traffic from `enp7s0f1np1` | accept established/related, drop invalid, **default drop** |
| `ipv6 name WAN6-IN` | `input` + `forward` filter for traffic from `enp7s0f1np1` | accept established/related, drop invalid, accept ICMPv6, **default drop** |
| `nat source rule 100` | postrouting | masquerade `10.0.97.0/24` out of `enp7s0f1np1` |

- `input` protects the node itself (Talos API `50000`, Kubernetes API `6443`, kubelet `10250`, ...), `forward` protects the LAN.
- ICMPv6 must be allowed: the node gets its public IPv6 address via router advertisements (SLAAC), and neighbor discovery and path MTU discovery depend on ICMPv6.
- Traffic from LAN/management interfaces is not filtered (default accept).

## Deploy and change the config

Initial deployment (namespace with privileged Pod Security, as VyOS needs privileged host access):
```bash
kubectl create namespace priviledged
kubectl label namespace priviledged pod-security.kubernetes.io/enforce=privileged
kubectl apply -f infra/vyos/cm.yaml -f infra/vyos/vyos.yaml
kubectl -n priviledged rollout status deploy/vyos
```

Changing the config: edit `cm.yaml`, apply and restart (the LAN loses connectivity for 1-2 minutes, management access is not affected):
```bash
kubectl apply -f infra/vyos/cm.yaml
kubectl -n priviledged rollout restart deploy/vyos
kubectl -n priviledged rollout status deploy/vyos
```

Tip: test a change live first (`configure`, `set ...`, `commit-confirm 5`, `confirm`), then copy the result into `cm.yaml` (`show configuration` prints it in `config.boot` format).

## Access

```bash
kubectl -n priviledged exec -it deploy/vyos -- su - vyos    # VyOS CLI (op mode, `configure` for config mode)
ssh admin@10.0.97.1                                          # from the LAN, SSH key only
```

## Verify

In the VyOS CLI:
```
show firewall                 # ipv4 + ipv6 rule sets with counters
show nat source rules         # rule 100 masquerade
show dhcp server leases
show interfaces
```

nftables on the host (from the VyOS pod or a node debug pod, e.g. `kubectl debug node/dels-cp-0 -n priviledged -it --profile=sysadmin --image=nicolaka/netshoot`):
```bash
nft list tables                                   # vyos_filter (ip/ip6), vyos_nat, ...
nft list table ip vyos_filter
nft list chain ip6 vyos_filter NAME6_WAN6-IN      # drop counter should increase
nft list table ip vyos_nat
```

External test: must run **from outside** (e.g. mobile hotspot). From the LAN, traffic to the WAN IP arrives on `br-lan` and never passes `WAN-IN`, so ports look open.
For IPv6 the scanning device needs real IPv6 connectivity (check with test-ipv6.com), otherwise `filtered` means nothing.
```bash
nmap -Pn -p 22,50000,6443,10250 <WAN_IPv4>        # expected: filtered
nmap -6 -Pn -p 22,50000,6443,10250 <WAN_IPv6>     # expected: filtered
```

### Known harmless log messages

| Message | Reason |
|---|---|
| `Config file '/opt/vyatta/etc/config/config.boot' not writeable`, `Read-only file system` | Config comes from the read-only ConfigMap |
| `PermissionError: ... /sys/kernel/debug/dynamic_debug` | No debugfs inside the container |
| `nfct ... netlink error: Invalid argument` | Conntrack timeout tuning not allowed from the container, kernel defaults apply |
| `perl: warning: Setting locale failed` | Missing locale in the image |

## Open points

- **IPv6 for LAN clients**: Currently only the node has IPv6 (SLAAC on the WAN). LAN clients would need a prefix via DHCPv6-PD from the ISP and router advertisements on `br-lan` (then the `WAN6-IN` forward rules become relevant and DHCPv6 replies (UDP 546) must be allowed on the WAN).
- **Firewall for LAN → node**: The LAN can reach all node services (Talos API, Kubernetes API, ...). Restrict to what is needed (DHCP, DNS, NTP, SSH).
- **SSH listens only on the LAN** (`10.0.97.1`): Consider adding `172.16.254.1` for access from the management network.
- **Pinned image**: `ghcr.io/severindellsperger/vyos-container:2026.03` (VyOS 2026.03, circinus). `config.boot` has no version footer, so VyOS runs all migration scripts on every boot; check the logs for migration errors after an image update.
- **Flux**: VyOS is applied manually. It could be added to Flux after the bootstrap (the manual apply is still needed for the first install).
- **VyManager**: A web UI for VyOS ([community-vyprojects](https://github.com/community-vyprojects)) that configures VyOS via its HTTPS API (REST/GraphQL, API key). A local docker-compose test setup exists under `tmp/vymanager/` (not in git, contains secrets).
  It conflicts with the GitOps approach: changes via the API are only live and are lost on the next pod restart, as `config.boot` comes from the read-only ConfigMap.
  Options if it is deployed: use it only for testing/viewing and copy changes into `cm.yaml`, or move the config to the PVC (persistent, but no longer declared in git).
