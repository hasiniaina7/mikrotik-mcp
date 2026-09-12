# MikroTik User Internet Optimization Audit

Audit date: 2026-07-07  
Router audited: `imandry-1-remplacement`  
Router management: `192.168.1.47`, WireGuard NAS IP `10.5.1.14`  
Scope: optimize user Internet experience for Hotspot and PPPoE while keeping RadiusDesk, WireGuard, firewall, and management security intact.

## Executive Finding

The current router can serve both Hotspot and PPPoE users, but it is not ready for aggressive optimization until the security baseline is tightened. The biggest mistake would be to add burst profiles while leaving management services exposed and WAN classification inconsistent.

Current state:

- Hotspot is enabled on `bridge`, subnet `192.168.14.0/24`, profile `hsprof1`, RADIUS enabled.
- PPPoE server is enabled on `bridge`, pool `172.16.14.10-172.16.14.250`, RADIUS enabled.
- RadiusDesk tunnel is healthy through WireGuard `wg1`, NAS source `10.5.1.14`.
- Dynamic queues are created for Hotspot and PPPoE users.
- No active burst is configured; dynamic queues show `burst-limit=0/0`, `burst-threshold=0/0`, `burst-time=0s/0s`.
- A PPPoE NAT gap was already corrected with `src-address=172.16.14.0/24 out-interface=bridge1`.

Decision: optimize with controlled per-user burst and parent capacity shaping, but only after fixing WAN interface lists, input firewall, service exposure, RadiusDesk ownership, and CoA restrictions.

## Current Observations

### Access Models

Hotspot:

- Interface: `bridge`
- Gateway: `192.168.14.1/24`
- Pool: `hs-pool-9`, `192.168.14.2-192.168.14.254`
- Profile: `hsprof1`
- RADIUS: `use-radius=yes`, accounting enabled, interim update `1m`
- Login methods: `cookie,http-chap,https,mac-cookie`
- Current dynamic queues include examples like `4M/12M`, `6M/15M`, `10M/15M`.

PPPoE:

- Interface: `bridge`
- Pool: `172.16.14.10-172.16.14.250`
- Default profile: `default`
- RADIUS: enabled through `/ppp aaa`
- Current dynamic queue example: `<pppoe-Sophie>` with `limit-at=4M/12M`, `max-limit=4M/12M`, no burst.
- PPPoE server allows `pap,chap,mschap1,mschap2`.

Uplink:

- Default route exits via gateway `192.168.1.1%bridge1`.
- `ether1` is a member of `bridge1`.
- Interface list `WAN` currently contains `ether1`, not `bridge1`.

This matters. If firewall and NAT rules match `out-interface-list=WAN`, but real traffic exits via `bridge1`, policy may not hit where expected.

### RadiusDesk

Read-only database check:

- `nasname=10.5.1.14`, `type=Mikrotik-API`, `cloud_id=-1`
- Reference router `10.5.1.16` has `cloud_id=24`
- User `Sophie` exists as a permanent user with `cloud_id=24`, but her profile ID `73` has `cloud_id=-1`

This is weak ownership hygiene. It may work operationally, but it is not clean enough for scaled production policy management.

## Critical Risks

### 1. WAN Classification Is Wrong

The router uses `bridge1` as the effective uplink interface, but `WAN` contains `ether1`. This already caused a PPPoE NAT failure pattern.

Required correction:

```routeros
/interface/list/member/add list=WAN interface=bridge1 comment="WAN uplink bridge"
```

After validation, decide whether `ether1` should remain in `WAN`. If `ether1` stays enslaved under `bridge1`, policy should normally target `bridge1`.

Do not enable stricter WAN drop rules until this is validated, otherwise you may lock out legitimate uplink behavior.

### 2. Input Firewall Is Too Permissive

The default input drop rule is disabled:

```text
defconf: drop all not coming from LAN = disabled
```

That is not acceptable if Hotspot and PPPoE are active. Users should authenticate and get Internet, not gain router management surface.

Minimum target:

- Allow established/related.
- Drop invalid.
- Allow ICMP with reasonable limits.
- Allow DHCP/DNS/Hotspot services from client networks where required.
- Allow SSH/Winbox/API only from trusted admin subnets, preferably WireGuard admin IPs.
- Allow RadiusDesk CoA only from `10.5.1.1` to UDP `3799`.
- Drop all other input from WAN, Hotspot users, and PPPoE users.

### 3. Management Services Are Exposed Too Broadly

Current `/ip service` has Telnet, FTP, WWW, Winbox, API, API-SSL, and SSH present without address restrictions shown.

Recommended baseline:

```routeros
/ip/service/set telnet disabled=yes
/ip/service/set ftp disabled=yes
/ip/service/set www disabled=yes
/ip/service/set www-ssl disabled=yes
/ip/service/set api address=10.5.1.0/24
/ip/service/set api-ssl disabled=yes
/ip/service/set ssh address=10.5.1.0/24,192.168.88.0/24
/ip/service/set winbox address=10.5.1.0/24,192.168.88.0/24
```

Do not expose plain API or Winbox to client subnets. RadiusDesk can reach the router through WireGuard; use that path.

### 4. Radius Incoming Is Enabled Without a Visible Restriction

`/radius incoming accept=yes port=3799` is useful for Disconnect-Request / CoA. It must be firewall-restricted to RadiusDesk only.

Required input rule:

```routeros
/ip/firewall/filter/add chain=input action=accept protocol=udp src-address=10.5.1.1 dst-port=3799 comment="Allow RadiusDesk CoA only"
```

Then make sure a later rule drops unsolicited input.

### 5. Walled Garden Must Not Be Trusted Yet

The router showed walled-garden entries marked:

```text
inactivated, not allowed by device-mode
```

Even though device mode reports `hotspot=yes`, this output means the walled-garden behavior must be validated with a real unauthenticated client before relying on it.

Do not launch a production captive portal that depends on walled-garden destinations until this is proven from a phone/laptop on the Hotspot network.

### 6. IPv6 Is Enabled in PPP Profiles Without an IPv6 Policy

The PPP profile shows `use-ipv6=yes`. If IPv6 is not intentionally deployed with firewalling, prefix policy, and accounting, this is unnecessary risk.

Recommendation:

```routeros
/ppp/profile/set default use-ipv6=no
```

Only keep IPv6 enabled if there is a complete IPv6 plan.

## Optimization Strategy

### Principle

Burst should improve perceived speed for short actions, not let heavy users consume the uplink indefinitely.

A good burst design has three layers:

1. Per-user sustained limit: the speed sold to the user.
2. Per-user burst: short extra speed for page loads, app updates, DNS-heavy mobile usage, and messaging.
3. Parent uplink cap: total shaping below real ISP capacity to avoid bufferbloat and uncontrolled queueing upstream.

Without the parent uplink cap, per-user burst can make the link feel worse during busy hours.

### Measure the Real Uplink First

Before setting parent queues, measure the actual WAN capacity at quiet hours and busy hours.

Use conservative production values:

- Parent download cap: 85-90% of stable measured download.
- Parent upload cap: 80-90% of stable measured upload.
- If the ISP link varies a lot, use the busy-hour stable value, not the best speedtest result.

Example:

```text
Measured stable WAN: 100M down / 30M up
Parent cap:          90M down / 25M up
```

### Recommended Profiles

These are starting profiles. They must be adjusted after observing real congestion.

| Profile | Sustained | Burst | Threshold | Time | Use case |
| --- | ---: | ---: | ---: | ---: | --- |
| Basic Hotspot | 4M up / 12M down | 8M up / 24M down | 3M up / 8M down | 15s | phones, short sessions |
| Standard PPPoE | 4M up / 12M down | 8M up / 24M down | 3M up / 8M down | 20s | home users |
| Plus PPPoE | 8M up / 20M down | 12M up / 35M down | 6M up / 14M down | 20s | heavier homes |
| Business | fixed plan | no or small burst | 80% of plan | 10s | predictable service |

Hard rule: do not set burst above what the parent uplink can absorb when several users trigger it together.

Bad example:

```text
50 users x 50M burst on a 100M uplink
```

That is fake performance. It creates congestion and support tickets.

### RadiusDesk / RADIUS Rate Policy

Use RadiusDesk profiles as the source of truth. The router should receive dynamic queues from RADIUS rather than maintain hand-written per-user queues.

For MikroTik, the RADIUS attribute normally used is:

```text
Mikrotik-Rate-Limit
```

The value can include sustained rate, burst rate, burst threshold, burst time, priority, and minimum rate. Validate direction on a test user before changing production profiles because RadiusDesk and RouterOS display upload/download from different perspectives.

Example intent for a `4M up / 12M down` plan with burst:

```text
sustained:       4M up / 12M down
burst-limit:     8M up / 24M down
burst-threshold: 3M up / 8M down
burst-time:      20s
```

Rollout sequence:

1. Create one test RadiusDesk profile.
2. Assign it to one test Hotspot user and one test PPPoE user.
3. Authenticate both users.
4. Confirm the dynamic simple queue has the expected `max-limit`, `burst-limit`, `burst-threshold`, and `burst-time`.
5. Run short and sustained download/upload tests.
6. Only then migrate real profiles.

### Parent Queue Design

The current dynamic queues are created with parent `none`. That works, but it does not protect the uplink from aggregate overload.

Target model:

- One parent for WAN upload.
- One parent for WAN download, if packet marking is implemented correctly.
- Per-user dynamic queues under controlled parent queues when possible.
- PCQ queue types for fairness if many users share a package.

If using only dynamic simple queues from RADIUS, keep the design simple first:

- Add burst to RADIUS profiles.
- Set conservative max rates.
- Monitor CPU and latency.
- Add queue tree/PCQ only when aggregate contention is proven.

Do not introduce mangle-heavy queue trees before the basic RADIUS queue behavior is stable.

## Hotspot and PPPoE Coexistence

The current design puts both Hotspot clients and PPPoE discovery on `bridge`. This can work, but it is a mixed access domain.

Safer target:

- Hotspot SSID/VLAN: Hotspot only.
- PPPoE access VLAN or Ethernet segment: PPPoE only.
- Management VLAN/WireGuard: admin only.
- WAN bridge/uplink: not mixed with client access.

If VLANs are not introduced yet, enforce these controls:

- Keep Hotspot authentication on `192.168.14.0/24`.
- Keep PPPoE addressing on `172.16.14.0/24`.
- Block client-to-router management except Hotspot/DNS/DHCP/PPPoE control.
- Block client-to-client traffic if users should not see each other.
- Avoid broad `bypassed` Hotspot bindings except for infrastructure devices.

Current bypass:

```text
AP MAC 7C:27:3C:72:F4:E0 bypassed as 192.168.14.249
```

That is acceptable only if it is truly infrastructure. Bypass entries for ordinary clients destroy the captive portal model.

## Security Baseline Before Optimization

Apply in this order, testing after each group.

### Phase 1: Correct Interface Classification

```routeros
/interface/list/member/add list=WAN interface=bridge1 comment="WAN uplink bridge"
```

Validation:

```routeros
/ip/route/print detail where dst-address=0.0.0.0/0
/tool/ping 8.8.8.8 count=4
/tool/ping 8.8.8.8 src-address=172.16.14.1 count=4
```

### Phase 2: Restrict Services

```routeros
/ip/service/set telnet disabled=yes
/ip/service/set ftp disabled=yes
/ip/service/set www disabled=yes
/ip/service/set www-ssl disabled=yes
/ip/service/set api address=10.5.1.0/24
/ip/service/set api-ssl disabled=yes
/ip/service/set ssh address=10.5.1.0/24,192.168.88.0/24
/ip/service/set winbox address=10.5.1.0/24,192.168.88.0/24
```

Validation:

- SSH still works over WireGuard.
- RadiusDesk API still reaches `10.5.1.14:8728`.
- Winbox works only from trusted admin paths.

### Phase 3: Harden Input Firewall

Use a staged approach. Do not paste a drop rule first.

Required allows:

```routeros
/ip/firewall/filter/add chain=input action=accept connection-state=established,related,untracked comment="accept established"
/ip/firewall/filter/add chain=input action=drop connection-state=invalid comment="drop invalid"
/ip/firewall/filter/add chain=input action=accept protocol=icmp limit=10,20:packet comment="limited ICMP"
/ip/firewall/filter/add chain=input action=accept in-interface=wg1 src-address=10.5.1.0/24 comment="admin/RADIUS side over WireGuard"
/ip/firewall/filter/add chain=input action=accept protocol=udp src-address=10.5.1.1 dst-port=3799 comment="RadiusDesk CoA only"
```

Client service allows, if needed:

```routeros
/ip/firewall/filter/add chain=input action=accept in-interface=bridge protocol=udp dst-port=67,68 comment="DHCP for client bridge"
/ip/firewall/filter/add chain=input action=accept in-interface=bridge protocol=udp dst-port=53 comment="DNS UDP for clients"
/ip/firewall/filter/add chain=input action=accept in-interface=bridge protocol=tcp dst-port=53 comment="DNS TCP for clients"
```

Then add drops after confirming required services:

```routeros
/ip/firewall/filter/add chain=input action=drop in-interface-list=WAN comment="drop input from WAN"
/ip/firewall/filter/add chain=input action=drop src-address=192.168.14.0/24 comment="drop Hotspot users to router"
/ip/firewall/filter/add chain=input action=drop src-address=172.16.14.0/24 comment="drop PPPoE users to router"
```

This must be tested carefully because Hotspot installs dynamic rules. Do not break captive portal DNS/redirect behavior.

### Phase 4: Validate RadiusDesk Ownership

Fix or explicitly accept these before mass rollout:

- NAS `10.5.1.14` has `cloud_id=-1`.
- Profile `010-illimite-49000-Ar-3-appareils` has `cloud_id=-1` while assigned user Sophie is in `cloud_id=24`.

This is not cosmetic. Cloud ownership affects reporting, API scoping, and future automation.

## Burst Rollout Plan

### Test Profile

Create a test profile in RadiusDesk:

```text
Name: TEST-12M-burst-24M
Sustained: 4M up / 12M down
Burst: 8M up / 24M down
Threshold: 3M up / 8M down
Time: 20s
```

Expected MikroTik dynamic queue after login:

```text
max-limit=4M/12M
burst-limit=8M/24M
burst-threshold=3M/8M
burst-time=20s/20s
```

If the direction appears reversed, stop and correct the RadiusDesk attribute mapping before touching real users.

### Validation Tests

For one Hotspot test user and one PPPoE test user:

```routeros
/queue/simple/print detail stats where name~"test|TEST|pppoe|hotspot"
/ip/firewall/connection/print count-only where src-address~"192.168.14|172.16.14"
/tool/profile
```

From the client:

- Open several websites: should feel fast.
- Run a 60-second download: should fall back near sustained rate after burst window.
- Run upload test: confirm upload does not saturate the WAN.
- Start multiple clients: latency should not collapse.

Success criteria:

- Short actions benefit from burst.
- Long downloads settle near plan rate.
- Router CPU stays below sustained high load.
- Ping latency under load remains acceptable.
- RADIUS accounting continues updating every minute.

## Monitoring

Add routine checks:

```routeros
/queue/simple/print stats
/interface/monitor-traffic bridge1 once
/interface/monitor-traffic bridge once
/ppp/active/print count-only
/ip/hotspot/active/print count-only
/radius/monitor 0 once
/tool/profile
```

RadiusDesk checks:

```sql
SELECT COUNT(*) FROM radacct WHERE acctstoptime IS NULL;
SELECT username,nasipaddress,framedipaddress,acctinputoctets,acctoutputoctets,acctsessiontime
FROM radacct
WHERE acctstoptime IS NULL
ORDER BY acctstarttime DESC
LIMIT 20;
```

Track:

- Active users by access type.
- Top talkers.
- Queue drops.
- RADIUS timeouts/rejects.
- WAN saturation periods.
- CPU during peak traffic.

## Recommended Final Architecture

Target architecture for stable production:

```text
WAN / ISP
  -> bridge1 or direct WAN
  -> NAT and parent WAN shaping

Client access bridge/VLANs
  -> Hotspot SSID/VLAN: 192.168.14.0/24
  -> PPPoE segment/VLAN: 172.16.14.0/24
  -> no router management access

WireGuard wg1
  -> RadiusDesk RADIUS/API/CoA only
  -> admin management only

RadiusDesk
  -> owns profiles, user limits, burst, quotas, accounting
```

## Priority Checklist

1. Fix WAN list to include effective uplink `bridge1`.
2. Restrict `/ip service` to WireGuard/admin subnets.
3. Restrict UDP `3799` CoA to `10.5.1.1`.
4. Re-enable input drop policy carefully after captive portal validation.
5. Resolve RadiusDesk `cloud_id=-1` on NAS and active profiles.
6. Validate walled-garden behavior from an unauthenticated client.
7. Create one RadiusDesk burst test profile.
8. Test Hotspot and PPPoE dynamic queues with burst.
9. Add parent shaping only after measuring real WAN capacity.
10. Roll out by package, not by editing individual users manually.

## Bottom Line

Burst is useful here, but it is not the first fix. The router currently needs security hardening and RadiusDesk ownership cleanup before scaled optimization.

The correct path is:

```text
secure management plane
-> correct WAN/NAT/interface policy
-> validate Hotspot and PPPoE auth/accounting
-> introduce controlled RadiusDesk burst profiles
-> monitor aggregate load and add parent shaping if needed
```

Anything else is just making speedtest results look better while leaving the network fragile.
