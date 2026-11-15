# 🌐 Inter-VLAN Routing Troubleshooting Lab

A Cisco Packet Tracer exercise focused on diagnosing and correcting inter-VLAN connectivity problems in two networks. The task states that inter-VLAN routing is not functioning and that there is one misconfiguration per network device.

This walkthrough follows the actual command sequence and separates initial diagnostic evidence from configuration changes and PC-based verification.

## 📋 Lab Overview

- **Platform:** Cisco Packet Tracer
- **Network 1:** VLAN 10 (10.0.1.0/24) and VLAN 20 (10.0.2.0/24), through SW1
- **Network 2:** VLAN 30 (10.0.3.0/24) and VLAN 40 (10.0.4.0/24), through SW2 and R2
- **Verification:** CLI show commands and PC ping tests

## 🗺️ Original Topology and Task

![Original topology and troubleshooting task](./Photos/Screenshot%202026-11-14%20123913.png)

---

## 🔍 Troubleshooting Workflow

**Symptom → Evidence → Root Cause → Correction → Verification**

## 🧩 1. Network 1 — SW1 VLAN Assignment

### Symptom

PC1 could not reach its VLAN 10 gateway.

### Evidence — initial switch configuration

I checked the VLAN database:

```cisco
show vlan brief
```

The initial output shows:
- VLAN 10 exists, but no access port is assigned to it.
- VLAN 11 has FastEthernet0/1 assigned.
- VLAN 20 has FastEthernet0/2 assigned.

This is the key initial evidence: Fa0/1, connected to the VLAN 10 PC, was placed in VLAN 11.

![SW1 initial show vlan brief output: Fa0/1 is in VLAN 11 and Fa0/2 is in VLAN 20](./Photos/Screenshot%202026-11-14%20124857.png)

The initial PC1 ping to its gateway `10.0.1.1` also fails with 100% packet loss.

![PC1 initial ping failure to 10.0.1.1](./Photos/Screenshot%202026-11-14%20125046.png)

### Root cause

SW1 FastEthernet0/1 was assigned to VLAN 11 instead of VLAN 10. VLAN 11 was not required by the given topology.

### Correction

I changed the access-port VLAN and removed the unintended VLAN:

```cisco
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
exit
no vlan 11
```

### Verification

I ran `show vlan brief` again. The final output shows:
- VLAN 10 active with Fa0/1.
- VLAN 20 active with Fa0/2.
- VLAN 11 removed.

![SW1 correction and final show vlan brief output](./Photos/Screenshot%202026-11-14%20125017.png)

I then repeated the PC1 tests:
- `ping 10.0.1.1`: 4/4 replies (0% loss).
- `ping 10.0.2.10`: 3/4 replies (25% loss).

The replies from `10.0.2.10` demonstrate connectivity from PC1 to the host in VLAN 20. The first lost packet is consistent with initial ARP resolution.

![PC1 before-and-after ping tests: gateway and VLAN 20 host are reachable after the correction](./Photos/Screenshot%202026-11-14%20125046.png)

---

## 🔌 2. Network 2 — R2 Subinterface IP and SW2 Trunk

### Symptom

PC3 in VLAN 30 could not reach its gateway or PC4 in VLAN 40.

### Evidence — initial PC connectivity

Before making changes, PC3 pinged:
- `10.0.4.10`: 0/4 replies.
- `10.0.3.1`: 0/4 replies.

![PC3 initial ping failures to 10.0.4.10 and 10.0.3.1](./Photos/Screenshot%202026-11-14%20123323.png)

### Evidence — R2 initial subinterface address

I checked the router interfaces with:

```cisco
show ip interface brief
```

The initial output shows GigabitEthernet0/0.30 configured with `10.0.0.1`, which is outside the VLAN 30 subnet (`10.0.3.0/24`). The VLAN 40 subinterface uses `10.0.4.1`.

![R2 initial show ip interface brief output showing the incorrect G0/0.30 address](./Photos/Screenshot%202026-11-14%201232139.png)

### Evidence — SW2 initial trunk status

Before configuring the switch, I checked:

```cisco
do show interfaces trunk
```

The command returned no trunk entries. I also checked the interface brief output; no operational trunk was shown.

![SW2 initial trunk and interface checks before troubleshooting](./Photos/Screenshot%202026-11-14%20123756.png)

### Root causes

1. **R2:** The IP address on GigabitEthernet0/0.30 was incorrect for VLAN 30.
2. **SW2:** The uplink GigabitEthernet0/1 was not configured as a trunk.

### Corrections

**On R2**, I removed the incorrect address and configured the correct VLAN 30 gateway:

```cisco
interface GigabitEthernet0/0.30
 no ip address 10.0.0.1 255.255.255.0
 ip address 10.0.3.1 255.255.255.0
```

The command sequence and subsequent `show ip interface brief` output show G0/0.30 as `10.0.3.1`, up/up, and G0/0.40 as `10.0.4.1`, up/up.

![R2 correction of G0/0.30 and post-change show ip interface brief output](./Photos/Screenshot%202026-11-14%20123213.png)

**On SW2**, I configured the uplink:

```cisco
interface GigabitEthernet0/1
 switchport mode trunk
```

The interface generated line-protocol down/up notifications after the change.

![SW2 G0/1 trunk-mode configuration and interface state notifications](./Photos/Screenshot%202026-11-14%20123805.png)

### Verification — PC3 ping tests

After the corrections, I repeated the ping tests from PC3:
- `ping 10.0.3.1`: 4/4 replies (0% loss).
- `ping 10.0.4.10`: 3/4 replies (25% loss).

The same screenshot captures the initial failed attempts followed by successful replies after the corrections. This demonstrates restored gateway reachability and connectivity from VLAN 30 to the PC4 host in VLAN 40.

![PC3 before-and-after ping tests: 10.0.3.1 succeeds 4/4 and 10.0.4.10 succeeds 3/4 after troubleshooting](./Photos/Screenshot%202026-11-14%20123904.png)

**Verification note:** The initial `show interfaces trunk` output is included as diagnostic evidence. I did not capture a second `show interfaces trunk` output after the change. Therefore, I use the successful PC ping tests as the recorded connectivity verification and do not claim a post-change trunk-table check.

---

## ✅ Final Outcome

| Network | Device | Root cause | Evidence and verification |
|---|---|---|---|
| Network 1 | SW1 | Fa0/1 assigned to VLAN 11 instead of VLAN 10 | Initial/final `show vlan brief`; PC1 gateway and inter-VLAN pings |
| Network 2 | R2 | Incorrect G0/0.30 address (`10.0.0.1` instead of `10.0.3.1`) | Initial/final `show ip interface brief`; PC3 gateway ping |
| Network 2 | SW2 | G0/1 not configured as a trunk | Initial empty trunk check; trunk-mode command and interface notifications; PC3 inter-VLAN ping |

## 🛠️ Commands and Skills Practiced

- `show vlan brief` — inspect VLANs and access-port membership.
- `switchport mode access` / `switchport access vlan 10` — assign an access port to the correct VLAN.
- `no vlan 11` — remove an unintended VLAN.
- `show ip interface brief` — inspect router subinterface addresses and interface state.
- `show interfaces trunk` — inspect operational trunk interfaces.
- `switchport mode trunk` — configure a switch uplink as a trunk.
- `ping` — test gateway reachability and inter-VLAN connectivity.

**Result:** Corrected the SW1 access VLAN, R2 VLAN 30 subinterface IP address, and SW2 trunk configuration. The before-and-after PC ping screenshots demonstrate restored connectivity in both networks.
