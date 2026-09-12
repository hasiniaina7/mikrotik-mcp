# MikroTik 192.168.1.124 WireGuard RadiusDesk Deployment Notes

SECURITY SENSITIVE - DO NOT COMMIT OR SHARE.

Reusable baseline: use `docs/mikrotik-wireguard-radiusdesk-runbook.md` for new
routers. This file is a router-specific deployment note and contains live
secrets.

This document contains live deployment values and secrets. It is intended as an operational handoff for another AI agent or engineer.

Collected and applied on 2026-07-07.

## Goal

Link the MikroTik router reachable at `192.168.1.124` to the RadiusDesk / FreeRADIUS server through WireGuard, then make the router reachable from the server on MikroTik API TCP `8728`.

Target topology:

```text
MikroTik 192.168.1.124
  WireGuard wg1: 10.5.1.10/24
  RADIUS source: 10.5.1.10
  RADIUS server: 10.5.1.1
  API: 10.5.1.10:8728

RadiusDesk EC2 / FreeRADIUS
  public: 13.247.123.11
  wg1: 10.5.1.1/24
  WireGuard listen: UDP 51821
```

## Mandatory Variables

Router access:

```bash
ROUTER_SSH_KEY=/home/mastershark-linux/ssh/mikrotik_admin
ROUTER_SSH_USER=admin-ssh
ROUTER_LAN_IP=192.168.1.124
```

Server access:

```bash
SERVER_SSH_KEY=/home/mastershark-linux/ssh/key-not-for-faneva.pem
SERVER_SSH_USER=ubuntu
SERVER_HOST=ec2-13-247-123-11.af-south-1.compute.amazonaws.com
SERVER_PUBLIC_IP=13.247.123.11
SERVER_WG_INTERFACE=wg1
SERVER_WG_IP=10.5.1.1/24
SERVER_WG_PORT=51821
SERVER_WG_PUBLIC_KEY=sRRYLe7zedWwvW7WEDdXYYa94E65XAr9Oi9DXvfAn3M=
```

Router WireGuard:

```bash
ROUTER_WG_INTERFACE=wg1
ROUTER_WG_LISTEN_PORT=13231
ROUTER_WG_PRIVATE_KEY='aN3k+ROvmMcrFmF8pV5+xe5fy4H7sMP4HUjnzzMUQWI='
ROUTER_WG_PUBLIC_KEY='4lHiQ2393+4lMUP7AhLD8siUgGycPam+YAKZZ0BeZiQ='
ROUTER_WG_ADDRESS=10.5.1.10/24
ROUTER_WG_NAS_IP=10.5.1.10
ROUTER_WG_ALLOWED_ADDRESS=10.5.1.0/24
```

RadiusDesk / FreeRADIUS:

```bash
RADIUS_SERVER_IP=10.5.1.1
RADIUS_AUTH_PORT=1812
RADIUS_ACCT_PORT=1813
RADIUS_SECRET='techzone2090'
NAS_IP=10.5.1.10
NAS_SHORTNAME=10-Imandry-remplacent
NAS_TYPE=Mikrotik-API
```

Important: the supplied WireGuard client config had `AllowedIPs = 0.0.0.0/0, ::/0`. Do not translate that into a MikroTik production route unless the design is intentionally full-tunnel. For RadiusDesk/RADIUS, use `allowed-address=10.5.1.0/24` on the MikroTik peer so the router keeps its WAN default route.

## Initial State Observed

Router:

```routeros
/system identity
name=MikroTik

/system resource
version=7.19.6 stable
board-name=hAP ax^2
architecture-name=arm64

/ip/address
192.168.88.1/24 interface=bridge comment=defconf
192.168.1.124/24 interface=ether1 dynamic=yes
192.168.15.1/24 interface=bridge comment="hotspot network"

/ip/route
0.0.0.0/0 gateway=192.168.1.1%ether1 dynamic
192.168.15.0/24 gateway=bridge connected
192.168.88.0/24 gateway=bridge connected
```

Hotspot, DHCP, RADIUS, PPP and PPPoE were already partially configured:

```routeros
/ip/hotspot
name=hotspot1 interface=bridge address-pool=hs-pool-9 profile=hsprof1 invalid=yes
comment="inactivated, not allowed by device-mode"

/ip/hotspot/profile
name=hsprof1 hotspot-address=192.168.15.1 dns-name=login.techzone.lat use-radius=yes radius-accounting=yes radius-interim-update=1m

/radius
service=ppp,hotspot address=10.5.1.1 secret="" protocol=udp

/ppp/aaa
use-radius=yes accounting=yes interim-update=1m

/interface/pppoe-server/server
service-name=service1 interface=bridge default-profile=default
```

Device mode was the blocker:

```routeros
/system/device-mode
mode=basic
hotspot=no
```

Server:

```bash
wg1 IP: 10.5.1.1/24
wg1 public key: sRRYLe7zedWwvW7WEDdXYYa94E65XAr9Oi9DXvfAn3M=
wg1 listen port: 51821
```

The server already had a peer for this router:

```text
peer public key: 4lHiQ2393+4lMUP7AhLD8siUgGycPam+YAKZZ0BeZiQ=
allowed ips: 10.5.1.10/32
```

RadiusDesk DB already had a NAS row:

```sql
SELECT id,nasname,shortname,type,secret,cloud_id FROM nas WHERE nasname='10.5.1.10';
```

Observed:

```text
id=73
nasname=10.5.1.10
shortname=10-Imandry-remplacent
type=Mikrotik-API
secret=techzone2090
cloud_id=-1
```

`cloud_id=-1` is suspicious for production use. Do not change it blindly; confirm which cloud this router belongs to before editing RadiusDesk data.

## Commands Applied

WireGuard interface, address, peer, and LAN membership:

```routeros
:if ([:len [/interface/wireguard/find name="wg1"]] = 0) do={
  /interface/wireguard/add name="wg1" mtu=1420 listen-port=13231 private-key="aN3k+ROvmMcrFmF8pV5+xe5fy4H7sMP4HUjnzzMUQWI="
} else={
  /interface/wireguard/set [find name="wg1"] mtu=1420 listen-port=13231 private-key="aN3k+ROvmMcrFmF8pV5+xe5fy4H7sMP4HUjnzzMUQWI="
}

:if ([:len [/ip/address/find where interface="wg1" and address="10.5.1.10/24"]] = 0) do={
  /ip/address/add address=10.5.1.10/24 interface=wg1 comment="RadiusDesk WireGuard"
}

:if ([:len [/interface/wireguard/peers/find where interface="wg1" and public-key="sRRYLe7zedWwvW7WEDdXYYa94E65XAr9Oi9DXvfAn3M="]] = 0) do={
  /interface/wireguard/peers/add interface=wg1 public-key="sRRYLe7zedWwvW7WEDdXYYa94E65XAr9Oi9DXvfAn3M=" endpoint-address=13.247.123.11 endpoint-port=51821 allowed-address=10.5.1.0/24 persistent-keepalive=25s comment="RadiusDesk wg1"
} else={
  /interface/wireguard/peers/set [find where interface="wg1" and public-key="sRRYLe7zedWwvW7WEDdXYYa94E65XAr9Oi9DXvfAn3M="] endpoint-address=13.247.123.11 endpoint-port=51821 allowed-address=10.5.1.0/24 persistent-keepalive=25s comment="RadiusDesk wg1"
}

:if ([:len [/interface/list/member/find where list="LAN" and interface="wg1"]] = 0) do={
  /interface/list/member/add list=LAN interface=wg1 comment="Allow RadiusDesk/API over WireGuard"
}
```

RADIUS alignment:

```routeros
/radius set [find where address=10.5.1.1] secret="techzone2090" src-address=10.5.1.10 timeout=1100ms require-message-auth=yes-for-request-resp
```

Device-mode activation was attempted:

```routeros
/system device-mode update hotspot=yes
```

RouterOS replied:

```text
update: turn off power or reboot by pressing reset or mode button in 5m to activate changes
```

The remote session was interrupted. Because this requires physical confirmation, `hotspot=yes` was not activated.

## Verification Results

Router WireGuard:

```routeros
/interface/wireguard
name=wg1 mtu=1420 listen-port=13231 running=yes
private-key="aN3k+ROvmMcrFmF8pV5+xe5fy4H7sMP4HUjnzzMUQWI="
public-key="4lHiQ2393+4lMUP7AhLD8siUgGycPam+YAKZZ0BeZiQ="

/interface/wireguard/peers
interface=wg1
public-key="sRRYLe7zedWwvW7WEDdXYYa94E65XAr9Oi9DXvfAn3M="
endpoint-address=13.247.123.11
endpoint-port=51821
allowed-address=10.5.1.0/24
persistent-keepalive=25s
last-handshake=9s
```

Router to server ping:

```routeros
/ping 10.5.1.1 src-address=10.5.1.10 count=4
sent=4 received=4 packet-loss=0% avg-rtt=55ms901us
```

Server to router ping:

```bash
ping -c 2 -W 2 10.5.1.10
2 packets transmitted, 2 received, 0% packet loss
```

MikroTik API port from server:

```bash
timeout 4 bash -lc 'cat < /dev/null > /dev/tcp/10.5.1.10/8728'
API_8728_OPEN
```

Server WireGuard handshake:

```text
peer: 4lHiQ2393+4lMUP7AhLD8siUgGycPam+YAKZZ0BeZiQ=
endpoint: 129.222.109.158:4138
allowed ips: 10.5.1.10/32
latest handshake: 11 seconds ago
```

RADIUS after correction:

```routeros
/radius
service=ppp,hotspot
address=10.5.1.1
secret="techzone2090"
authentication-port=1812
accounting-port=1813
src-address=10.5.1.10
require-message-auth=yes-for-request-resp
```

Remaining Hotspot blocker:

```routeros
/system/device-mode
hotspot=no
attempt-count=1

/ip/hotspot
hotspot1 invalid=yes comment="inactivated, not allowed by device-mode"
```

## Required Next Step

A person with physical access to the router must activate Hotspot capability in device-mode:

```routeros
/system device-mode update hotspot=yes
```

Then, within the 5-minute window, physically power-cycle the router or reboot using the reset/mode button as RouterOS requests. A normal remote `/system reboot` is not enough for this confirmation path.

After the router returns, verify:

```routeros
/system device-mode print
/ip hotspot print detail
/interface wireguard peers print detail
/ping 10.5.1.1 src-address=10.5.1.10 count=4
```

Expected:

```text
hotspot=yes
hotspot1 no longer invalid
wg1 peer latest-handshake is recent
ping to 10.5.1.1 succeeds
```

## Autonomous Runbook For Next Agent

1. SSH to router:

```bash
ssh -i "$ROUTER_SSH_KEY" "$ROUTER_SSH_USER@$ROUTER_LAN_IP"
```

2. Confirm whether WireGuard already exists:

```routeros
/interface wireguard print detail
/interface wireguard peers print detail
/ip address print detail where interface=wg1
```

3. If missing, apply the WireGuard commands from this document.

4. Confirm server peer exists:

```bash
ssh -i "$SERVER_SSH_KEY" "$SERVER_SSH_USER@$SERVER_HOST" 'sudo wg show wg1'
```

5. If server peer is missing, add a server peer for:

```text
public key: 4lHiQ2393+4lMUP7AhLD8siUgGycPam+YAKZZ0BeZiQ=
allowed ips: 10.5.1.10/32
```

6. Test MikroTik to server:

```routeros
/ping 10.5.1.1 src-address=10.5.1.10 count=4
```

7. Test server to MikroTik:

```bash
ping -c 4 -W 2 10.5.1.10
timeout 4 bash -lc 'cat < /dev/null > /dev/tcp/10.5.1.10/8728'
```

8. Verify RADIUS points through the tunnel:

```routeros
/radius print detail
```

Required values:

```routeros
address=10.5.1.1
secret="techzone2090"
src-address=10.5.1.10
service=ppp,hotspot
```

9. Check Hotspot device-mode:

```routeros
/system device-mode print
/ip hotspot print detail
```

If `hotspot=no`, stop and request physical confirmation. Do not waste time debugging Hotspot rules while device-mode blocks the feature.

10. After device-mode is fixed, perform a real captive portal login test using a RadiusDesk voucher or permanent user, then verify accounting in RadiusDesk.

## Problems To Not Ignore

- `device-mode hotspot=no` currently blocks the Hotspot. This is the main remaining blocker.
- The RadiusDesk NAS row for `10.5.1.10` has `cloud_id=-1`; confirm this is intentional before production login testing.
- The router keeps default `192.168.88.1/24` and the hotspot network `192.168.15.1/24` on the same `bridge`. That may be acceptable during staging, but a clean deployment should remove unused default addressing or separate management/client networks deliberately.
- MikroTik API `8728` is reachable from the server, but API authentication was not tested because only SSH key access was provided.
- Do not configure `AllowedIPs=0.0.0.0/0` on MikroTik unless full-tunnel routing is explicitly intended.
