# Small Office Network Design (Cisco Packet Tracer)

A secure, segmented network for a 30-person accounting firm, designed from client requirements and built and verified in Cisco Packet Tracer.

The project covers the full workflow a network engineer follows: **requirements → logical design → physical design → phased build → verification → documentation.**

![Topology](images/topology.png)

---

## Highlights

- **5 user/infrastructure VLANs** plus a parking VLAN and an unused native VLAN
- **VLSM addressing** sized for 50% growth, with a reserved block for future VLANs
- **Router-on-a-stick** inter-VLAN routing on a single ISR4331
- **DHCP** for users, static addressing for infrastructure
- **PAT (NAT overload)** to one public IP, for user VLANs only
- **Guest Wi-Fi isolation:** internet only, nothing internal
- **SSH-only management** from a single admin workstation
- **Port security** and **DHCP snooping** on every access switch

---

## Requirements

| Group | Devices | Needs |
| --- | --- | --- |
| Staff | 25 wired PCs | Internet + file server |
| Managers | 5 wired PCs | Same as staff, separate network |
| Servers | File server, printer | Fixed IP addresses |
| Guests | Up to 20 Wi-Fi devices | Internet only |
| IT | Router and switches | Secure remote management |
| Internet | 1 public IP | 203.0.113.2/30 from the ISP |

The client expects roughly **50% growth** within two years.

---

## Devices

| Device | Model | Role |
| --- | --- | --- |
| R1 | Cisco ISR4331 | Gateway for all VLANs, DHCP server, PAT, ACLs |
| SW1 | Cisco Catalyst 2960 | Access switch: staff PCs, printer |
| SW2 | Cisco Catalyst 2960 | Access switch: managers, server, guest AP, IT-ADMIN; uplink to R1 |
| ISP | Router | Simulated provider; loopback 8.8.8.8 represents the internet |
| AP | AccessPoint-PT | SSID `OFFICE-GUEST`, WPA2-PSK, bridged into VLAN 30 |

---

## VLAN Design

| VLAN | Name | Purpose |
| --- | --- | --- |
| 10 | STAFF | Staff PCs |
| 20 | MANAGERS | Manager PCs |
| 30 | GUESTS | Guest Wi-Fi |
| 40 | SERVERS | File server and printer |
| 99 | NET-MGMT | Switch/router management and IT-ADMIN PC |
| 998 | UNUSED | Parking VLAN for shut-down ports |
| 999 | NATIVE | Native VLAN on trunks (carries no users) |

**Trunks**

| Link | Allowed VLANs | Native |
| --- | --- | --- |
| SW2 ↔ R1 | 10, 20, 30, 40, 99 | 999 |
| SW1 ↔ SW2 | 10, 40, 99 (pruned) | 999 |

---

## IP Addressing Plan

Internal range: `192.168.10.0/24`, allocated largest subnet first. Gateways use the first usable address.

| VLAN | Subnet | Gateway | Usable range | Assignment |
| --- | --- | --- | --- | --- |
| 10 STAFF | 192.168.10.0/26 | .1 | .2 – .62 | DHCP |
| 30 GUESTS | 192.168.10.64/26 | .65 | .66 – .126 | DHCP |
| 20 MANAGERS | 192.168.10.128/28 | .129 | .130 – .142 | DHCP |
| 40 SERVERS | 192.168.10.144/29 | .145 | .146 – .150 | Static |
| 99 NET-MGMT | 192.168.10.152/29 | .153 | .154 – .158 | Static |
| WAN | 203.0.113.0/30 | ISP .1 | R1 .2 | Static |

`192.168.10.160 – .255` is reserved for future VLANs (e.g. VoIP, CCTV).

| Device | Address |
| --- | --- |
| SW1 / SW2 management (VLAN 99 SVI) | .154 / .155 |
| IT-ADMIN PC | .156 |
| File server | .146 |
| Printer | .147 |

---

## Security Policy

| Source | Internal network | Internet | Device management |
| --- | --- | --- | --- |
| Staff | ✅ | ✅ | ❌ |
| Managers | ✅ | ✅ | ❌ |
| Guests | ❌ | ✅ | ❌ |
| Servers / printer | ✅ | ❌ | ❌ |
| IT-ADMIN PC | ✅ | ❌ | ✅ SSH only |

**How it is enforced**

| Control | Where | Purpose |
| --- | --- | --- |
| `GUEST-IN` extended ACL | R1 G0/0/0.30, inbound | Permit DHCP, deny internal, permit internet |
| `NAT-USERS` standard ACL | R1 PAT rule | Only staff, managers and guests are translated |
| `MGMT-ACCESS` standard ACL | VTY lines (`access-class`) | Only the IT-ADMIN PC can log in; others are logged |
| SSH v2 + local users | R1, SW1, SW2 | Encrypted logins, no Telnet |
| Port security | Access ports | Sticky MACs; `restrict` on user ports, `shutdown` on infrastructure ports |
| DHCP snooping | SW1, SW2 | Blocks rogue DHCP servers; only uplinks toward R1 are trusted |
| Unused port hardening | SW1, SW2 | Shut down, access mode, parked in VLAN 998 |

---

## Key Configuration Examples

**Router-on-a-stick subinterface (R1)**
```
interface GigabitEthernet0/0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.192
 ip nat inside
```

**PAT for user VLANs only (R1)**
```
ip access-list standard NAT-USERS
 permit 192.168.10.0 0.0.0.63
 permit 192.168.10.64 0.0.0.63
 permit 192.168.10.128 0.0.0.15
!
ip nat inside source list NAT-USERS interface GigabitEthernet0/0/1 overload
ip route 0.0.0.0 0.0.0.0 203.0.113.1
```

**Guest isolation (R1)**
```
ip access-list extended GUEST-IN
 permit udp any any eq bootps
 deny   ip any 192.168.10.0 0.0.0.255
 permit ip any any
!
interface GigabitEthernet0/0/0.30
 ip access-group GUEST-IN in
```

**Management access (all devices)**
```
ip access-list standard MGMT-ACCESS
 permit host 192.168.10.156
 deny   any log
!
line vty 0 15
 login local
 transport input ssh
 access-class MGMT-ACCESS in
```

Full configurations are in [`configs/`](configs/).

---

## Build Phases

1. **Layer 2:** VLANs, access ports, pruned trunks, unused ports parked and shut down
2. **Inter-VLAN routing:** R1 subinterfaces, switch management SVIs
3. **DHCP + guest Wi-Fi:** pools with excluded ranges, WPA2-PSK SSID
4. **Internet:** default route and PAT
5. **Security:** guest ACL, SSH-only management, port security, DHCP snooping
6. **Verification + documentation**

Each phase was verified before the next began.

---

## Verification

| Test | Expected | Result |
| --- | --- | --- |
| Ping gateway from each VLAN | Success | ✅ |
| Inter-VLAN ping (staff ↔ manager ↔ server) | Success | ✅ |
| Staff, manager, guest → 8.8.8.8 | Success | ✅ |
| Server → 8.8.8.8 | Blocked | ✅ |
| Guest → any internal host | Blocked | ✅ |
| DHCP renew on user devices | Correct subnet + gateway | ✅ |
| SSH from IT-ADMIN to R1/SW1/SW2 | Login | ✅ |
| SSH from a staff PC | Refused | ✅ |
| Telnet from IT-ADMIN | Refused | ✅ |
| Unknown device on a staff port | Dropped, violation logged | ✅ |

---

## Troubleshooting Log

Real problems found during the build, and how verification output exposed them:

| Symptom | Cause | Fix |
| --- | --- | --- |
| Only one trunk shown on SW2; a port stuck in VLAN 1 | Port to R1 was left as an access port | Configured it as a trunk with the correct allowed list |
| R1 missing from `show cdp neighbors` | Router interfaces are shut down by default | `no shutdown` on R1 G0/0/0 |
| Server still reached the internet after NAT change | Old NAT rule was still active alongside the new one | Cleared translations, removed old rule and leftover ACLs |
| A test ping seemed to bypass the guest ACL | Wrong IP pinged; ACL hit counters showed no match | Retested with the correct IP and confirmed with counters |
| VTY ACL had no effect | `access-class` referenced an ACL not yet created | Created the named ACL correctly |
| Staff PCs would lose DHCP | DHCP snooping on SW1 had no trusted uplink | Trusted the uplink toward R1 |

---

## Lessons Learned

- Know what correct output looks like **before** running a `show` command.
- Adding configuration never removes old configuration. Always check `show running-config`.
- Communication needs a path in **both** directions. When a ping fails, ask whether the request or the reply failed.
- ACL hit counters are proof. A single ping result can mislead.
- Least privilege applies everywhere: trunks, NAT, management access, switch ports.

---

## Repository Structure

```
.
├── README.md
├── Project1-Small-Office.pkt
├── configs/
│   ├── R1.txt
│   ├── SW1.txt
│   ├── SW2.txt
│   └── ISP.txt
├── docs/
│   └── Project1-Design-Document.pdf
└── images/
    └── topology.png
```

## How to Use

1. Install [Cisco Packet Tracer](https://www.netacad.com/courses/packet-tracer) (8.x or later).
2. Open `Project1-Small-Office.pkt`.
3. Lab credentials: username `admin` (passwords are lab-only and should never be reused in production).
4. Run the tests in the Verification table from the IT-ADMIN, staff and guest devices.

---

## Next Project

**Project 2: School campus network:** hierarchical design, multilayer switching with SVIs, EtherChannel, Rapid PVST+, HSRP and a wireless LAN controller.

---

## Author

**KAKEMBO USAMA** · CCNA · Aspiring Network Engineer
https://linkedin.com/in/usamakakembo · https://github.com/usamakaks
