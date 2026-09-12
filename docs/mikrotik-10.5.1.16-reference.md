# MikroTik 10.5.1.16 Reference

SECURITY SENSITIVE - DO NOT COMMIT OR SHARE.

Reusable baseline: use `docs/mikrotik-wireguard-radiusdesk-runbook.md` for new
routers. This file is a router-specific reference and contains live secrets.

This document intentionally includes live secrets because it is meant to be a complete reference for later configuration scripts. Treat it as credential material: WireGuard private key, RADIUS shared secret, and WiFi passphrase are present.

Collected on 2026-07-06 from:

```bash
ssh -i /home/mastershark-linux/ssh/mikrotik_admin admin-ssh@10.5.1.16
```

No configuration changes were applied during collection.

## 1. Identity And Platform

Router identity:

```routeros
/system identity
name: Ax2-Imandry-2
```

Platform:

| Field | Value |
| --- | --- |
| Model | hAP ax^2 / C52iG-5HaxD2HaxD |
| Board | hAP ax^2 |
| Architecture | arm64 |
| CPU | ARM64, 4 cores, 864 MHz |
| RouterOS | 7.19.6 stable |
| Build time | 2025-09-12 09:02:42 |
| Factory software | 7.11.2 |
| Current firmware | 7.19.6 |
| Upgrade firmware | 7.19.6 |
| Serial | HKF0AJPEDEE |
| Memory | 1024 MiB total, about 647 MiB free during collection |
| Storage | 128 MiB total, about 92 MiB free during collection |
| Bad blocks | 0% |
| Time zone | Indian/Antananarivo, GMT+03:00 |

## 2. Interfaces, Addressing, DNS, Routes

### Interfaces

| Interface | Type | State | Role / Notes |
| --- | --- | --- | --- |
| ether1 | ether | running | WAN uplink, DHCP client address `192.168.1.115/24` |
| ether2 | ether | running, slave | bridge port |
| ether3 | ether | slave, inactive | bridge port |
| ether4 | ether | running, slave | bridge port |
| ether5 | ether | slave, inactive | bridge port |
| bridge | bridge | running | main LAN / hotspot bridge |
| bridge1 | bridge | running | secondary bridge, DHCP server currently invalid because no active IP |
| wg1 | WireGuard | running | VPN to RadiusDesk/RADIUS side |
| wifi1 | wifi | bound | AP SSID `TECHZONE_WIFI_5G`, bridged to `bridge` |
| wifi2 | wifi | bound | AP SSID `TECHZONE_WIFI`, bridged to `bridge` |
| wifi3 | wifi virtual | bound | AP SSID `prise`, bridged to `bridge1` |
| <pppoe-Ozidine> | pppoe-in | dynamic running | active PPPoE session |
| <pppoe-Soalahy> | pppoe-in | dynamic running | active PPPoE session |

Bridge ports:

```routeros
/interface bridge port
bridge=bridge  interface=ether2
bridge=bridge  interface=ether3
bridge=bridge  interface=ether4
bridge=bridge  interface=ether5
bridge=bridge  interface=wifi1
bridge=bridge  interface=wifi2
bridge=bridge1 interface=wifi3
```

Interface lists:

```routeros
/interface/list/member
list=LAN interface=bridge
list=WAN interface=ether1
list=LAN interface=wg1
```

This matters: `wg1` is in `LAN`, so firewall rules that allow LAN management also allow management from the WireGuard side.

### WiFi

```routeros
/interface wifi
wifi1: mode=ap ssid="TECHZONE_WIFI_5G" band=5ghz-ax width=20/40/80mhz authentication-types=""
wifi2: mode=ap ssid="TECHZONE_WIFI" band=2ghz-ax width=20/40mhz authentication-types=""
wifi3: mode=ap ssid="prise" master-interface=wifi1 security=sec1
```

WiFi security profile:

```routeros
/interface wifi security
name=sec1 authentication-types=wpa2-psk passphrase="Prise-Tcz2k26"
```

The main SSIDs currently show empty `authentication-types`; this is consistent with an open hotspot model where the captive portal/RADIUS path enforces access.

### IP Addresses

```routeros
/ip/address
192.168.10.1/24 interface=bridge   network=192.168.10.0 comment=defconf
10.5.1.16/24    interface=wg1      network=10.5.1.0
10.10.10.1/24   interface=bridge1  network=10.10.10.0 disabled=yes
192.168.1.115/24 interface=ether1  network=192.168.1.0 dynamic=yes
172.16.16.1/32  interface=<pppoe-Ozidine> network=172.16.16.2 dynamic=yes
172.16.16.1/32  interface=<pppoe-Soalahy> network=172.16.16.3 dynamic=yes
```

### Routes

```routeros
/ip/route
0.0.0.0/0       gateway=192.168.1.1%ether1 dynamic active
10.5.1.0/24     gateway=wg1 connected
172.16.16.2/32  gateway=<pppoe-Ozidine> connected
172.16.16.3/32  gateway=<pppoe-Soalahy> connected
192.168.1.0/24  gateway=ether1 connected
192.168.10.0/24 gateway=bridge connected
```

### DNS

```routeros
/ip/dns
servers=8.8.8.8,1.1.1.1
dynamic-servers=192.168.1.1
allow-remote-requests=yes
cache-size=2048KiB
cache-used=364KiB
```

DNS remote requests are enabled, which is required for the hotspot DNS interception model but should remain protected by firewall and interface scoping.

### Management Services

```routeros
/ip/service
ftp      port=21   enabled address=any
ssh      port=22   enabled address=any
telnet   port=23   enabled address=any
www      port=80   enabled address=any
www-ssl  port=443  disabled
winbox   port=8291 enabled address=any
api      port=8728 enabled address=any
api-ssl  port=8729 enabled address=any
```

This is too open for a production template. For the future script, restrict admin services to the WireGuard/RADIUS admin subnet or disable unused services, especially Telnet, FTP, and plain API.

## 3. Hotspot

### Server

```routeros
/ip/hotspot
name=hotspot1
interface=bridge
address-pool=default-dhcp
profile=hsprof1
idle-timeout=5m
keepalive-timeout=none
login-timeout=none
addresses-per-mac=2
ip-of-dns-name=192.168.10.1
proxy-status=running
```

### Profiles

Default profile exists but does not use RADIUS:

```routeros
name=default hotspot-address=0.0.0.0 html-directory=hotspot login-by=cookie,http-chap use-radius=no
```

Active hotspot profile:

```routeros
/ip/hotspot/profile
name=hsprof1
hotspot-address=192.168.10.1
dns-name=login.techzone.lat
html-directory=hotspot
login-by=cookie,https,http-pap,mac-cookie
http-cookie-lifetime=3d
ssl-certificate=cert.p12_0
use-radius=yes
radius-accounting=yes
radius-interim-update=1m
nas-port-type=wireless-802.11
radius-mac-format=XX:XX:XX:XX:XX:XX
```

This is the core RadiusDesk integration point: Hotspot authentication and accounting are delegated to RADIUS.

### User Profiles And Local Users

```routeros
/ip/hotspot/user/profile
default: idle-timeout=none keepalive-timeout=2m shared-users=1 add-mac-cookie=yes mac-cookie-timeout=3d
```

Local hotspot users are minimal:

```routeros
/ip/hotspot/user
default-trial dynamic/default counters only
admin profile=default
```

Most live users are RADIUS-backed, not local.

### Active Hotspot Sessions

At collection time there were 12 active RADIUS hotspot sessions. Examples:

```routeros
user=sharen         address=192.168.10.10 login-by=cookie     limit-bytes-total=576577503641
user=danger-rare    address=192.168.10.13 login-by=mac-cookie session-time-left=3w3d14h26m30s
user=loreno         address=192.168.10.14 login-by=mac-cookie session-time-left=3w4d5h42m47s
user=chaleur-muet   address=192.168.10.15 login-by=https      session-time-left=6d2h21m29s
user=tiana2         address=192.168.10.16 login-by=mac-cookie session-time-left=3w4d5h42m47s
user=patrice        address=192.168.10.18 login-by=mac-cookie limit-bytes-total=532189994498
user=fitahiantsoa   address=192.168.10.19 login-by=mac-cookie
user=parfait        address=192.168.10.20 login-by=mac-cookie limit-bytes-total=550267169827
user=harinarivo     address=192.168.10.21 login-by=mac-cookie limit-bytes-total=508190105341
user=mamonjisoa     address=192.168.10.23 login-by=https
user=hasinaii       address=192.168.10.24 login-by=mac-cookie limit-bytes-total=543246247927
user=douleur-grave  address=192.168.10.66 login-by=mac-cookie limit-bytes-total=12647285257
```

Hotspot host count during collection: `16`.
Hotspot cookie count during collection: `20`.

### Walled Garden

```routeros
/ip/hotspot/walled-garden
dst-address=13.247.123.11 action=allow comment="hotspot.techzone.lat" dynamic/inactivated
action=allow comment="place hotspot rules here" disabled/inactivated
```

The `inactivated, not allowed by device-mode` state is a problem for a production baseline. If the captive portal must allow RadiusDesk or landing-page resources before authentication, the future script should validate device-mode compatibility and generate working walled-garden entries.

### IP Bindings

```routeros
/ip/hotspot/ip-binding
disabled bypassed mac=9C:E5:49:65:37:14 comment="AP-IMD-2"
disabled bypassed mac=B8:3A:08:3C:38:30 server=hotspot1
bypassed mac=9C:E5:49:64:66:C8 address=192.168.10.36 to-address=192.168.10.36 server=hotspot1 comment="AP imandry 2"
bypassed mac=DC:71:96:1B:F4:B3 address=192.168.10.42 to-address=192.168.10.42 server=hotspot1 comment="PC-Hasina"
```

These bypasses should be kept explicit in generated scripts because they affect whether APs/admin machines are forced through the captive portal.

## 4. RADIUS

RADIUS client config on MikroTik:

```routeros
/radius
service=ppp,hotspot
address=10.5.1.1
secret="techzone2090"
authentication-port=1812
accounting-port=1813
timeout=1s100ms
radsec-timeout=3s300ms
accounting-backup=no
src-address=10.5.1.16
protocol=udp
certificate=none
require-message-auth=yes-for-request-resp
```

Incoming RADIUS:

```routeros
/radius/incoming
accept=no
port=3799
vrf=main
```

Important implications for RadiusDesk:

- RadiusDesk/FreeRADIUS should define the NAS as `10.5.1.16`.
- The NAS shared secret must match `techzone2090`.
- The RADIUS server must be reachable through WireGuard at `10.5.1.1`.
- MikroTik sends requests from `src-address=10.5.1.16`, which is the correct tunnel-side NAS IP.
- Disconnect-Request / CoA is not currently enabled because `/radius incoming accept=no`. If RadiusDesk must disconnect users live, enable and firewall UDP `3799` only from the RADIUS server.

## 5. PPP And PPPoE

### PPP AAA

```routeros
/ppp/aaa
use-radius=yes
accounting=yes
interim-update=1m
use-circuit-id-in-nas-port-id=no
enable-ipv6-accounting=no
```

### PPP Profiles

```routeros
/ppp/profile
name=default
local-address=172.16.16.1
remote-address=pool-pppoe
use-ipv6=yes
change-tcp-mss=yes
dns-server=8.8.8.8
wins-server=1.1.1.1

name=default-encryption
use-encryption=yes
use-ipv6=yes
change-tcp-mss=yes
```

There are no local PPP secrets in the collected config; PPP/PPPoE authentication is RADIUS-backed.

### PPPoE Server

```routeros
/interface/pppoe-server/server
service-name=service1
interface=bridge
max-mtu=auto
max-mru=auto
mrru=disabled
authentication=pap,chap,mschap1,mschap2
keepalive-timeout=10
one-session-per-host=no
max-sessions=unlimited
pado-delay=0
default-profile=default
```

Active dynamic PPPoE sessions:

```routeros
name=<pppoe-Ozidine> user=Ozidine service=service1 mtu=1480 mru=1492 remote-address=9A:E5:49:65:3A:AE interface=bridge
name=<pppoe-Soalahy> user=Soalahy service=service1 mtu=1480 mru=1492 remote-address=9A:E5:49:65:39:02 interface=bridge
```

Pool:

```routeros
/ip/pool
pool-pppoe ranges=172.16.16.2-172.16.16.254
```

## 6. DHCP Server

Pools:

```routeros
/ip/pool
default-dhcp ranges=192.168.10.10-192.168.10.254
pool-pppoe    ranges=172.16.16.2-172.16.16.254
pool-prise    ranges=10.10.10.2-10.10.10.254
```

DHCP servers:

```routeros
/ip/dhcp-server
name=defconf interface=bridge lease-time=30m address-pool=default-dhcp use-radius=no
name=prise   interface=bridge1 lease-time=30m address-pool=pool-prise use-radius=no invalid=yes comment="No IP address on interface"
```

DHCP network:

```routeros
/ip/dhcp-server/network
address=192.168.10.0/24 gateway=192.168.10.1 dns-server=192.168.10.1 comment=defconf
```

The `prise` DHCP server is invalid because `10.10.10.1/24` on `bridge1` is disabled. A future script must either enable that address or remove/skip the `prise` network.

Observed DHCP leases included APs and clients on `192.168.10.0/24`, for example:

```routeros
192.168.10.36  9C:E5:49:64:66:C8 host=HQ835 static/bound
192.168.10.25  E8:48:B8:EF:E2:B5 host=TL-WR940N dynamic/bound
192.168.10.57  44:F9:71:20:01:0F host=TL-WR820N dynamic/bound
192.168.10.225 F4:8C:EB:B0:29:08 host=dlinkap dynamic/bound
192.168.10.73  00:D8:61:85:FA:C7 host=Jess-Iris dynamic/bound
```

## 7. Firewall

### Filter Rules

Dynamic hotspot rules are installed before the default firewall rules:

```routeros
chain=forward action=jump jump-target=hs-unauth hotspot=from-client,!auth dynamic
chain=forward action=jump jump-target=hs-unauth-to hotspot=to-client,!auth dynamic
chain=input   action=jump jump-target=hs-input hotspot=from-client dynamic
chain=input   action=drop protocol=tcp hotspot=!from-client dst-port=64872-64875 dynamic
chain=hs-input action=accept protocol=udp dst-port=64872 dynamic
chain=hs-input action=accept protocol=tcp dst-port=64872-64875 dynamic
chain=hs-unauth action=return dst-address=13.247.123.11 comment="hotspot.techzone.lat" dynamic
chain=hs-unauth-to action=return src-address=13.247.123.11 comment="hotspot.techzone.lat" dynamic
chain=hs-unauth action=reject reject-with=tcp-reset protocol=tcp dynamic
chain=hs-unauth action=reject reject-with=icmp-net-prohibited dynamic
chain=hs-unauth-to action=reject reject-with=icmp-host-prohibited dynamic
```

Static/default rules:

```routeros
input   accept established,related,untracked
input   drop   invalid
input   accept icmp
input   accept dst-address=127.0.0.1 comment="for CAPsMAN"
input   drop   in-interface-list=!LAN
forward accept ipsec-policy=in,ipsec
forward accept ipsec-policy=out,ipsec
forward fasttrack established,related disabled=yes
forward accept established,related,untracked
forward drop   invalid
forward drop   connection-state=new connection-nat-state=!dstnat in-interface-list=WAN
```

The input policy allows anything from interfaces in `LAN`, and `wg1` is a LAN member. That is useful for management over WireGuard, but scripts should restrict service exposure to expected admin subnets rather than relying only on interface membership.

### NAT

Dynamic hotspot NAT:

```routeros
dstnat jump hotspot hotspot=from-client dynamic
hotspot redirect DNS UDP/TCP 53 to 64872 dynamic
hotspot redirect unauth HTTP/HTTPS/proxy ports to 64874/64875 dynamic
hotspot return dst-address=13.247.123.11 comment="hotspot.techzone.lat" dynamic
```

Static NAT:

```routeros
srcnat masquerade out-interface-list=WAN ipsec-policy=out,none comment="defconf: masquerade"
srcnat masquerade src-address=192.168.10.0/24 comment="masquerade hotspot network"
srcnat masquerade src-address=192.168.10.0/24 comment="masquerade hotspot network"
```

There are two duplicate hotspot masquerade rules for `192.168.10.0/24`. A future baseline should create only one.

### Mangle And Raw

The router has Buananet-style traffic classification:

```routeros
mangle marks Speedtest/Youtube/Tiktok/Netflix/Facebook connections and packets
raw adds destinations to Mikrotik-Youtube, Mikrotik-Tiktok, Mikrotik-Netflix, Mikrotik-Facebook using TLS host matches
```

Raw TLS host matches include:

```routeros
*youtube.com*, *.youtube.*, *.googlevideo.*, *.youtu.*, *.ytimg.*, *yt3.ggpht.com*, *youtubei.googleapis.com*
*tiktok.com*, *.tiktok.*, *.tiktoktv.*, *.e.tiktok.*, *.tiktokcdn.*, *.musical.*
*netflix.com*, *.netflix.*, *.nflxvideo.*, *.nflxext.*
*facebook.com*, *.facebook.*, *.m.me*, *.fb.me*, *.msngr.*, *.messenger.*, *.fbcdn.*, *.fbsbx.*
```

Address-list scale at collection time:

```text
total address-list entries: 9011
static entries: 8659
dynamic entries: 352
```

Do not blindly copy all address-list entries into a clean Hotspot/RADIUS script. They are not part of the minimum RadiusDesk integration and will make the generated script brittle and huge.

### Firewall Service Ports

```routeros
ftp enabled port=21
tftp enabled port=69
irc disabled port=6667
h323 enabled
sip enabled ports=5060,5061 sip-direct-media=yes sip-timeout=1h
pptp enabled
rtsp disabled port=554
udplite enabled
dccp enabled
sctp enabled
```

## 8. WireGuard

Interface:

```routeros
/interface/wireguard
name=wg1
mtu=1420
listen-port=13231
private-key="4Mm2+xzb+8xPv0tHsIgNduiY3sThXGUNCSFDD1zISEA="
public-key="VcsrpBqBkjJHjqeLjN/xPLLiOnc6qLrBAKb4Og6GmyY="
```

Peer:

```routeros
/interface/wireguard/peers
interface=wg1
name=peer1
public-key="sRRYLe7zedWwvW7WEDdXYYa94E65XAr9Oi9DXvfAn3M="
endpoint-address=13.247.123.11
endpoint-port=51821
current-endpoint-address=13.247.123.11
current-endpoint-port=51821
allowed-address=10.5.1.0/24,172.16.0.0/16
persistent-keepalive=25s
rx=467.9KiB
tx=3919.8KiB
last-handshake=50s
```

WireGuard IP:

```routeros
/ip/address
10.5.1.16/24 interface=wg1 network=10.5.1.0
```

This is already the correct structure for using WireGuard as the RADIUS path:

```text
MikroTik NAS 10.5.1.16  <---- WireGuard 10.5.1.0/24 ---->  RADIUS/RadiusDesk 10.5.1.1
```

The peer allowed-address also includes `172.16.0.0/16`, which covers PPPoE client addressing.

## 9. RadiusDesk Integration Model

Target flow:

```text
client device
  -> WiFi/open LAN on bridge
  -> MikroTik hotspot1 captive portal
  -> MikroTik RADIUS request from 10.5.1.16
  -> WireGuard wg1
  -> FreeRADIUS/RadiusDesk at 10.5.1.1
  -> Access-Accept with session/data/time attributes
  -> MikroTik creates authenticated hotspot session
  -> Accounting-Start / Interim-Update every 1m / Accounting-Stop
```

Minimum RadiusDesk/FreeRADIUS NAS definition:

```text
NAS name / shortname: Ax2-Imandry-2 or imandry-2
NAS IP address: 10.5.1.16
Shared secret: techzone2090
Type: MikroTik
Ports: auth UDP 1812, acct UDP 1813
Optional CoA/Disconnect: UDP 3799 only if /radius incoming accept=yes
```

MikroTik attributes already aligned with RadiusDesk:

```routeros
/radius service=ppp,hotspot address=10.5.1.1 src-address=10.5.1.16
/ip/hotspot/profile hsprof1 use-radius=yes radius-accounting=yes radius-interim-update=1m
/ppp/aaa use-radius=yes accounting=yes interim-update=1m
```

The dynamic login page should work when:

- `login.techzone.lat` resolves to or reaches the captive portal path expected by the hotspot.
- RadiusDesk has a NAS entry for `10.5.1.16` with secret `techzone2090`.
- FreeRADIUS listens on `10.5.1.1:1812/1813`.
- Firewall on the RadiusDesk host allows UDP `1812/1813` from `10.5.1.16`.
- The WireGuard server routes `10.5.1.0/24` and knows the MikroTik peer public key.

## 10. Future Script Baseline

A clean script generated from this reference should separate concerns:

1. Base identity and admin hardening.
2. Bridge/LAN/WAN/WiFi layout.
3. WireGuard client-side tunnel to RadiusDesk.
4. IP addressing, routes, DNS.
5. DHCP for hotspot LAN.
6. Hotspot server/profile with RADIUS authentication and accounting.
7. RADIUS client pointing to `10.5.1.1` from `10.5.1.16`.
8. PPPoE server and PPP AAA if PPPoE remains in scope.
9. Minimal firewall required for WAN NAT, hotspot dynamic rules, WireGuard/RADIUS reachability, and management from trusted subnets.
10. Optional traffic classification lists, but not as part of the minimum RadiusDesk integration.

Do not carry these current weaknesses into the generated production script without an explicit reason:

- Plain Telnet, FTP, WWW, and API exposed to `address=any`.
- Duplicate hotspot masquerade rule.
- Invalid `prise` DHCP server because `bridge1` has no active IP.
- Walled garden entries marked inactivated by device-mode.
- Huge static address-list imports unrelated to the RadiusDesk path.
- RADIUS incoming disabled if live disconnect/CoA is required.

## 11. Useful Read-Only Audit Commands

Use these commands to refresh the inventory before generating scripts:

```routeros
/system identity print
/system resource print
/system routerboard print
/interface print detail
/interface bridge print detail
/interface bridge port print detail
/interface wifi print detail
/interface list print detail
/interface list member print detail
/ip address print detail
/ip route print detail
/ip dns print
/ip hotspot print detail
/ip hotspot profile print detail
/ip hotspot user profile print detail
/ip hotspot active print detail
/ip hotspot host print count-only
/ip hotspot cookie print count-only
/ip hotspot walled-garden print detail
/ip hotspot ip-binding print detail
/radius print detail
/radius incoming print
/ppp aaa print
/ppp profile print detail
/ppp secret print detail
/interface pppoe-server server print detail
/interface pppoe-server print detail
/ip pool print detail
/ip dhcp-server print detail
/ip dhcp-server network print detail
/ip dhcp-server lease print detail without-paging
/ip firewall filter print detail without-paging
/ip firewall nat print detail without-paging
/ip firewall mangle print detail
/ip firewall raw print detail
/ip firewall address-list print count-only
/interface wireguard print detail
/interface wireguard peers print detail
/ip service print detail
```
