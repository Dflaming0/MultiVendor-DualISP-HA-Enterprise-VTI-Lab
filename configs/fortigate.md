# FortiGate HA cluster

Sanitized FortiOS configuration for the two `FortiGate` cluster members in HQ. `FGT-HQ_1` and `FGT-HQ_2` run FGCP in active-passive mode and share both bridges toward the Juniper core, so either member can own the `10.20.0.1` and `10.20.1.1` cluster addresses. `port1` and `port2` face the core (`Bridge-HQ_1`, `Bridge-HQ_2`), while `port5`, `port6` and `port7` are the inside gateways for the `SRV`, `USR` and `DMZ` zones.

OSPF runs in area 10 as a stub: router ID `10.255.255.7`, MD5 authentication and point-to-point links on `port1` and `port2`; `port5`–`port7` are passive, and `lo0` is advertised as a /32 with cost 10.

Addressing follows the plan in the [topology table](../README.md#addressing-plan): every cluster-facing port carries one address that both members share. `port3` and `port4` are FGCP heartbeat devices and carry no address.

Captured from the running lab with `show system global`, `show system interface`, `show system ha` and `show router ospf` on 2026-11-18 and 2026-11-19 (FortiOS 7.4.12). The HA password and both OSPF MD5 keys are replaced with `REPLACE_ME`, and everything that comes with the image is omitted: `system accprofile`, `system admin`, `system replacemsg`, `firewall internet-service-name`, the `naf.root` tunnel, the unused `port8`–`port12` stubs, the empty `config redistribute` blocks, and the default `set vdom "root"`, `set type physical` and `set snmp-index` lines inside each interface. `FGT-HQ_2` is shown with its `system ha` block only, because the rest of its configuration is delivered by FGCP synchronization from the primary.

## Port map

| Port | Address | Connected to | Role |
| :--- | :--- | :--- | :--- |
| `port1` | `10.20.0.1/24` | `Bridge-HQ_1` (`JUN-CORE1` `ge-0/0/2`) | Cluster address, OSPF area 10 |
| `port2` | `10.20.1.1/24` | `Bridge-HQ_2` (`JUN-CORE2` `ge-0/0/2`) | Cluster address, OSPF area 10 |
| `port3` | — | `FGT-HQ_2` `port3` | HA heartbeat |
| `port4` | — | `FGT-HQ_2` `port4` | HA heartbeat (backup) |
| `port5` | `10.20.3.254/24` | `SW-CORE2-HQ` `Gi0/0` | `SRV` gateway |
| `port6` | `10.20.4.254/24` | `SW-CORE1-HQ` `Gi0/1` | `USR` gateway |
| `port7` | `10.20.5.254/24` | `SW-CORE1-HQ` `Gi1/1` | `DMZ` gateway |
| `lo0` | `10.255.255.7/32` | — | Loopback, router ID |

The addresses are listed once because FGCP owns them: the same IP is configured on both members and only the primary answers.

## FGT-HQ_1

```fortios
config system global
    set hostname "FGT-HQ_1"
end
config system interface
    edit "port1"
        set ip 10.20.0.1 255.255.255.0
        set allowaccess ping https ssh http
        set alias "Bridge-HQ_1"
    next
    edit "port2"
        set ip 10.20.1.1 255.255.255.0
        set alias "Bridge-HQ_2"
    next
    edit "port3"
        set alias "HA-heartbeat"
    next
    edit "port4"
        set alias "HA-Heartbeat-Backup"
    next
    edit "port5"
        set ip 10.20.3.254 255.255.255.0
        set alias "SRV-Zone-Gateway"
    next
    edit "port6"
        set ip 10.20.4.254 255.255.255.0
        set alias "USR-Zone-Gateway"
    next
    edit "port7"
        set ip 10.20.5.254 255.255.255.0
        set alias "DMZ-Zone-Gateway"
    next
    edit "lo0"
        set ip 10.255.255.7 255.255.255.255
        set type loopback
    next
end
config system ha
    set group-id 1
    set group-name "HA-Cluster-HQ"
    set mode a-p
    set password ENC REPLACE_ME
    set hbdev "port3" 200 "port4" 100
    set override disable
    set priority 200
end
config router ospf
    set router-id 10.255.255.7
    config area
        edit 0.0.0.10
            set authentication message-digest
            set type stub
        next
    end
    config ospf-interface
        edit "WAN1-OSPF"
            set interface "port1"
            set authentication message-digest
            set network-type point-to-point
            config md5-keys
                edit 1
                    set key-string ENC REPLACE_ME
                next
            end
        next
        edit "WAN2-OSPF"
            set interface "port2"
            set authentication message-digest
            set network-type point-to-point
            config md5-keys
                edit 1
                    set key-string ENC REPLACE_ME
                next
            end
        next
        edit "LO-OSPF"
            set interface "lo0"
            set cost 10
        next
    end
    config network
        edit 1
            set prefix 10.20.0.0 255.255.255.0
            set area 0.0.0.10
            set comments "Bridge-HQ_1"
        next
        edit 2
            set prefix 10.20.1.0 255.255.255.0
            set area 0.0.0.10
            set comments "Bridge-HQ_2"
        next
        edit 3
            set prefix 10.20.3.0 255.255.255.0
            set area 0.0.0.10
            set comments "SRV-Zone"
        next
        edit 4
            set prefix 10.20.4.0 255.255.255.0
            set area 0.0.0.10
            set comments "USR-Zone"
        next
        edit 5
            set prefix 10.20.5.0 255.255.255.0
            set area 0.0.0.10
            set comments "DMZ-Zone"
        next
        edit 6
            set prefix 10.255.255.7 255.255.255.255
            set area 0.0.0.10
        next
    end
    set passive-interface "lo0" "port5" "port6" "port7"
end
```

## FGT-HQ_2

```fortios
config system ha
    set group-id 1
    set group-name "HA-Cluster-HQ"
    set mode a-p
    set password ENC REPLACE_ME
    set hbdev "port3" 200 "port4" 100
    set override disable
    set priority 100
end
```

## HA layout

| Member | Priority | Heartbeat devices | Group |
| :--- | :---: | :--- | :--- |
| `FGT-HQ_1` | 200 | `port3` (200), `port4` (100) | group ID 1, `HA-Cluster-HQ`, active-passive |
| `FGT-HQ_2` | 100 | `port3` (200), `port4` (100) | group ID 1, `HA-Cluster-HQ`, active-passive |

`FGT-HQ_1` carries the higher priority, so it wins the election; `override` is disabled, so the active member keeps the role as long as it is up. The two heartbeat devices give the cluster a redundant path if one of the direct links between the members fails.
