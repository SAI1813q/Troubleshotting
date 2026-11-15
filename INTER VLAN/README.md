# 🌐 Inter-VLAN Routing Troubleshooting Lab

A Cisco Packet Tracer exercise focused on diagnosing and correcting inter-VLAN connectivity problems in two small networks. The supplied task states that inter-VLAN routing is not working and that there is one misconfiguration per network device.

This walkthrough records the configuration evidence, root causes, corrections, and PC-to-PC ping verification. Screenshots are stored in the [Photos](./Photos/) folder.

## 📋 Lab Overview

- **Platform:** Cisco Packet Tracer
- **Switches:** Catalyst 2960
- **Router:** Cisco 2911 (R2, as labelled in the topology and CLI evidence)
- **Network 1:** VLAN 10 (10.0.1.0/24) and VLAN 20 (10.0.2.0/24), connected through SW1
- **Network 2:** VLAN 30 (10.0.3.0/24) and VLAN 40 (10.0.4.0/24), connected through SW2 and R2 router-on-a-stick
- **Verification:** PC command-prompt ping tests

## 🗺️ Original Topology and Task

The exercise contains two separate inter-VLAN routing networks. The task is to identify the configuration error in each network device and restore communication between the VLANs.

![Original Inter-VLAN troubleshooting topology and task](./Photos/Screenshot%202026-11-14%20123913.png)

---

## 🔍 Troubleshooting Workflow

Each network is documented using:

**Symptom → Evidence → Root Cause → Correction → Verification**

## 🧩 1. Network 1 — SW1 VLAN Assignment and VLAN Database

### Symptom

Hosts in VLAN 10 and VLAN 20 could not communicate through inter-VLAN routing.

### Evidence and diagnosis

I inspected the VLAN database using:

```cisco
show vlan brief
```

The initial output showed VLAN 11 assigned to FastEthernet0/1, even though the topology identifies that port as belonging to VLAN 10. VLAN 20 was present on FastEthernet0/2.

![SW1 initial VLAN database, correction of Fa0/1, removal of VLAN 11, and final VLAN verification](./Photos/Screenshot%202026-11-14%20125017.png)

### Root cause

The access port connected to the VLAN 10 host was assigned to the wrong VLAN (VLAN 11), and VLAN 11 was an unintended VLAN in this lab configuration.

### Correction

I corrected the access-port assignment and removed the incorrect VLAN:

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
- VLAN 11 no longer present.

I then tested connectivity from PC1 using ping. The PC screenshot shows successful replies to the tested remote VLAN gateway/host after the correction.

![PC1 ping verification after the SW1 VLAN correction](./Photos/Screenshot%202026-11-14%20125046.png)

---

## 🔌 2. Network 2 — R2 Subinterface IP Address and SW2 Trunking

### Symptom

PC3 in VLAN 30 could not reach its gateway or PC4 in VLAN 40.

### Evidence and diagnosis

I checked the router subinterface addresses with:

```cisco
show ip interface brief
```

The initial output showed the incorrect address `10.0.0.1` on subinterface GigabitEthernet0/0.30. The VLAN 30 gateway needed to be in the `10.0.3.0/24` subnet.

I also checked the switch trunk status before troubleshooting:

```cisco
show interfaces trunk
```

The command returned no trunk entries, indicating that no operational trunk was being displayed at that time.

![R2 subinterface IP before and after correction, including show ip interface brief output](./Photos/Screenshot%202026-11-14%20123213.png)

![SW2 trunk check before troubleshooting, with no trunk entries displayed](./Photos/Screenshot%202026-11-14%20123756.png)

### Root causes

1. **R2:** The IP address configured on GigabitEthernet0/0.30 was incorrect for VLAN 30.
2. **SW2:** The switch uplink GigabitEthernet0/1 was not configured as a trunk.

### Corrections

On R2, I corrected the VLAN 30 subinterface address:

```cisco
interface GigabitEthernet0/0.30
 no ip address 10.0.0.1 255.255.255.0
 ip address 10.0.3.1 255.255.255.0
```

The final `show ip interface brief` output showed:

- GigabitEthernet0/0.30 — 10.0.3.1, up/up
- GigabitEthernet0/0.40 — 10.0.4.1, up/up

On SW2, I enabled trunking on the uplink:

```cisco
interface GigabitEthernet0/1
 switchport mode trunk
```

The interface produced line-protocol down/up notifications after the configuration change.

![SW2 GigabitEthernet0/1 configured for trunk mode and interface state notifications](./Photos/Screenshot%202026-11-14%20123805.png)

### Verification

I verified the restored connectivity from PC3 using ping tests.

The PC3 command prompt shows that the initial tests to `10.0.3.1` and `10.0.4.10` failed. After the corrections, the gateway test to `10.0.3.1` received 4/4 replies, and the PC4 test to `10.0.4.10` received 3/4 replies. The first packet loss is consistent with initial ARP resolution; subsequent replies demonstrate successful connectivity.

**Verification method used:** PC-to-PC/gateway ping tests. I did not capture a post-correction `show interfaces trunk` output, so the final trunk operational state is not separately documented by that command.

![PC3 before-and-after ping tests showing restored reachability to the VLAN 30 gateway and PC4 in VLAN 40](./Photos/Screenshot%202026-11-14%20123904.png)

---

## ✅ Final Outcome

| Network | Device | Root cause | Verification |
|---|---|---|---|
| Network 1 | SW1 | Fa0/1 assigned to VLAN 11 instead of VLAN 10; unintended VLAN 11 present | `show vlan brief` and PC1 ping tests |
| Network 2 | R2 | Incorrect IP address on G0/0.30 | `show ip interface brief` and PC3 ping tests |
| Network 2 | SW2 | G0/1 was not configured as a trunk | Initial `show interfaces trunk` was empty; connectivity restored after enabling trunk mode and ping testing |

## 🛠️ Commands and Skills Practiced

- `show vlan brief` — inspect VLANs and access-port membership.
- `switchport mode access` and `switchport access vlan` — configure access-port VLAN membership.
- `no vlan 11` — remove an unintended VLAN from the VLAN database.
- `show ip interface brief` — inspect router subinterface addressing and interface state.
- `show interfaces trunk` — check operational trunk interfaces.
- `switchport mode trunk` — configure a switch uplink as a trunk.
- `ping` — verify gateway reachability and end-to-end communication between VLANs.

**Result:** Corrected the SW1 VLAN assignment and the Network 2 router subinterface/trunk configuration. PC ping tests then demonstrated restored inter-VLAN connectivity in both networks.
