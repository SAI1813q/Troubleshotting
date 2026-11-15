# 🌐 Inter-VLAN Routing Troubleshooting Lab

A Cisco Packet Tracer exercise focused on diagnosing and correcting inter-VLAN connectivity problems in two networks. The task requires identifying the misconfiguration in each network and restoring communication between the VLANs.

This walkthrough uses the actual CLI and PC command-prompt screenshots to separate the initial symptoms from the diagnostic evidence and final verification.

## 📋 Lab Overview

- **Platform:** Cisco Packet Tracer
- **Network 1:** VLAN 10 (10.0.1.0/24) and VLAN 20 (10.0.2.0/24), connected through SW1
- **Network 2:** VLAN 30 (10.0.3.0/24) and VLAN 40 (10.0.4.0/24), connected through SW2 and router R2
- **Verification:** Switch/router show commands and PC ping tests

## 🗺️ Original Topology and Task

The exercise contains two independent inter-VLAN routing networks. The task states that inter-VLAN routing is not functioning properly and asks for the configuration errors to be found and corrected.

![Original Inter-VLAN troubleshooting topology and task](./Photos/Screenshot%202026-11-14%20123913.png)

---

## 🔍 Troubleshooting Workflow

Each network is documented using:

**Symptom → Evidence → Root Cause → Correction → Verification**

## 🧩 1. Network 1 — SW1 VLAN Assignment

### Symptom

PC1 could not reach its VLAN 10 gateway or the host in VLAN 20.

### Evidence and diagnosis

I checked the switch VLAN database with:

```cisco
show vlan brief
```

The initial SW1 output showed that FastEthernet0/1 was assigned to VLAN 11, although the connected PC1 belongs to VLAN 10. VLAN 20 was present for the second network segment. The PC1 command prompt also showed the initial connectivity failure: the first ping to `10.0.1.1` received 0/4 replies.

![SW1 VLAN database and correction showing the incorrect VLAN 11 assignment, port reassignment, removal of VLAN 11, and final VLAN membership](./Photos/Screenshot%202026-11-14%20125017.png)

![PC1 initial ping failure to its VLAN 10 gateway](./Photos/Screenshot%202026-11-14%20125046.png)

### Root cause

FastEthernet0/1 was assigned to the wrong access VLAN (VLAN 11) instead of VLAN 10. VLAN 11 was not required by the given topology.

### Correction

I assigned FastEthernet0/1 to VLAN 10 and removed the unintended VLAN 11:

```cisco
interface FastEthernet0/1
 switchport mode access
 switchport access vlan 10
exit
no vlan 11
```

FastEthernet0/2 remained assigned to VLAN 20.

### Verification

I ran `show vlan brief` again. The final output showed:

- VLAN 10 active with FastEthernet0/1.
- VLAN 20 active with FastEthernet0/2.
- VLAN 11 removed from the VLAN database.

I then repeated the PC1 ping tests:

- `ping 10.0.1.1`: 4/4 replies (0% loss).
- `ping 10.0.2.10`: 3/4 replies (25% loss).

The successful replies from `10.0.2.10` confirm that PC1 could communicate with the host in the other VLAN. The initial single lost packet is consistent with ARP resolution on the first communication attempt.

![PC1 successful gateway ping and inter-VLAN ping to PC2 after the SW1 correction](./Photos/Screenshot%202026-11-14%20125046.png)

---

## 🔌 2. Network 2 — R2 Subinterface Address and SW2 Trunk

### Symptom

PC3 in VLAN 30 could not reach its default gateway or PC4 in VLAN 40.

### Evidence and diagnosis

**PC-side evidence before troubleshooting**

Before making the corrections, I tested from PC3:

- `ping 10.0.4.10`: 0/4 replies.
- `ping 10.0.3.1`: 0/4 replies.

This established that PC3 could not reach either its VLAN 30 gateway or the PC4 host in VLAN 40.

![PC3 before troubleshooting: both the VLAN 40 host and VLAN 30 gateway pings fail](./Photos/Screenshot%202026-11-14%20123323.png)

**Router-side evidence**

I inspected the router interfaces with:

```cisco
show ip interface brief
```

The initial output showed the incorrect address `10.0.0.1` on GigabitEthernet0/0.30. The VLAN 30 gateway needed to use the `10.0.3.0/24` subnet.

![R2 initial subinterface IP address and corrected address, with show ip interface brief output](./Photos/Screenshot%202026-11-14%20123213.png)

**Switch-side evidence**

Before troubleshooting, I checked the trunk table on SW2:

```cisco
show interfaces trunk
```

The command returned no trunk entries, so there was no operational trunk displayed at that time.

![SW2 initial show interfaces trunk output with no trunk entries](./Photos/Screenshot%202026-11-14%20123756.png)

### Root causes

1. **R2:** The IP address configured on GigabitEthernet0/0.30 was incorrect for VLAN 30.
2. **SW2:** The uplink GigabitEthernet0/1 had not been configured as a trunk.

### Corrections

**On R2**, I removed the incorrect subinterface address and configured the correct VLAN 30 gateway:

```cisco
interface GigabitEthernet0/0.30
 no ip address
 ip address 10.0.3.1 255.255.255.0
```

The final `show ip interface brief` output showed:

| Interface | IP address | Status |
|---|---|---|
| G0/0.30 | 10.0.3.1 | up/up |
| G0/0.40 | 10.0.4.1 | up/up |

**On SW2**, I enabled trunk mode on the uplink:

```cisco
interface GigabitEthernet0/1
 switchport mode trunk
```

The interface generated line-protocol down/up notifications after the configuration change.

![SW2 GigabitEthernet0/1 configured with switchport mode trunk and interface state notifications](./Photos/Screenshot%202026-11-14%20123805.png)

### Verification

After correcting the router subinterface address and enabling trunk mode on SW2, I repeated the ping tests from PC3.

- `ping 10.0.3.1`: 4/4 replies (0% loss).
- `ping 10.0.4.10`: 3/4 replies (25% loss).

The first ping to each destination had timed out during the initial attempts. The repeated tests then received replies from the VLAN 30 gateway and PC4 in VLAN 40. This demonstrates restored gateway reachability and inter-VLAN connectivity.

![PC3 after troubleshooting: 4/4 replies from 10.0.3.1 and 3/4 replies from 10.0.4.10](./Photos/Screenshot%202026-11-14%20123904.png)

**Verification note:** The initial `show interfaces trunk` output is included as diagnostic evidence. I did not capture a second `show interfaces trunk` output after the correction, so the final operational trunk state is not claimed as directly verified by that command. The post-fix PC ping tests are the recorded connectivity verification.

---

## ✅ Final Outcome

| Network | Device | Root cause | Evidence of correction |
|---|---|---|---|
| Network 1 | SW1 | Fa0/1 assigned to VLAN 11 instead of VLAN 10; VLAN 11 was not required | Final `show vlan brief` and PC1 ping tests |
| Network 2 | R2 | Incorrect IP address on G0/0.30 | `show ip interface brief` shows 10.0.3.1 up/up; PC3 ping tests |
| Network 2 | SW2 | G0/1 was not configured as a trunk | Initial empty trunk table, trunk-mode configuration, and successful PC3 inter-VLAN ping tests |

## 🛠️ Commands and Skills Practiced

- `show vlan brief` — inspect VLANs and access-port membership.
- `switchport mode access` and `switchport access vlan` — configure access-port VLAN membership.
- `no vlan 11` — remove an unintended VLAN from the VLAN database.
- `show ip interface brief` — inspect router subinterface addressing and interface state.
- `show interfaces trunk` — check operational trunk interfaces.
- `switchport mode trunk` — configure a switch uplink as a trunk.
- `ping` — verify gateway reachability and communication between VLANs.

**Result:** Corrected the SW1 access-port VLAN assignment, the R2 VLAN 30 subinterface address, and the SW2 trunk configuration. The before-and-after PC screenshots show that connectivity was restored in both networks.
