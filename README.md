# Networking: Switching and VLAN Concepts Project

**Author:** Callum Matthews 

---

## Background

This project creates a prototype LAN for a local gym. The network contains separate VLANs for five distinct user groups: **Reception, Finance, Executive, Instructors, and Members**. The native VLAN has been moved away from VLAN 1 (reassigned to VLAN 99).

This project was completed as a part of my Networking: Switching and VLAN Concepts Module. Grade: 92%

---

## Network Overview

A **Multi-Layer Switch (MLS)** was chosen over the router-on-a-stick approach to handle inter-VLAN communication. This decision enables:

- **HSRP (Hot Standby Router Protocol)** for gateway redundancy.
- **STP (Spanning Tree Protocol)** by interconnecting Layer 2 switches, preventing broadcast storms.
- Greater scalability for future network expansion.

A **dedicated DHCP server** is connected to the main MLS and provides address pools for each VLAN, excluding the static IPs reserved for the MLS SVIs.

> **Note:** As this is a prototype, computers from multiple VLANs are connected to the same physical switch for demonstration purposes. In a real deployment, each department would be physically separated.

---

## VLAN Structure

| VLAN | Name | IP Range | Default Gateway | Subnet Mask |
|------|------|----------|-----------------|-------------|
| VLAN 5 | Management | 192.168.5.0/24 | 192.168.5.1 | 255.255.255.0 |
| VLAN 10 | Reception | 192.168.10.0/24 | 192.168.10.1 | 255.255.255.0 |
| VLAN 20 | Finance | 192.168.20.0/24 | 192.168.20.1 | 255.255.255.0 |
| VLAN 30 | Executive | 192.168.30.0/24 | 192.168.30.1 | 255.255.255.0 |
| VLAN 40 | Instructors | 192.168.40.0/24 | 192.168.40.1 | 255.255.255.0 |
| VLAN 50 | Members | 192.168.50.0/24 | 192.168.50.1 | 255.255.255.0 |

**IP Reservation Convention:**
- `.1` — Default gateway (HSRP virtual IP)
- `.2` — Main MLS SVI
- `.3` — Secondary MLS SVI
- `.5` — Left Layer 2 switch
- `.6` — Right Layer 2 switch
- `.7` — Bottom Layer 2 switch
- `.10+` — DHCP pool (available for client allocation)

---

## Traffic Separation Between Departments

Each department is isolated into its own VLAN, creating separate broadcast domains. This provides:

- **Improved security** — users cannot access resources outside their VLAN without explicit routing.
- **Reduced broadcast traffic** — broadcast domains are kept small, improving performance as the network grows.
- **Scalability** — new VLANs can be added in the future; trunk lines can then be updated to permit their traffic.

Trunk lines are configured to only carry approved VLAN traffic. Connections between the DHCP server and the MLS devices are configured as **access links**.

```
interface FastEthernet0/4
 switchport access vlan 10
 switchport mode access
!
interface FastEthernet0/5
 switchport access vlan 20
 switchport mode access
!
interface FastEthernet0/6
 switchport access vlan 30
 switchport mode access
!
interface FastEthernet0/7
 switchport access vlan 40
 switchport mode access
!
interface FastEthernet0/8
 switchport access vlan 50
 switchport mode access
```

---

## Inter-VLAN Communication

Inter-VLAN routing is handled by the Multi-Layer Switches using **Switched Virtual Interfaces (SVIs)**. Each SVI:

- Provides a default gateway for its respective VLAN.
- Is configured with an `ip helper-address` to forward DHCP requests to the dedicated server.
- Includes a description tag for administrator clarity.

**Example SVI configuration (MLS1):**

```
description Default Gateway SVI for 192.168.10.0/24
mac-address 00e0.8f3d.9402
ip address 192.168.10.2 255.255.255.0
ip helper-address 192.168.5.4
standby 10 ip 192.168.10.1
standby 10 priority 150
standby 10 preempt

interface Vlan20
description Default Gateway SVI for 192.168.20.0/24
mac-address 00e0.8f3d.9403
ip address 192.168.20.2 255.255.255.0
ip helper-address 192.168.5.4
standby 20 ip 192.168.20.1
standby 20 priority 150
standby 20 preempt
```

### External Network Connectivity

For demonstration, an external PC (IP: `10.10.10.5`) and switch are connected to the main MLS via FastEthernet port 5. This port is configured with `no switchport`, turning it into a **routed port** capable of forwarding traffic off the internal network — simulating an ISP uplink. All VLAN devices can ping this external network through the MLS.

```
interface FastEthernet0/5
 no switchport
 ip address 10.10.10.1 255.0.0.0
 duplex auto
 speed auto
```

---

## IP Address Allocation (DHCP)

A dedicated DHCP server handles address allocation for all VLANs. It is connected to both the primary and secondary MLS for redundancy (an additional Ethernet module was added to the server for this purpose).

All DHCP pools begin allocation at `.10` to preserve the lower addresses for static device assignments. DNS is set to `8.8.8.8` for all pools.

| Pool Name | Default Gateway | Start IP | Subnet Mask | Max Users |
|-----------|-----------------|----------|-------------|-----------|
| vlan 10 (Reception) | 192.168.10.1 | 192.168.10.10 | 255.255.255.0 | 246 |
| vlan 20 (Finance) | 192.168.20.1 | 192.168.20.10 | 255.255.255.0 | 246 |
| vlan 30 (Executive) | 192.168.30.1 | 192.168.30.10 | 255.255.255.0 | 246 |
| vlan 40 (Instructors) | 192.168.40.1 | 192.168.40.10 | 255.255.255.0 | 246 |
| vlan 50 (Members) | 192.168.50.1 | 192.168.50.10 | 255.255.255.0 | 246 |

---

## Security Measures

All switches and MLS devices are secured with passwords and encryption. A banner message is configured to warn against unauthorised access.

| Access Method | Password |
|---------------|----------|
| Console / Local | `class` |
| Privileged EXEC | `cisco` |
| VTY Lines | `encryption` |

The `service password-encryption` command was applied across all devices and configurations saved to startup-config.

**Example (encrypted output):**
```
banner motd ^CNo Unauthorised Access!^C
!
line con 0
 password 7 0822404F1A0A
 login
!
line vty 0 4
 password 7 0824424D1B0015031B0402
 login
line vty 5 15
 password 7 0824424D1B0015031B0402
 login
```

---

## Redundancy

The topology incorporates redundancy at multiple levels:

### Spanning Tree Protocol (STP)
Each Layer 2 switch is connected to the others and to the main MLS. STP logically blocks select links to prevent broadcast storms and MAC table inconsistencies, while keeping physical redundant paths available for failover.

### HSRP (Hot Standby Router Protocol)
Two Multi-Layer Switches provide gateway redundancy:

| Device | HSRP Priority |
|--------|--------------|
| Main MLS | 150 |
| Secondary MLS | 99 |

Both MLS devices have SVIs for every VLAN and are both connected to the DHCP server, ensuring IP address availability even if one MLS fails.

### EtherChannel
The main MLS connects to the bottom Layer 2 switch via **two crossover cables configured as an EtherChannel (Port-channel1)**. This presents as a single logical link, providing both redundancy (automatic failover if one physical link fails) and load balancing across both links.

```
interface Port-channel1
 description Etherchannel to Bottom switch
 switchport trunk native vlan 99
 switchport trunk allowed vlan 5,10,20,30,40,50,99
 switchport trunk encapsulation dot1q
 switchport mode trunk
```

---

## Files

| File | Description |
|------|-------------|
| `Sem1Project.pkt` | Cisco Packet Tracer topology file |
| `Sem1Project.pdf` | Full project report with screenshots |

---

## References

- Networkforyou (2022) *DHCP Lab for Multiple VLAN in Packet Tracer*, YouTube. https://www.youtube.com/watch?v=FvNVaEJc1xo
- Cisco (2024) *IP Addressing Services Configuration Guide, Cisco IOS XE Gibraltar 16.11.x — Configuring HSRP*. https://www.cisco.com/c/en/us/td/docs/switches/lan/catalyst9300/software/release/16-11/configuration_guide/ip/b_1611_ip_9300_cg/configuring_hsrp.html
