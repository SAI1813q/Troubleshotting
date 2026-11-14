# OSPF Multi-Router Troubleshooting Lab

A hands-on Cisco Packet Tracer troubleshooting exercise involving a five-router OSPF network, serial and Ethernet links, internal LANs, and an ISP-facing connection.

The purpose of this lab was not simply to enter configuration commands. I worked through the faults by identifying the symptom, collecting evidence from the routers, isolating the root cause, applying the smallest relevant correction, and checking whether the expected behavior returned.

- **Platform:** Cisco Packet Tracer
- **Routing protocol:** OSPF
- **Routers:** R1–R5, plus the ISP-facing router shown in the topology
- **Evidence:** [All troubleshooting screenshots](./Photos/)

---

## Troubleshooting methodology

Each issue is documented using the same sequence:

1. **Symptom** — What was failing or behaving unexpectedly?
2. **Evidence** — Which commands or outputs helped narrow it down?
3. **Root cause** — What specific configuration or network condition caused the issue?
4. **Correction** — What was changed, and on which device/interface?
5. **Verification** — What output confirmed that the correction worked?

This distinction matters: a command being accepted is not proof that a fault is fixed. The verification step must show the expected network behavior.

---

## Initial configuration — R1–R2 serial link

This was part of the lab's initial configuration, **not a troubleshooting fault**. I identified the DCE end of the R1–R2 serial connection and configured the required clock rate on R1.

```cisco
show controllers serial 0/0/0

interface serial 0/0/0
 clock rate 128000
```

I then checked the interface status and serial details:

```cisco
show ip interface brief
show interfaces serial 0/0/0
```

This is recorded as a setup/configuration step and is not counted as one of the troubleshooting incidents below.

---

## Troubleshooting issue 1 — Missing OSPF route to 10.0.2.0/24

### Symptom

The expected route to **10.0.2.0/24** was not appearing in the routing information where it was needed. The investigation therefore focused on the OSPF link connecting the routers, rather than immediately adding a static route.

### Evidence

I compared the OSPF interface information at both ends of the R3–R4 connection. The outputs showed different OSPF network types:

| Device/interface | OSPF network type |
|---|---|
| R3 FastEthernet 2/0 | BROADCAST |
| R4 interface connected to R3 | POINT-TO-POINT |

The useful checks for this kind of fault are:

```cisco
show ip ospf interface
show ip ospf interface fastethernet 2/0
show ip ospf neighbor
show ip route
```

The key evidence was the mismatch between the two ends of the same link.

### Root cause

The OSPF network type on R3 FastEthernet 2/0 did not match the intended point-to-point network type already configured on the connected R4 interface.

OSPF network type affects how OSPF treats a link. A broadcast network supports DR/BDR election behavior, while a point-to-point network does not use a DR/BDR election on that link. The two ends should be configured consistently for the topology being built.

### Correction

I changed the OSPF network type on R3 FastEthernet 2/0 from BROADCAST to POINT-TO-POINT so that it matched the connected end:

```cisco
R3(config)# interface fastethernet 2/0
R3(config-if)# ip ospf network point-to-point
```

I did not change R4 for this correction because its connected interface was already using POINT-TO-POINT.

### Verification

After the change, the relevant checks were:

```cisco
show ip ospf interface fastethernet 2/0
show ip ospf neighbor
show ip route
```

I checked that R3 reported the intended point-to-point network type, that the OSPF relationship was established as expected, and that the missing network was being learned through routing.

**Troubleshooting lesson:** When a route is missing, inspect the OSPF relationship and interface characteristics on both ends of the path before adding routes manually. A routing-table symptom can originate from a link-level or OSPF-interface configuration issue.

---

## Troubleshooting issue 2 — OSPF neighbor adjacency failure caused by timer mismatch

### Symptom

R5 was not forming the expected OSPF neighbor adjacencies with the connected routers. Without a working adjacency, routers cannot exchange the link-state information needed to build a consistent OSPF database and calculate routes through that neighbor.

### Evidence

I checked the neighbor table and the OSPF interface settings:

```cisco
show ip ospf neighbor
show ip ospf interface
show ip protocols
```

The OSPF interface output showed a Hello/Dead timer mismatch:

| Router configuration | Hello interval | Dead interval |
|---|---:|---:|
| R5 (initial setting) | 5 seconds | 20 seconds |
| Neighboring routers | 10 seconds | 40 seconds |

OSPF neighbors on a link must agree on the Hello and Dead intervals. The different values were direct evidence of a configuration mismatch.

### Root cause

R5 was using Hello 5 / Dead 20, while the connected routers were using Hello 10 / Dead 40. Because the timer values did not match, the routers could not establish the expected adjacency.

### Correction

I corrected the affected OSPF interface timer configuration so that R5 and its neighbors used matching values: Hello 10 seconds and Dead 40 seconds.

The interface-level command format is:

```cisco
interface <connected-interface>
 ip ospf hello-interval 10
 ip ospf dead-interval 40
```

The settings must be consistent on both ends of each affected link.

### Verification

I checked the neighbor table again:

```cisco
show ip ospf neighbor
```

The affected adjacencies progressed to **FULL**. I also used the OSPF interface output to confirm the timer values:

```cisco
show ip ospf interface
```

**Why FULL matters:** FULL indicates that the neighbors have completed database synchronization for the adjacency. It is stronger evidence than merely seeing a router ID in a partial neighbor state.

---

## Troubleshooting issue 3 — Default route not available to the internal OSPF routers

### Symptom

Internal devices needed a path toward destinations outside the OSPF domain, including the ISP-facing side of the topology. An internal router needs a default route when no more-specific route exists for an external destination.

### Evidence

I inspected the routing table on the internal router and checked whether a default route was present:

```cisco
show ip route
```

I also inspected the OSPF external link-state information:

```cisco
show ip ospf database external
```

The later routing-table output on R3 showed an OSPF external default route:

```
O*E2 0.0.0.0/0
```

This indicates that R3 learned an OSPF external Type-5 default route.

### Root cause

The internal OSPF routers needed a default route originated from the edge router. R5 was configured to provide the upstream default route and advertise it into OSPF.

### Correction

On R5, I configured both parts:

**1. Configure a static default route toward the upstream next hop:**

```cisco
ip route 0.0.0.0 0.0.0.0 <upstream-next-hop>
```

**2. Originate the default route into OSPF:**

```cisco
router ospf 1
 default-information originate
```

The placeholder `<upstream-next-hop>` represents the actual next-hop address used in the lab; it should be replaced with that configured address when reproducing the setup.

### Verification

On an internal router such as R3, I checked:

```cisco
show ip route
show ip ospf database external
```

The `O*E2 0.0.0.0/0` entry confirms that the default route was learned through OSPF as an external route.

**Important distinction:** Seeing the default route in the routing table proves that OSPF distributed the route. It does not, by itself, prove that traffic can reach the internet. Full end-to-end verification also requires testing the path to the external destination and confirming that the upstream router has a valid route back to the internal networks.

---

## Additional verification — Inspecting the OSPF link-state database

### Objective

After restoring the OSPF behavior, I inspected the link-state database to understand what information the routers were exchanging and how the default route was represented.

### Evidence and commands

I used the following commands:

```cisco
show ip ospf database
show ip ospf database router
show ip ospf database network
show ip ospf database external
show ip route
```

### What I examined

- **Type 1 — Router LSA:** Describes a router's OSPF links and their characteristics within an area.
- **Type 2 — Network LSA:** Describes a multi-access network and is generated by the DR when applicable. It is not expected on a point-to-point link.
- **Type 5 — AS External LSA:** Advertises external routes into the OSPF domain. The externally originated default route is represented by a Type-5 LSA.

### Verification

I compared the LSDB information with the routing table. In particular, I looked for the external default route information and the corresponding OSPF-learned default route on the internal router.

This helped connect three pieces of evidence: the route configured at the edge, the external LSA distributed by OSPF, and the route installed in the downstream router's routing table.

---

## Summary — Troubleshooting fault isolation and verification

| Issue | Evidence that isolated the fault | Root cause | Correction | Verification |
|---|---|---|---|---|
| Missing 10.0.2.0/24 route | OSPF interface outputs showed BROADCAST on R3 and POINT-TO-POINT on the connected R4 end | OSPF network-type mismatch | Set R3 F2/0 to POINT-TO-POINT | OSPF interface type, neighbor state, routing table |
| R5 neighbor adjacency | Hello/Dead values differed: 5/20 versus 10/40 | OSPF timer mismatch | Make timer values 10/40 on both ends | Neighbor state reached FULL |
| Default route missing internally | Routing table and external LSDB inspection | Edge default route needed to be advertised into OSPF | Static default route plus `default-information originate` on R5 | `O*E2 0.0.0.0/0` learned on R3 |
| Understanding OSPF advertisements | Router, Network, and External LSA database outputs | Needed to inspect what OSPF was distributing | No configuration change; performed LSDB inspection | Compared LSDB entries with routing-table entries |

---

## Skills practised

- Initial serial DCE/DTE identification and clock-rate configuration
- Comparing interface and OSPF settings across both ends of a link
- Troubleshooting OSPF network-type mismatches
- Diagnosing Hello/Dead timer mismatches
- Reading OSPF neighbor states and verifying FULL adjacency
- Configuring a static default route and originating it into OSPF
- Interpreting OSPF Type 1, Type 2, and Type 5 LSAs
- Correlating the OSPF LSDB with installed routing-table entries
- Following a repeatable symptom → evidence → root cause → correction → verification workflow

## Evidence screenshots

The screenshots are listed in capture-time order. They provide the original topology, diagnostic command outputs, configuration changes, and post-change verification.

### 01. Troubleshooting evidence
![Evidence 1](./Photos/Screenshot%202026-11-14%20170829.png)

### 02. Troubleshooting evidence
![Evidence 2](./Photos/Screenshot%202026-11-14%20171141.png)

### 03. Troubleshooting evidence
![Evidence 3](./Photos/Screenshot%202026-11-14%20171241.png)

### 04. Troubleshooting evidence
![Evidence 4](./Photos/Screenshot%202026-11-14%20171713.png)

### 05. Troubleshooting evidence
![Evidence 5](./Photos/Screenshot%202026-11-14%20172335.png)

### 06. Troubleshooting evidence
![Evidence 6](./Photos/Screenshot%202026-11-14%20172354.png)

### 07. Troubleshooting evidence
![Evidence 7](./Photos/Screenshot%202026-11-14%20172504.png)

### 08. Troubleshooting evidence
![Evidence 8](./Photos/Screenshot%202026-11-14%20172513.png)

### 09. Troubleshooting evidence
![Evidence 9](./Photos/Screenshot%202026-11-14%20172956.png)

### 10. Troubleshooting evidence
![Evidence 10](./Photos/Screenshot%202026-11-14%20173018.png)

### 11. Troubleshooting evidence
![Evidence 11](./Photos/Screenshot%202026-11-14%20173112.png)

### 12. Troubleshooting evidence
![Evidence 12](./Photos/Screenshot%202026-11-14%20173333.png)

### 13. Troubleshooting evidence
![Evidence 13](./Photos/Screenshot%202026-11-14%20173456.png)

### 14. Troubleshooting evidence
![Evidence 14](./Photos/Screenshot%202026-11-14%20173537.png)

### 15. Troubleshooting evidence
![Evidence 15](./Photos/Screenshot%202026-11-14%20173711.png)

### 16. Troubleshooting evidence
![Evidence 16](./Photos/Screenshot%202026-11-14%20174159.png)

### 17. Troubleshooting evidence
![Evidence 17](./Photos/Screenshot%202026-11-14%20174314.png)

### 18. Troubleshooting evidence
![Evidence 18](./Photos/Screenshot%202026-11-14%20174448.png)

### 19. Troubleshooting evidence
![Evidence 19](./Photos/Screenshot%202026-11-14%20174522.png)

### 20. Troubleshooting evidence
![Evidence 20](./Photos/Screenshot%202026-11-14%20180059.png)

### 21. Troubleshooting evidence
![Evidence 21](./Photos/Screenshot%202026-11-14%20181150.png)

## Key takeaway

The R1–R2 serial clock-rate work was an initial configuration task, not a fault investigation. The actual troubleshooting focused on the missing OSPF route, the neighbor timer mismatch, and default-route distribution. I used interface and OSPF outputs to isolate the causes, applied targeted corrections, and verified the results through neighbor state, routing-table entries, and LSDB contents.
