# Juniper vSRX core

Sanitized Junos configuration for the two `vSRX` routers in the dual-ISP lab. `JUN-CORE1` terminates one link from each ISP ASBR pair (`C1-ISP1`, `C2-ISP1`), `JUN-CORE2` does the same on the ISP-2 side (`C1-ISP2`, `C2-ISP2`). vSRX is used as the Juniper core because it runs the same Junos routing stack as the branch SRX line, so OSPF and route selection behave as they would on hardware.

Both devices are switched to packet-based mode. In flow mode every packet would cross the SRX session engine, which adds state and overhead on a pure transit core; in packet mode the SRX forwards like a plain router, so the only lines left from the security configuration are the three `forwarding-options` commands below.

The two cores are also cross-connected on `ge-0/0/4` (`10.100.100.0/24`), and that link carries OSPF as well, so the core is a triangle — `JUN-CORE1` and `JUN-CORE2` learn each other's routes both directly and through the ISP routers. The four FortiGate-facing links are passive, because FortiGate does not run OSPF yet.

Captured from the running lab with `show configuration | display set` on 2026-11-16 (Junos 24.4R1.9). The root password hash and the OSPF MD5 keys are replaced with `REPLACE_ME`, and everything that comes with the image is omitted: `version`, `root-authentication`, `services ssh` and `web-management`, `syslog`, `license autoupdate`, the `pki ca-profile` stubs, `interfaces fxp0 unit 0`, and the pre-installed security baseline (`screen`, zones and default `policies`). Neither device has a `host-name` configured, so the EVE-NG node names are used below.

## Interface map

| Router | Interface | Address | Role |
| :--- | :--- | :--- | :--- |
| `JUN-CORE1` | `ge-0/0/0` | `10.1.2.1/24` | Uplink to `C2-ISP1` `Gi0/2` |
| | `ge-0/0/1` | `10.0.2.1/24` | Uplink to `C1-ISP1` `Gi0/2` |
| | `ge-0/0/2` | `192.168.100.254/24` | Downlink to `FGT-HQ_1` `port1` |
| | `ge-0/0/3` | `192.168.110.254/24` | Downlink to `FGT-HQ_2` `port2` |
| | `ge-0/0/4` | `10.100.100.1/24` | Cross-link to `JUN-CORE2` `ge-0/0/4` (`MGMT`) |
| | `lo0` | `10.255.255.5/32` | OSPF router ID, passive |
| `JUN-CORE2` | `ge-0/0/0` | `10.2.2.1/24` | Uplink to `C1-ISP2` `Gi0/2` |
| | `ge-0/0/1` | `10.3.2.1/24` | Uplink to `C2-ISP2` `Gi0/2` |
| | `ge-0/0/2` | `192.168.200.254/24` | Downlink to `FGT-HQ_2` `port1` |
| | `ge-0/0/3` | `192.168.210.254/24` | Downlink to `FGT-HQ_1` `port2` |
| | `ge-0/0/4` | `10.100.100.2/24` | Cross-link to `JUN-CORE1` `ge-0/0/4` (`MGMT`) |
| | `lo0` | `10.255.255.6/32` | OSPF router ID, passive |

## JUN-CORE1

```junos
set security forwarding-options family inet mode packet-based
set security forwarding-options family inet6 mode packet-based
set security forwarding-options family mpls mode packet-based
set interfaces ge-0/0/0 unit 0 family inet address 10.1.2.1/24
set interfaces ge-0/0/1 unit 0 family inet address 10.0.2.1/24
set interfaces ge-0/0/2 unit 0 family inet address 192.168.100.254/24
set interfaces ge-0/0/3 unit 0 family inet address 192.168.110.254/24
set interfaces ge-0/0/4 unit 0 family inet address 10.100.100.1/24
set interfaces lo0 unit 0 family inet address 10.255.255.5/32
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface ge-0/0/4.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/4.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/4.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0 passive
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 passive
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface lo0.0 passive
set routing-options router-id 10.255.255.5
```

## JUN-CORE2

```junos
set security forwarding-options family inet mode packet-based
set security forwarding-options family inet6 mode packet-based
set security forwarding-options family mpls mode packet-based
set interfaces ge-0/0/0 unit 0 family inet address 10.2.2.1/24
set interfaces ge-0/0/1 unit 0 family inet address 10.3.2.1/24
set interfaces ge-0/0/2 unit 0 family inet address 192.168.200.254/24
set interfaces ge-0/0/3 unit 0 family inet address 192.168.210.254/24
set interfaces ge-0/0/4 unit 0 family inet address 10.100.100.2/24
set interfaces lo0 unit 0 family inet address 10.255.255.6/32
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/0.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/1.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface ge-0/0/4.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/4.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/4.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0 passive
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/2.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 passive
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 metric 10
set protocols ospf area 0.0.0.0 interface ge-0/0/3.0 authentication md5 1 key "REPLACE_ME"
set protocols ospf area 0.0.0.0 interface lo0.0 passive
set routing-options router-id 10.255.255.6
```

## OSPF layout

Both routers run a single OSPF process in Area 0. Every interface is `point-to-point` with `metric 10` and MD5 authentication (key ID 1); the loopbacks and the four FortiGate-facing links are `passive`, so the only adjacencies are the two ISP uplinks plus the `ge-0/0/4` cross-link between the cores. There are no static routes — the default comes from the ISP routers via `default-information originate`.

| Router | Router ID | Neighbors in `Full` | Passive interfaces |
| :--- | :--- | :--- | :--- |
| `JUN-CORE1` | `10.255.255.5` | `C1-ISP1` (10.0.2.254), `C2-ISP1` (10.1.2.254), `JUN-CORE2` (10.100.100.2) | `ge-0/0/2`, `ge-0/0/3`, `lo0` |
| `JUN-CORE2` | `10.255.255.6` | `C1-ISP2` (10.2.2.254), `C2-ISP2` (10.3.2.254), `JUN-CORE1` (10.100.100.1) | `ge-0/0/2`, `ge-0/0/3`, `lo0` |
