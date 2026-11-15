# Cisco ISP routers

Sanitized Cisco IOS configuration for the four IOSv transit routers in the dual-ISP lab: `C1-ISP1` and `C2-ISP1` sit on the ISP-1 cloud (`pnet0`), `C1-ISP2` and `C2-ISP2` sit on the ISP-2 cloud (`pnet1`). Each pair is joined by a point-to-point inter-ISP link and runs OSPF across it; the customer-facing interfaces are OSPF-enabled but currently have no adjacency. The routers do not carry user prefixes themselves; they provide WAN transit, originate a default route, and PAT Internet-bound traffic.

Captured from the running lab with `show running-config` on 2026-11-15. The OSPF MD5 key is replaced with `REPLACE_ME`, and the EVE-NG node names are used below for readability: all four devices still run the default `hostname Router`. `no shutdown` is shown explicitly because all routed interfaces are up.

## Interface map

| Router | Interface | Address | Role |
| :--- | :--- | :--- | :--- |
| `C1-ISP1` | `Gi0/0` | `172.16.172.200/24` | Uplink to the EVE-NG cloud `pnet0` (ISP-1) |
| | `Gi0/1` | `10.0.1.1/24` | Inter-ISP link to `C2-ISP1` `Gi0/1` |
| | `Gi0/2` | `10.0.2.254/24` | Downlink to `JUN-CORE1` `ge-0/0/1` |
| | `Gi0/3` | `10.0.3.254/24` | Downlink to `CP-GW-BR1_1` `eth1` |
| | `Loopback0` | `10.255.255.1/32` | OSPF router ID |
| `C2-ISP1` | `Gi0/0` | `172.16.172.201/24` | Uplink to the EVE-NG cloud `pnet0` (ISP-1) |
| | `Gi0/1` | `10.0.1.2/24` | Inter-ISP link to `C1-ISP1` `Gi0/1` |
| | `Gi0/2` | `10.1.2.254/24` | Downlink to `JUN-CORE1` `ge-0/0/0` |
| | `Gi0/3` | `10.1.3.254/24` | Downlink to `CP-GW-BR1_2` `eth1` |
| | `Loopback0` | `10.255.255.2/32` | OSPF router ID |
| `C1-ISP2` | `Gi0/0` | `172.16.172.202/24` | Uplink to the EVE-NG cloud `pnet1` (ISP-2) |
| | `Gi0/1` | `10.2.1.1/24` | Inter-ISP link to `C2-ISP2` `Gi0/1` |
| | `Gi0/2` | `10.2.2.254/24` | Downlink to `JUN-CORE2` `ge-0/0/0` |
| | `Gi0/3` | `10.2.3.254/24` | Downlink to `FGT-BR2_1` `port3` |
| | `Loopback0` | `10.255.255.3/32` | OSPF router ID |
| `C2-ISP2` | `Gi0/0` | `172.16.172.203/24` | Uplink to the EVE-NG cloud `pnet1` (ISP-2) |
| | `Gi0/1` | `10.2.1.2/24` | Inter-ISP link to `C1-ISP2` `Gi0/1` |
| | `Gi0/2` | `10.3.2.254/24` | Downlink to `JUN-CORE2` `ge-0/0/1` |
| | `Gi0/3` | `10.3.3.254/24` | Downlink to `FGT-BR2_2` `port3` |
| | `Loopback0` | `10.255.255.4/32` | OSPF router ID |

Both clouds share the upstream gateway `172.16.172.254`.

## C1-ISP1

```cisco
interface Loopback0
 ip address 10.255.255.1 255.255.255.255
 ip ospf 1 area 0
!
interface GigabitEthernet0/0
 ip address 172.16.172.200 255.255.255.0
 ip nat outside
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.0.1.1 255.255.255.0
 ip nat inside
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 REPLACE_ME
 ip ospf network point-to-point
 ip ospf dead-interval 40
 ip ospf 1 area 0
 ip ospf cost 20
 no shutdown
!
interface GigabitEthernet0/2
 ip address 10.0.2.254 255.255.255.0
 ip nat inside
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 REPLACE_ME
 ip ospf network point-to-point
 ip ospf dead-interval 40
 ip ospf 1 area 0
 ip ospf cost 10
 no shutdown
!
interface GigabitEthernet0/3
 ip address 10.0.3.254 255.255.255.0
 ip nat inside
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 REPLACE_ME
 ip ospf network point-to-point
 ip ospf dead-interval 40
 ip ospf 1 area 0
 ip ospf cost 10
 no shutdown
!
router ospf 1
 router-id 10.255.255.1
 default-information originate metric 10
!
ip route 0.0.0.0 0.0.0.0 172.16.172.254
!
access-list 101 permit ip 10.0.0.0 0.255.255.255 any
access-list 101 permit ip 192.168.0.0 0.0.255.255 any
ip nat inside source list 101 interface GigabitEthernet0/0 overload
!
end
```

## C2-ISP1

```cisco
interface Loopback0
 ip address 10.255.255.2 255.255.255.255
 ip ospf 1 area 0
!
interface GigabitEthernet0/0
 ip address 172.16.172.201 255.255.255.0
 ip nat outside
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.0.1.2 255.255.255.0
 ip nat inside
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 REPLACE_ME
 ip ospf network point-to-point
 ip ospf 1 area 0
 ip ospf cost 20
 no shutdown
!
interface GigabitEthernet0/2
 ip address 10.1.2.254 255.255.255.0
 ip nat inside
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 REPLACE_ME
 ip ospf network point-to-point
 ip ospf 1 area 0
 ip ospf cost 10
 no shutdown
!
interface GigabitEthernet0/3
 ip address 10.1.3.254 255.255.255.0
 ip nat inside
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 REPLACE_ME
 ip ospf network point-to-point
 ip ospf 1 area 0
 ip ospf cost 10
 no shutdown
!
router ospf 1
 router-id 10.255.255.2
 default-information originate metric 10
!
ip route 0.0.0.0 0.0.0.0 172.16.172.254
!
access-list 101 permit ip 10.0.0.0 0.255.255.255 any
access-list 101 permit ip 192.168.0.0 0.0.255.255 any
ip nat inside source list 101 interface GigabitEthernet0/0 overload
!
end
```

## C1-ISP2

```cisco
interface Loopback0
 ip address 10.255.255.3 255.255.255.255
 ip ospf 1 area 0
!
interface GigabitEthernet0/0
 ip address 172.16.172.202 255.255.255.0
 ip nat outside
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.2.1.1 255.255.255.0
 ip nat inside
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 REPLACE_ME
 ip ospf network point-to-point
 ip ospf 1 area 0
 ip ospf cost 20
 no shutdown
!
interface GigabitEthernet0/2
 ip address 10.2.2.254 255.255.255.0
 ip nat inside
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 REPLACE_ME
 ip ospf network point-to-point
 ip ospf 1 area 0
 ip ospf cost 10
 no shutdown
!
interface GigabitEthernet0/3
 ip address 10.2.3.254 255.255.255.0
 ip nat inside
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 REPLACE_ME
 ip ospf network point-to-point
 ip ospf 1 area 0
 ip ospf cost 10
 no shutdown
!
router ospf 1
 router-id 10.255.255.3
 default-information originate metric 20
!
ip route 0.0.0.0 0.0.0.0 172.16.172.254
!
access-list 101 permit ip 10.0.0.0 0.255.255.255 any
access-list 101 permit ip 192.168.0.0 0.0.255.255 any
ip nat inside source list 101 interface GigabitEthernet0/0 overload
!
end
```

## C2-ISP2

```cisco
interface Loopback0
 ip address 10.255.255.4 255.255.255.255
 ip ospf 1 area 0
!
interface GigabitEthernet0/0
 ip address 172.16.172.203 255.255.255.0
 ip nat outside
 no shutdown
!
interface GigabitEthernet0/1
 ip address 10.2.1.2 255.255.255.0
 ip nat inside
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 REPLACE_ME
 ip ospf network point-to-point
 ip ospf 1 area 0
 ip ospf cost 20
 no shutdown
!
interface GigabitEthernet0/2
 ip address 10.3.2.254 255.255.255.0
 ip nat inside
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 REPLACE_ME
 ip ospf network point-to-point
 ip ospf 1 area 0
 ip ospf cost 10
 no shutdown
!
interface GigabitEthernet0/3
 ip address 10.3.3.254 255.255.255.0
 ip nat inside
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 REPLACE_ME
 ip ospf network point-to-point
 ip ospf 1 area 0
 ip ospf cost 10
 no shutdown
!
router ospf 1
 router-id 10.255.255.4
 default-information originate metric 20
!
ip route 0.0.0.0 0.0.0.0 172.16.172.254
!
access-list 101 permit ip 10.0.0.0 0.255.255.255 any
access-list 101 permit ip 192.168.0.0 0.0.255.255 any
ip nat inside source list 101 interface GigabitEthernet0/0 overload
!
end
```

## OSPF layout

All four routers run a single OSPF process in Area 0. The inter-ISP link and both customer downlinks are `point-to-point` and use MD5 authentication with key ID 1. Loopbacks are advertised as `/32`.

| Router | Router ID | Default metric | Interface costs |
| :--- | :--- | :---: | :--- |
| `C1-ISP1` | `10.255.255.1` | 10 | `Gi0/1` 20, `Gi0/2` 10, `Gi0/3` 10 |
| `C2-ISP1` | `10.255.255.2` | 10 | `Gi0/1` 20, `Gi0/2` 10, `Gi0/3` 10 |
| `C1-ISP2` | `10.255.255.3` | 20 | `Gi0/1` 20, `Gi0/2` 10, `Gi0/3` 10 |
| `C2-ISP2` | `10.255.255.4` | 20 | `Gi0/1` 20, `Gi0/2` 10, `Gi0/3` 10 |
