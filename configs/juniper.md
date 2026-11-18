# Juniper vSRX core

Sanitized Junos configuration for the two `vSRX` routers in the dual-ISP lab. `JUN-CORE1` terminates one link from each ISP ASBR pair (`C1-ISP1`, `C2-ISP1`), `JUN-CORE2` does the same on the ISP-2 side (`C1-ISP2`, `C2-ISP2`). vSRX is used as the Juniper core because it runs the same Junos routing stack as the branch SRX line, so OSPF and route selection behave as they would on hardware.

Both devices are switched to packet-based mode. In flow mode every packet would cross the SRX session engine, which adds state and overhead on a pure transit core; in packet mode the SRX forwards like a plain router, so the only lines left from the security configuration are the three `forwarding-options` commands below.

The two cores are also cross-connected on `ge-0/0/3` (`10.100.100.0/24`), and that link carries OSPF as well, so the core is a triangle — `JUN-CORE1` and `JUN-CORE2` learn each other's routes both directly and through the ISP routers. Each downlink lands in a shared bridge — `Bridge-HQ_1` on `JUN-CORE1`, `Bridge-HQ_2` on `JUN-CORE2` — alongside both FortiGate cluster members, so the segment has two gateways and either node can answer for it. Those bridges sit in their own stub area (area 10, `no-summaries`), so the cluster receives a single default route instead of the whole transit table.

Captured from the running lab with `show configuration | display set` on 2026-11-18 (Junos 24.4R1.9). The root password hash and the OSPF MD5 keys are replaced with `REPLACE_ME`, and everything that comes with the image is omitted: `version`, `root-authentication`, `services ssh` and `web-management`, `syslog`, `license autoupdate`, the `pki ca-profile` stubs, `interfaces fxp0 unit 0`, and the pre-installed security baseline (`screen`, zones and default `policies`). Neither device has a `host-name` configured, so the EVE-NG node names are used below.

## Interface map

| Router | Interface | Address | Role |
| :--- | :--- | :--- | :--- |
| `JUN-CORE1` | `ge-0/0/0` | `10.1.2.1/24` | Uplink to `C2-ISP1` `Gi0/2` |
| | `ge-0/0/1` | `10.0.2.1/24` | Uplink to `C1-ISP1` `Gi0/2` |
| | `ge-0/0/2` | `10.20.0.254/24` | Downlink to `Bridge-HQ_1`, OSPF area 10 (stub) |
| | `ge-0/0/3` | `10.100.100.1/24` | Cross-link to `JUN-CORE2` `ge-0/0/3` |
| | `lo0` | `10.255.255.5/32` | OSPF router ID, passive |
| `JUN-CORE2` | `ge-0/0/0` | `10.2.2.1/24` | Uplink to `C1-ISP2` `Gi0/2` |
| | `ge-0/0/1` | `10.3.2.1/24` | Uplink to `C2-ISP2` `Gi0/2` |
| | `ge-0/0/2` | `10.20.1.254/24` | Downlink to `Bridge-HQ_2`, OSPF area 10 (stub) |
| | `ge-0/0/3` | `10.100.100.2/24` | Cross-link to `JUN-CORE1` `ge-0/0/3` |
| | `lo0` | `10.255.255.6/32` | OSPF router ID, passive |

`ge-0/0/4` is unused and carries no configuration.

## JUN-CORE1

```junos
set security forwarding-options family inet mode packet-based
set security forwarding-options family inet6 mode packet-based
set security forwarding-options family mpls mode packet-based
set interfaces ge-0/0/0 unit 0 family inet address 10.1.2.1/24
set interfaces ge-0/0/1 unit 0 family inet address 10.0.2.1/24
set interfaces ge-0/0/2 unit 0 family inet address 10.20.0.254/24
set interfaces ge-0/0/3 unit 0 family inet address 10.100.100.1/24
set interfaces lo0 unit 0 family inet address 10.255.255.5/32
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface lo0.0 passive
set protocols ospf area 0.0.0.10 stub no-summaries
set protocols ospf area 0.0.0.10 interface ge-0/0/2.0 interface-type p2p
set protocols ospf area 0.0.0.10 interface ge-0/0/2.0 metric 10
set protocols ospf area 0.0.0.10 interface ge-0/0/2.0 authentication md5 1 key "REPLACE_ME"
set routing-options router-id 10.255.255.5
```

## JUN-CORE2

```junos
set security forwarding-options family inet mode packet-based
set security forwarding-options family inet6 mode packet-based
set security forwarding-options family mpls mode packet-based
set interfaces ge-0/0/0 unit 0 family inet address 10.2.2.1/24
set interfaces ge-0/0/1 unit 0 family inet address 10.3.2.1/24
set interfaces ge-0/0/2 unit 0 family inet address 10.20.1.254/24
set interfaces ge-0/0/3 unit 0 family inet address 10.100.100.2/24
set interfaces lo0 unit 0 family inet address 10.255.255.6/32
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface lo0.0 passive
set protocols ospf area 0.0.0.10 stub no-summaries
set protocols ospf area 0.0.0.10 interface ge-0/0/2.0 interface-type p2p
set protocols ospf area 0.0.0.10 interface ge-0/0/2.0 metric 10
set protocols ospf area 0.0.0.10 interface ge-0/0/2.0 authentication md5 1 key "REPLACE_ME"
set routing-options router-id 10.255.255.6
```

## OSPF layout

Both routers run a single OSPF process. Area 0 carries the two ISP uplinks, the `ge-0/0/3` cross-link and the loopback; those interfaces are `point-to-point` with `metric 10` and MD5 authentication (key ID 1), and the loopbacks are `passive`. Area 10 (`0.0.0.10` in `display set` notation) is a stub with `no-summaries` and holds the single `ge-0/0/2` downlink to the bridge shared with the FortiGate cluster: as the ABR, each core injects one Type-3 default into it and no inter-area prefixes, so the cluster routing table stays at a single entry. There are no static routes — the default in area 0 comes from the ISP routers via `default-information originate`.

| Router | Router ID | Neighbors in `Full` | Passive interfaces |
| :--- | :--- | :--- | :--- |
| `JUN-CORE1` | `10.255.255.5` | `C1-ISP1` (10.0.2.254), `C2-ISP1` (10.1.2.254), `JUN-CORE2` (10.100.100.2) | `lo0` |
| `JUN-CORE2` | `10.255.255.6` | `C1-ISP2` (10.2.2.254), `C2-ISP2` (10.3.2.254), `JUN-CORE1` (10.100.100.1) | `lo0` |

`ge-0/0/2` sits in area 10, so the FortiGate peer shows up in that area's neighbor table rather than in the area 0 list above.
