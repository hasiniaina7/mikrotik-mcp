# MikroTik WireGuard RadiusDesk Runbook

SECURITY SENSITIVE - keep deployment values outside Git unless they are redacted.

This is the reusable baseline for connecting a MikroTik router to RadiusDesk /
FreeRADIUS through WireGuard while keeping normal client Internet traffic on the
router's WAN uplink. Optional management access from VPN devices to PPPoE client
subnets is allowed when explicitly required.

Validated on 2026-07-07 with:

| Router LAN IP | WireGuard NAS IP | Public key | Result |
| --- | --- | --- | --- |
| `192.168.1.47` | `10.5.1.14` | `oiGfOhjoKp71Vtc9hUdasRHqcRiypS16BSkK3rv95Sg=` | WireGuard, RADIUS source, server ping, and API `8728` validated |
| `192.168.1.124` | `10.5.1.10` | `4lHiQ2393+4lMUP7AhLD8siUgGycPam+YAKZZ0BeZiQ=` | WireGuard/API validated; hotspot blocked by device-mode physical confirmation |
| existing reference | `10.5.1.16` | `VcsrpBqBkjJHjqeLjN/xPLLiOnc6qLrBAKb4Og6GmyY=` | Working reference router |

## Design Contract

The tunnel is for router-to-RadiusDesk control traffic plus explicitly declared
VPN/admin reachability:

- MikroTik RADIUS requests go to `10.5.1.1` through WireGuard.
- RadiusDesk can reach the router WireGuard IP on MikroTik API TCP `8728`.
- VPN devices on the `10.5.1.0/24` side may reach PPPoE client subnets when
  those subnets are deliberately added to the server peer `AllowedIPs`.
- Client Internet traffic must keep using the router WAN default route.
- Do not install `AllowedIPs = 0.0.0.0/0, ::/0` on RouterOS for this design.
- Server-side WireGuard `AllowedIPs` must include `<ROUTER_WG_IP>/32` and may
  include the router's PPPoE subnet, for example `172.16.14.0/24`, when VPN
  clients need to access PPPoE devices behind that router.

This point is non-negotiable: do not confuse targeted reachability with a
full-tunnel design. Adding `172.16.x.0/24` to one router peer is acceptable for
VPN/admin access to PPPoE clients. Adding `0.0.0.0/0` or copying unrelated LAN
subnets blindly is not acceptable.

## Variables

Set these per router:

```bash
ROUTER_SSH_KEY=/home/mastershark-linux/ssh/mikrotik_admin
ROUTER_SSH_USER=admin-ssh
ROUTER_LAN_IP=<router-lan-management-ip>

ROUTER_WG_INTERFACE=wg1
ROUTER_WG_LISTEN_PORT=13231
ROUTER_WG_PRIVATE_KEY='<router-private-key>'
ROUTER_WG_PUBLIC_KEY='<derived-router-public-key>'
ROUTER_WG_ADDRESS=<router-wg-ip>/24
ROUTER_WG_NAS_IP=<router-wg-ip>
ROUTER_PPPOE_SUBNET=<router-pppoe-subnet-or-empty>
```

Shared RadiusDesk side:

```bash
SERVER_SSH_KEY=/home/mastershark-linux/ssh/key-not-for-faneva.pem
SERVER_SSH_USER=ubuntu
SERVER_HOST=ec2-13-247-123-11.af-south-1.compute.amazonaws.com
SERVER_PUBLIC_IP=13.247.123.11
SERVER_WG_INTERFACE=wg1
SERVER_WG_IP=10.5.1.1/24
SERVER_WG_PORT=51821
SERVER_WG_PUBLIC_KEY=sRRYLe7zedWwvW7WEDdXYYa94E65XAr9Oi9DXvfAn3M=

RADIUS_SERVER_IP=10.5.1.1
RADIUS_SECRET='<radius-shared-secret>'
```

## Preflight

Run these before changing anything:

```bash
ssh -i "$ROUTER_SSH_KEY" "$ROUTER_SSH_USER@$ROUTER_LAN_IP" \
  '/system identity print; /system resource print; /ip address print detail; /ip route print detail; /interface wireguard print detail; /radius print detail; /ip hotspot print detail; /system device-mode print'

ssh -i "$SERVER_SSH_KEY" "$SERVER_SSH_USER@$SERVER_HOST" \
  'sudo wg show wg1; sudo grep -F "<router-public-key>" -A1 /etc/wireguard/wg1.conf || true'
```

Confirm:

- RouterOS is v7 and supports WireGuard.
- The requested `ROUTER_WG_NAS_IP` is not already used by another peer.
- The router has a WAN default route that must remain unchanged.
- Hotspot device-mode is `hotspot=yes`; otherwise a physical confirmation is
  required after `/system device-mode update hotspot=yes`.

## Server Peer

The server peer must use the router public key. Minimal RadiusDesk/API access
uses only the router WireGuard IP:

```bash
sudo wg set wg1 peer "$ROUTER_WG_PUBLIC_KEY" allowed-ips "$ROUTER_WG_NAS_IP/32"
```

If VPN devices must reach PPPoE clients behind the router, add that PPPoE subnet:

```bash
sudo wg set wg1 peer "$ROUTER_WG_PUBLIC_KEY" allowed-ips "$ROUTER_WG_NAS_IP/32,$ROUTER_PPPOE_SUBNET"
```

Persist it in `/etc/wireguard/wg1.conf`:

```ini
[Peer]
PublicKey = <router-public-key>
AllowedIps = <router-wg-ip>/32
```

or, with PPPoE access:

```ini
[Peer]
PublicKey = <router-public-key>
AllowedIps = <router-wg-ip>/32,<router-pppoe-subnet>
```

Do not add hotspot LAN, bridge LAN, or default routes unless there is a separate
reviewed requirement.

## MikroTik Configuration

Apply this idempotent RouterOS script with the variables replaced:

```routeros
:if ([:len [/interface/wireguard/find name="wg1"]] = 0) do={
  /interface/wireguard/add name="wg1" mtu=1420 listen-port=13231 private-key="<router-private-key>" comment="RadiusDesk WireGuard"
} else={
  /interface/wireguard/set [find name="wg1"] mtu=1420 listen-port=13231 private-key="<router-private-key>" comment="RadiusDesk WireGuard"
}

:if ([:len [/ip/address/find where interface="wg1" and address="<router-wg-ip>/24"]] = 0) do={
  /ip/address/add address=<router-wg-ip>/24 interface=wg1 comment="RadiusDesk WireGuard"
}

:if ([:len [/interface/wireguard/peers/find where interface="wg1" and public-key="sRRYLe7zedWwvW7WEDdXYYa94E65XAr9Oi9DXvfAn3M="]] = 0) do={
  /interface/wireguard/peers/add interface=wg1 public-key="sRRYLe7zedWwvW7WEDdXYYa94E65XAr9Oi9DXvfAn3M=" endpoint-address=13.247.123.11 endpoint-port=51821 allowed-address=10.5.1.0/24 persistent-keepalive=25s comment="RadiusDesk wg1"
} else={
  /interface/wireguard/peers/set [find where interface="wg1" and public-key="sRRYLe7zedWwvW7WEDdXYYa94E65XAr9Oi9DXvfAn3M="] endpoint-address=13.247.123.11 endpoint-port=51821 allowed-address=10.5.1.0/24 persistent-keepalive=25s comment="RadiusDesk wg1"
}

:if ([:len [/interface/list/member/find where list="LAN" and interface="wg1"]] = 0) do={
  /interface/list/member/add list=LAN interface=wg1 comment="Allow RadiusDesk/API over WireGuard"
}

/radius set [find where address=10.5.1.1] secret="<radius-shared-secret>" src-address=<router-wg-ip> timeout=1100ms require-message-auth=yes-for-request-resp
```

Router-side `allowed-address=10.5.1.0/24` gives the router a route to the
RadiusDesk WireGuard subnet. It does not replace the WAN default route.

## Validation

Run from the router:

```routeros
/ping 10.5.1.1 src-address=<router-wg-ip> count=5
/interface/wireguard/peers print detail
/ip route print detail where dst-address=0.0.0.0/0
/ip route print detail where dst-address=10.5.1.0/24
/radius print detail
```

Expected:

- Ping to `10.5.1.1` has `0%` loss.
- WireGuard peer has a recent `last-handshake`.
- Default route remains via WAN, for example `192.168.1.1%ether1`.
- `10.5.1.0/24` is connected via `wg1`.
- RADIUS has `address=10.5.1.1`, correct secret, and
  `src-address=<router-wg-ip>`.

Run from the server:

```bash
ping -c 4 -W 2 <router-wg-ip>
timeout 4 bash -lc 'cat < /dev/null > /dev/tcp/<router-wg-ip>/8728' \
  && echo API_8728_OPEN || echo API_8728_CLOSED
sudo wg show wg1 | sed -n '/<router-public-key>/,/^$/p'
```

Expected:

- Ping succeeds.
- `API_8728_OPEN` is printed.
- Server peer shows `allowed ips: <router-wg-ip>/32` and, when required, the
  router PPPoE subnet.

If PPPoE reachability is required, validate server routing:

```bash
ip route get <pppoe-client-or-pool-ip>
```

Expected: the route uses `dev wg1`.

Confirm normal client traffic is not tunneled:

```routeros
/tool traceroute 8.8.8.8 count=1
/tool traceroute 10.5.1.1 count=1
```

Expected:

- `8.8.8.8` first hop is the router's WAN gateway, not `wg1`.
- `10.5.1.1` resolves directly through the WireGuard route.

## RadiusDesk Checks

The RadiusDesk NAS row must exist before real authentication can work:

```sql
SELECT id,nasname,shortname,type,secret,cloud_id
FROM nas
WHERE nasname='<router-wg-ip>';
```

Required:

- `nasname` equals the router WireGuard NAS IP.
- `secret` matches the MikroTik `/radius` secret.
- `type` is suitable for MikroTik API integration.
- `cloud_id` is assigned to the intended cloud. `cloud_id=-1` is suspicious
  for production and must be corrected or explicitly accepted by the operator.

## Validated 192.168.1.47 Result

The router at `192.168.1.47` was configured on 2026-07-07 with:

```text
WireGuard IP: 10.5.1.14/24
Router public key: oiGfOhjoKp71Vtc9hUdasRHqcRiypS16BSkK3rv95Sg=
Server peer allowed ips: 10.5.1.14/32,172.16.14.0/24
Router peer allowed-address: 10.5.1.0/24
PPPoE pool: 172.16.14.10-172.16.14.250
RADIUS src-address: 10.5.1.14
Hotspot device-mode: hotspot=yes
```

Validation output summary:

```text
MikroTik -> 10.5.1.1 ping: 5 sent, 5 received, 0% loss
Server -> 10.5.1.14 ping: 4 sent, 4 received, 0% loss
Server -> 10.5.1.14:8728: API_8728_OPEN
Server route to 172.16.14.10: dev wg1
Default route: 0.0.0.0/0 via 192.168.1.1%ether1
WireGuard route: 10.5.1.0/24 via wg1
RadiusDesk NAS row: id=77, nasname=10.5.1.14, type=Mikrotik-API, cloud_id=-1
```

The remaining production risk is not the tunnel. It is RadiusDesk ownership:
`cloud_id=-1` must be reviewed before relying on this NAS in production.
