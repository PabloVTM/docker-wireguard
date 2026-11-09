# wireguard (fork)

A Docker image running [WireGuard®](https://www.wireguard.com/) in server or client mode.

This is a fork of [linuxserver/docker-wireguard](https://github.com/linuxserver/docker-wireguard), maintained at [PabloVTM/docker-wireguard](https://github.com/PabloVTM/docker-wireguard). The base image, s6 init and the WireGuard setup logic come from upstream.

## Requirements

* Docker with the WireGuard kernel module available on the host (built into most current kernels). If it is not loaded, load it on the host first, or give the container `SYS_MODULE` and mount `/lib/modules`.
* Network access to the UDP port you publish (default `51820`).

## Image

Images are published to GitHub Container Registry for `linux/amd64` and `linux/arm64`:

| Tag | Content |
| --- | --- |
| `ghcr.io/pablovtm/wireguard:latest` | Latest build of `master`. |
| `ghcr.io/pablovtm/wireguard:YYYY-MM-DD` | Dated version, e.g. `2026-11-06`. Each build upgrades the Alpine packages, so a new dated version also brings the latest security fixes. |

To use the published image instead of building locally, replace `build: .` with `image: ghcr.io/pablovtm/wireguard:latest` below.

## Quick start

```yaml
---
services:
  wireguard:
    build: .
    container_name: wireguard
    cap_add:
      - NET_ADMIN
      - SYS_MODULE #optional
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - SERVERURL=wireguard.domain.com #optional
      - SERVERPORT=51820 #optional
      - PEERS=1 #optional, enables server mode
      - PEERDNS=auto #optional
      - INTERNAL_SUBNET=10.13.13.0 #optional
      - ALLOWEDIPS=0.0.0.0/0 #optional
      - PERSISTENTKEEPALIVE_PEERS= #optional
      - LOG_CONFS=true #optional
    volumes:
      - ./config:/config
      - /lib/modules:/lib/modules #optional
    ports:
      - 51820:51820/udp
    sysctls:
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```

Build and start:

```bash
docker compose up -d --build
```

## Server mode

Set `PEERS` to a number (`PEERS=3`) or a comma-separated list of names (`PEERS=myPC,myPhone`). The container then generates the server config and one config per peer.

* Peer configs, QR codes (PNG) and text files are written to `/config/peerX` (or `/config/peer_<name>`).
* Set `LOG_CONFS=true` to print the QR codes in the container log.
* To add peers later, increase `PEERS` or add names, then recreate the container.
* To show a peer's QR code again: `docker exec -it wireguard /app/show-peer 1 4 5` or `docker exec -it wireguard /app/show-peer myPC myPhone`.
* Templates for generated configs are in `/config/templates`. To regenerate configs after editing them, delete `/config/wg_confs/wg0.conf` and restart.
* Changing `SERVERURL`, `SERVERPORT`, `INTERNAL_SUBNET`, `PEERDNS`, `INTERFACE`, `ALLOWEDIPS` or `PERSISTENTKEEPALIVE_PEERS` regenerates the configs. Existing keys are kept; delete a peer's folder to regenerate its keys.

The server config is always `wg0.conf`. Extra tunnels can be added as `.conf` files in `/config/wg_confs/`. They start in alphabetical order, and if one fails, all tunnels stop and the default route is removed until the config is fixed and the container restarted.

## Client mode

Leave `PEERS` unset. Put your client config in `/config/wg_confs/<tunnel name>.conf` and start the container. Multiple `.conf` files start in alphabetical order, with the same failure behaviour as above.

If the log shows IPv6 errors and the connection fails, change `AllowedIPs` in the client config to `0.0.0.0/0` (remove `::/0`) and restart.

Client mode does not support read-only containers.

## Environment variables

| Variable | Default | Description |
| --- | --- | --- |
| `PUID` / `PGID` | `1000` | User and group that own files in `/config`. |
| `TZ` | `Etc/UTC` | Timezone, see the [tz database list](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones#List). |
| `SERVERURL` | — | Public IP or domain of the host. Use `auto` to detect the external IP. Server mode. |
| `SERVERPORT` | `51820` | Public UDP port of the host. Server mode. |
| `PEERS` | — | Number of peers, or a list of alphanumeric names. Setting it enables server mode. |
| `PEERDNS` | `auto` | DNS set in peer configs. `auto` uses the host's DNS through the bundled CoreDNS. Server mode. |
| `INTERNAL_SUBNET` | `10.13.13.0` | Internal WireGuard subnet. Change only if it clashes with your network. Server mode. |
| `ALLOWEDIPS` | `0.0.0.0/0` | Networks peers route through the tunnel. Use specific subnets for split tunnelling. |
| `PERSISTENTKEEPALIVE_PEERS` | — | `all` or a comma-separated list of peers that send keepalives every 25 s. Useful with dynamic IPs. Server mode. |
| `LOG_CONFS` | `true` | Print peer QR codes in the log. |
| `SERVER_ALLOWEDIPS_PEER_<name or number>` | — | Extra subnets added to one peer's `AllowedIPs`, comma-separated. Applied on config regeneration. |
| `UMASK` | — | Umask for services, e.g. `022`. |
| `FILE__<VAR>` | — | Read `<VAR>` from a file, e.g. for Docker secrets. |

## Volumes, ports and capabilities

| Setting | Purpose |
| --- | --- |
| `-p 51820:51820/udp` | WireGuard port. |
| `./config:/config` | Configs, keys, peer files. |
| `/lib/modules:/lib/modules` | Host kernel modules, if WireGuard is not already loaded (optional). |
| `--cap-add=NET_ADMIN` | Needed to create the VPN interface. |
| `--cap-add=SYS_MODULE` | Needed only to load the kernel module (optional). |
| `--sysctl=net.ipv4.conf.all.src_valid_mark=1` | Needed for client mode. |

## Common setups

### iptables modules not loaded on the host

Some hosts do not load the iptables kernel modules by default. Give the container `SYS_MODULE` and mount `/lib/modules`, or run `modprobe` on the host before starting.

### Accessing a container's web UI locally while routing through WireGuard

This is not supported by upstream and is used at your own risk. Add the local subnets to the `[Interface]` section of the client `wg0.conf`, so they bypass the tunnel:

```ini
[Interface]
PrivateKey = <private key>
Address = 9.8.7.6/32
DNS = 8.8.8.8
PostUp = DROUTE=$(ip route | grep default | awk '{print $3}'); HOMENET=192.168.0.0/16; HOMENET2=10.0.0.0/8; HOMENET3=172.16.0.0/12; ip route add $HOMENET3 via $DROUTE;ip route add $HOMENET2 via $DROUTE; ip route add $HOMENET via $DROUTE;iptables -I OUTPUT -d $HOMENET -j ACCEPT;iptables -A OUTPUT -d $HOMENET2 -j ACCEPT; iptables -A OUTPUT -d $HOMENET3 -j ACCEPT;  iptables -A OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
PreDown = DROUTE=$(ip route | grep default | awk '{print $3}'); HOMENET=192.168.0.0/16; HOMENET2=10.0.0.0/8; HOMENET3=172.16.0.0/12; ip route del $HOMENET3 via $DROUTE;ip route del $HOMENET2 via $DROUTE; ip route del $HOMENET via $DROUTE; iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT; iptables -D OUTPUT -d $HOMENET -j ACCEPT; iptables -D OUTPUT -d $HOMENET2 -j ACCEPT; iptables -D OUTPUT -d $HOMENET3 -j ACCEPT
```

### Site-to-site VPN (server mode)

Not supported by upstream; use at your own risk. Set an extra subnet for a peer:

```bash
SERVER_ALLOWEDIPS_PEER_laptop="192.168.1.0/24,192.168.2.0/24"
```

This produces `AllowedIPs = 10.13.13.2,192.168.1.0/24,192.168.2.0/24` for the peer `laptop`. The change applies only when configs are regenerated: delete `wg0.conf` and restart to force it. Add the matching `PostUp`/`PostDown` rules to the client's config for LAN access.

### Roaming and returning home

If you use the VPN both remotely and at home, your router may not route forwarded WAN ports back to the LAN. Fix this with NAT reflection on the router or split-horizon DNS, which returns the private IP when you are on the LAN.

## Operations

```bash
# Shell in the container
docker exec -it wireguard /bin/bash

# Follow logs
docker logs -f wireguard

# Container build version
docker inspect -f '{{ index .Config.Labels "build_version" }}' wireguard
```

To update, rebuild the image (`docker compose build --no-cache`) and recreate the container. Your `/config` volume is kept.

## Releasing

Images are built by GitHub Actions. A push to `master` publishes `latest`; a dated version is published by pushing a tag named after the day:

```bash
git tag 2026-11-09
git push origin 2026-11-09
```

The workflow tests the image first, then publishes `ghcr.io/pablovtm/wireguard:2026-11-09` and `latest` for `linux/amd64` and `linux/arm64`. There is no scheduled rebuild: tag a new version when you want one.

## Building

```bash
docker build --no-cache --pull -t wireguard:latest .
```

For ARM images on x86 hardware (or the reverse), register QEMU first:

```bash
docker run --rm --privileged lscr.io/linuxserver/qemu-static --reset
docker build -f Dockerfile.aarch64 -t wireguard:arm64 .
```

## Credits

WireGuard is a trademark of Jason A. Donenfeld. The container base, s6 init and original setup scripts are from [LinuxServer.io](https://linuxserver.io), upstream at [linuxserver/docker-wireguard](https://github.com/linuxserver/docker-wireguard).
