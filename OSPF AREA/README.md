# 🌐 OSPF Multi-Area Troubleshooting Lab

A practical Cisco Packet Tracer troubleshooting exercise focused on diagnosing and correcting OSPF issues across multiple areas. The lab scenario states that routers are not receiving the expected OSPF routes and that R5 is missing the expected `10.0.0.0/8` summary route.

The work below documents the issues found, the evidence used to identify them, the configuration changes made, and the verification performed. Screenshots are from the `Photos` folder in this directory.

## 📋 Lab Overview

- **Platform:** Cisco Packet Tracer
- **Routing protocol:** OSPF, process ID 1
- **Routers involved:** R1, R2, R3, R4 and R5
- **Areas:** Area 0 (backbone), Area 1 and Area 2
- **Main objective:** Restore the required OSPF advertisements and adjacencies, and ensure R5 learns the expected inter-area summary route.

### 🗺️ Original Topology and Task

The topology is divided into three OSPF areas. The task specifically identifies missing OSPF routes and the missing `10.0.0.0/8` summary route on R5.

![Original multi-area OSPF topology and task](./Photos/Screenshot%202026-11-14%20165319.png)

---

## 🔍 Troubleshooting Workflow

Each issue is documented using the same sequence:

**Symptom → Evidence → Root cause → Correction → Verification**

### 🧩 1. R1 — Loopback Address Not Advertised into OSPF

**Symptom**

The loopback address `1.1.1.1` was not included in OSPF advertisements as required by the lab.

**Evidence and diagnosis**

I checked the OSPF configuration using `show ip protocols`. The routing network statements showed the existing Area 1 networks, but the loopback address needed an OSPF network statement in Area 0.

![R1 initial OSPF protocol information](./Photos/Screenshot%202026-11-14%20205758.png)

**Root cause**

The loopback interface address was not included in the OSPF process for Area 0.

**Correction**

Under OSPF process 1, I added the loopback host address:

```cisco
router ospf 1
 network 1.1.1.1 0.0.0.0 area 0
```

The `0.0.0.0` wildcard matches only the exact host address `1.1.1.1`.

![R1 adding the loopback address to OSPF Area 0](./Photos/Screenshot%202026-11-14%20205939.png)

**Verification**

The updated `show ip protocols` output displays `1.1.1.1 0.0.0.0 area 0` under Routing for Networks.

---

### 🔌 2. R2 — FastEthernet2/0 Administratively Down

**Symptom**

The R2–R3 link was not operational, preventing the expected connectivity and OSPF communication over that interface.

**Evidence and diagnosis**

I used:

```cisco
show ip interface brief
```

The output showed FastEthernet2/0 with the status **administratively down** and protocol **down**. This indicates that the interface had been manually disabled in the configuration.

![R2 interface status showing FastEthernet2/0 administratively down](./Photos/Screenshot%202026-11-14%20210332.png)

**Root cause**

FastEthernet2/0 on R2 was administratively shut down.

**Correction**

I entered the interface configuration and enabled it:

```cisco
interface FastEthernet2/0
 no shutdown
```

**Verification**

The interface status changed from administratively down/down to an operational state after enabling it. The interface and routing information were then checked again.

![R2 OSPF protocol and routing information after the interface correction](./Photos/Screenshot%202026-11-14%20210240.png)

---

### 🔗 3. R4 — OSPF Adjacency Prevented by a Passive Interface

**Symptom**

R4 was not forming the expected OSPF neighbor adjacency with R1 over GigabitEthernet0/0.

**Evidence and diagnosis**

Before changing the configuration, I checked both the OSPF process and its neighbor table:

```cisco
show ip protocols
show ip ospf neighbor
```

The `show ip protocols` output listed GigabitEthernet0/0 under **Passive Interface(s)**. Then, `show ip ospf neighbor` returned no neighbor entries. This was important: it showed that R4 had no established OSPF neighbor on that link at that point.

A passive OSPF interface can still have its connected network advertised, but it does not send OSPF Hello packets on that interface. Without Hello packets, R4 cannot discover and form an adjacency with R1 over GigabitEthernet0/0.

![R4 before troubleshooting: G0/0 is passive and the OSPF neighbor table is empty](./Photos/51212.png)

**Root cause**

GigabitEthernet0/0 on R4 was configured as a passive OSPF interface even though the link needed to establish an adjacency with R1.

**Correction**

I removed the passive-interface setting:

```cisco
router ospf 1
 no passive-interface GigabitEthernet0/0
```

![R4 removing the passive-interface setting](./Photos/Screenshot%202026-11-14%20210151.png)

**Verification**

After removing the passive setting, R4 generated this adjacency message:

```text
%OSPF-5-ADJCHG: Process 1, Nbr 1.1.1.1
on GigabitEthernet0/0 from LOADING to FULL
```

I then ran `show ip ospf neighbor` again. This time, the neighbor table displayed R1 (router ID `1.1.1.1`) in the **FULL/BDR** state on GigabitEthernet0/0.

This before-and-after comparison confirms the result: before the fix, the neighbor table was empty; after the fix, R4 successfully formed an OSPF adjacency with R1.

![R4 after troubleshooting: syslog confirms LOADING to FULL and show ip ospf neighbor displays R1 as FULL/BDR](./Photos/Screenshot%202026-11-14%20210158.png)

---

### ⚠️ 4. R3 — OSPF Area Mismatch on the R2–R3 Link

**Symptom**

The OSPF adjacency over the R2–R3 link was not forming correctly. R3 was receiving OSPF packets that did not match the area configured for the corresponding link.

**Evidence and diagnosis**

The syslog messages on R3 explicitly reported an area mismatch:

```text
%OSPF-4-ERRRCV: Received invalid packet:
mismatch area ID, from backbone area must be
virtual-link but not found from 10.23.0.3
```

This message was the key diagnostic evidence. It indicated that the incoming OSPF packet was associated with the backbone area while the local link's OSPF area configuration did not match.

![R3 syslog showing OSPF area mismatch messages](./Photos/Screenshot%202026-11-14%20210356.png)

**Root cause**

The `10.23.0.0/24` network on R3 was assigned to Area 2, while the R2–R3 link was configured for Area 0 on the other side. OSPF neighbors on a common link must agree on the area ID.

**Correction**

I removed the incorrect Area 2 network statement and assigned the network to Area 0:

```cisco
router ospf 1
 no network 10.23.0.0 0.0.0.255 area 2
 network 10.23.0.0 0.0.0.255 area 0
```

![R3 correcting the 10.23.0.0/24 network assignment to Area 0](./Photos/Screenshot%202026-11-14%20210623.png)

**Verification**

Immediately after the area correction, R3 generated an adjacency-change message:

```text
%OSPF-5-ADJCHG: Process 1, Nbr 2.2.2.2
on FastEthernet1/0 from LOADING to FULL
```

I also verified the result directly with:

```cisco
show ip ospf neighbor
```

The output listed neighbor `2.2.2.2` in the **FULL/BDR** state on FastEthernet1/0. The other listed neighbor, `5.5.5.5`, was also FULL/DR on FastEthernet3/0. The updated `show ip protocols` output showed `10.23.0.0 0.0.0.255 area 0`.

This is the key before-and-after proof: the syslog records the adjacency transition, and the neighbor command confirms that R3 now has a stable FULL adjacency with R2.

![R3 after troubleshooting: corrected area, adjacency syslog, and show ip ospf neighbor displaying R2 as FULL/BDR](./Photos/Screenshot%202026-11-14%20210623.png)

---

### 🧮 5. R3 — Summary Range Configured Under the Wrong OSPF Area

**Symptom**

R5 was not receiving the expected `10.0.0.0/8` inter-area summary route.

**Evidence and diagnosis**

Before changing the summary configuration, I checked the routing table on R5:

```cisco
show ip route
```

The routing table showed the component networks as separate OSPF inter-area routes. In particular, R5 had individual entries for:

- `10.12.0.0`
- `10.14.0.0`
- `10.23.0.0`

These were being learned separately instead of appearing as the required single `10.0.0.0/8` summary route.

![R5 before troubleshooting: routing table contains separate inter-area routes for the 10.x networks](./Photos/Screenshot%202026-11-14%20210647.png)

I then inspected the OSPF configuration on R3 and found that the summary range was configured under Area 2:

```cisco
area 2 range 10.0.0.0 255.0.0.0
```

The range needed to summarize routes belonging to Area 0 toward the other OSPF area. The area specified in the range command determines which area's routes are summarized by the ABR.

![R3 OSPF configuration showing the summary range under Area 2](./Photos/Screenshot%202026-11-14%20210703.png)

**Root cause**

The summary range was associated with the wrong area on R3.

**Correction**

I removed the incorrect Area 2 range and configured the range under Area 0:

```cisco
router ospf 1
 no area 2 range 10.0.0.0 255.0.0.0
 area 0 range 10.0.0.0 255.0.0.0
```

![R3 changing the summary range from Area 2 to Area 0](./Photos/Screenshot%202026-11-14%20210729.png)

**Verification on R5**

After correcting the range on R3, I ran `show ip route` on R5 again.

The routing table changed from listing the component networks separately to showing the summarized route:

```text
10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
O IA 10.0.0.0/8 [110/2000] via 10.35.0.3
```

- `O` indicates that the route was learned through OSPF.
- `IA` identifies it as an OSPF inter-area route.
- `10.0.0.0/8` is the summary prefix advertised to R5.
- `110` is the OSPF administrative distance.
- `2000` is the OSPF metric shown in this lab's routing table.
- `10.35.0.3` is the next hop toward the summarizing router.

![R5 after troubleshooting: the routing table now contains the 10.0.0.0/8 inter-area summary route](./Photos/Screenshot%202026-11-14%20210740.png)

**💡 Why Summarization Was Useful Here**

Without summarization, R5 had to maintain separate inter-area route entries for the individual 10.x networks. With the Area 0 range configured on the ABR, those component prefixes could be represented to the other area by one broader `10.0.0.0/8` summary.

In a larger multi-area network, this is useful because it:

1. **Reduces routing-table entries:** routers in the receiving area can use one summary instead of carrying every component prefix separately.
2. **Reduces inter-area routing information:** the ABR can advertise a summary Type 3 LSA rather than separate Type 3 LSAs for every covered component prefix.
3. **Improves scalability:** adding more subnets inside the summarized block does not necessarily require every other area to learn each individual prefix.
4. **Limits routing changes across areas:** changes to individual component routes can often remain within the source area instead of causing a separate inter-area advertisement for every subnet.

Summarization must be planned carefully. The component networks should fit the chosen summary range, and the network design must account for unused address space inside that range so traffic is not attracted toward a summary when no valid destination exists. In this lab, the purpose was to replace the separate 10.x inter-area entries visible on R5 with the required `10.0.0.0/8` summary route.

---

## ✅ Final Verification and Outcome

The troubleshooting exercise covered five configuration issues across R1, R2, R3 and R4, followed by route verification on R5.

| Router | Issue | Verification evidence |
|---|---|---|
| R1 | Loopback address missing from OSPF Area 0 | `show ip protocols` lists 1.1.1.1 in Area 0 |
| R2 | FastEthernet2/0 administratively down | Interface enabled with `no shutdown` |
| R4 | G0/0 configured as passive | OSPF syslog reports neighbor 1.1.1.1 reaching FULL |
| R3 | R2–R3 network assigned to the wrong area | Syslog identifies mismatch; neighbor 2.2.2.2 reaches FULL after correction |
| R3 | Summary range configured under Area 2 instead of Area 0 | R5 routing table shows O IA 10.0.0.0/8 |

## 🛠️ Key Troubleshooting Skills Demonstrated

- Interpreting OSPF syslog messages to identify adjacency and area mismatches.
- Checking interface states with `show ip interface brief`.
- Inspecting OSPF network statements and passive interfaces with `show ip protocols`.
- Understanding OSPF neighbor formation and the FULL adjacency state.
- Correcting OSPF area assignments on a shared link.
- Configuring inter-area route summarization on an ABR.
- Verifying learned routes and next hops using `show ip route`.

## 📸 Lab Evidence

All screenshots used in this walkthrough are stored in the [Photos](./Photos/) directory.

**Result:** The identified OSPF configuration issues were corrected, the expected neighbor adjacencies reached FULL state, and R5 learned the expected `10.0.0.0/8` inter-area summary route.
