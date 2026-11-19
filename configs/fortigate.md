# FortiGate HA cluster

Sanitized FortiOS configuration for the two `FortiGate` cluster members in HQ. `FGT-HQ_1` and `FGT-HQ_2` run FGCP in active-passive mode and share both bridges toward the Juniper core, so either member can own the `10.20.0.1` and `10.20.1.1` cluster addresses. `port1` and `port2` face the core (`Bridge-HQ_1`, `Bridge-HQ_2`), while `port5`, `port6` and `port7` are the inside gateways for the `SRV`, `USR` and `DMZ` zones.

Addressing follows the plan in the [topology table](../README.md#addressing-plan): every cluster-facing port carries one address that both members share. `port3` and `port4` are FGCP heartbeat devices and carry no address.

Captured from the running lab with `show system global`, `show system interface` and `show system ha` on 2026-11-18 and 2026-11-19 (FortiOS 7.4.12). The HA password is replaced with `REPLACE_ME`, and everything that comes with the image is omitted: `system accprofile`, `system admin`, `system replacemsg`, `firewall internet-service-name`, the `naf.root` tunnel, the unused `port8`–`port12` stubs, and the default `set vdom "root"`, `set type physical` and `set snmp-index` lines inside each interface. `FGT-HQ_2` is shown with its `system ha` block only, because the rest of its configuration is delivered by FGCP synchronization from the primary.

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
