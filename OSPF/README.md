# OSPF Multi-Router Troubleshooting Lab

A Cisco Packet Tracer troubleshooting lab involving five internal routers (R1–R5), two user LANs, an Ethernet transit network, a shared router segment, serial connectivity, and an ISP-facing link.

This write-up separates the lab's initial configuration from the faults that were actually investigated. The troubleshooting sections use **symptom → evidence → root cause → correction → verification**, with cropped CLI screenshots placed beside the explanation so the reader can follow the investigation and see the router-generated syslog messages.

- **Platform:** Cisco Packet Tracer
- **Routing protocol:** OSPF, Process ID 1, Area 0
- **Main transit networks:** 192.168.34.0/30 and 192.168.245.0/29
- **External test destination:** 8.8.8.8
- **Evidence:** Cropped, relevant screenshots are embedded below and stored in [Photos](./Photos/).

---

## 1. Lab topology and initial setup

![Lab topology and task description](./Photos/Screenshot%202026-11-14%20170829.png)

The supplied topology already contained IP addressing and OSPF configuration. The first task was to add the R1–R2 serial connection, configure its serial interfaces, and establish OSPF across the new link. This was an initial setup task, not a fault found during troubleshooting.

### R1–R2 serial setup

![R1 serial interface and DCE controller output](./Photos/Screenshot%202026-11-14%20171141.png)

The controller output identified R1 Serial0/0/0 as the DCE end. I configured the requested clock rate of 128000 on that DCE interface, enabled the interface, and configured the serial IP address.

```cisco
show controllers serial 0/0/0

interface serial 0/0/0
 no shutdown
 ip address 192.168.12.1 255.255.255.252
 clock rate 128000
```

The R2 screenshot shows the serial interface configured with 192.168.12.2/30 and an OSPF adjacency message reaching FULL. This confirms the newly added serial connection was brought into OSPF.

![R2 serial interface and OSPF neighbor reaching FULL](./Photos/Screenshot%202026-11-14%20171713.png)

---

# 2. Troubleshooting issue 1 — The 10.0.2.0/24 LAN was not being learned by the other routers

## Symptom

The lab stated that only R3 had a route to **10.0.2.0/24**. R3 is directly connected to this LAN, so it naturally has a connected route. The problem was that the other routers were not learning the LAN route through OSPF as expected.

The relevant path is R3–R4 over the **192.168.34.0/30** transit link. R3 connects to the LAN 10.0.2.0/24 on G0/0 and to R4 on G0/1.

## Evidence — Compare the OSPF network type on both ends

The R3 output shows that its G0/1 interface, addressed as 192.168.34.1/30, was already operating as **POINT-TO-POINT**. It also shows the R4 neighbor (router ID 192.168.245.2) in FULL state.

![R3 G0/1 showing POINT-TO-POINT and a FULL neighbor](./Photos/Screenshot%202026-11-14%20173456.png)

The R4 output for G0/1, addressed as 192.168.34.2/30, showed **Network Type BROADCAST**. This was inconsistent with the point-to-point setting on R3's connected interface.

![R4 G0/1 showing BROADCAST before correction](./Photos/Screenshot%202026-11-14%20173537.png)

The useful commands for comparing the two ends are:

```cisco
show ip ospf interface gigabitEthernet 0/1
show ip ospf neighbor
show ip route
```

## Root cause

The two ends of the R3–R4 transit link were configured with different OSPF network types:

| Router | Interface | IP address | OSPF network type before correction |
|---|---|---|---|
| R3 | G0/1 | 192.168.34.1/30 | POINT-TO-POINT |
| R4 | G0/1 | 192.168.34.2/30 | BROADCAST |

A broadcast OSPF network uses DR/BDR election behavior. A point-to-point OSPF network does not elect a DR or BDR on that link. The transit link was intended to operate as point-to-point, so the network type needed to be consistent at both ends.

The LAN route itself was configured for OSPF advertisement on R3. The R3 routing-protocol output shows the 10.0.2.254 host address included in OSPF Area 0 and G0/0 listed as a passive interface. Making the transit adjacency consistent allowed the remote routers to receive the route advertisement.

## Correction

R3 G0/1 was already set to point-to-point. I changed the connected R4 G0/1 interface from BROADCAST to POINT-TO-POINT:

```cisco
R4(config)# interface gigabitEthernet 0/1
R4(config-if)# ip ospf network point-to-point
```

The cropped R4 CLI screenshot captures the command being entered and the immediate OSPF state change.

## Verification — Router-generated syslog proves the adjacency recovered

Immediately after the network-type change, the R4 console displayed these OSPF adjacency transitions:

```
%OSPF-5-ADJCHG: ... from FULL to DOWN
%OSPF-5-ADJCHG: ... from LOADING to FULL, Loading Done
```

This sequence is important:

1. **FULL to DOWN:** OSPF reset the existing neighbor relationship when the interface's OSPF network type changed. A brief adjacency reset is expected when changing a live OSPF interface setting.
2. **LOADING to FULL:** The routers re-established the adjacency and completed link-state database synchronization.
3. **Loading Done:** The router reported that database loading had completed for that adjacency.

![R4 network-type correction and OSPF syslog transition from FULL to DOWN, then LOADING to FULL](./Photos/Screenshot%202026-11-14%20173537.png)

The R3 output after the change confirms that G0/1 is point-to-point and the neighbor is FULL.

![R3 confirms point-to-point interface and FULL adjacency](./Photos/Screenshot%202026-11-14%20173456.png)

The route table on R5 later shows **10.0.2.0/24** as an OSPF-learned route via 192.168.245.2 (R4). This is stronger proof than checking only R3, because R3's own route to that LAN is directly connected.

![R5 routing table showing 10.0.2.0/24 learned through OSPF](./Photos/Screenshot%202026-11-14%20174159.png)

**Result:** The R3–R4 OSPF link used a consistent point-to-point network type, the adjacency returned to FULL, and a remote router learned the 10.0.2.0/24 route through OSPF.

---

# 3. Troubleshooting issue 2 — R5 was not forming OSPF adjacencies with R2 and R4

## Symptom

R5 was not forming the expected OSPF neighbor adjacencies on its shared Ethernet segment, 192.168.245.0/29. Without these adjacencies, R5 could not exchange the required link-state information with the connected routers.

## Evidence — Inspect R5's OSPF interface timers

I checked the OSPF interface configuration on R5:

```cisco
show ip ospf interface gigabitEthernet 0/0
show ip ospf neighbor
```

The R5 G0/0 output showed:

- Interface IP: 192.168.245.3/29
- OSPF network type: BROADCAST
- Hello interval: 5 seconds
- Dead interval: 20 seconds
- Neighbor count: 0

![R5 before correction showing Hello 5, Dead 20 and no OSPF neighbors](./Photos/Screenshot%202026-11-14%20172354.png)

The connected R4 interface output showed Hello 10 and Dead 40. That gave us a direct comparison between the settings on the shared segment.

![R4 interface showing Hello 10 and Dead 40](./Photos/Screenshot%202026-11-14%20172513.png)

## Root cause

The OSPF Hello and Dead intervals did not match:

| Router/interface | Hello interval | Dead interval |
|---|---:|---:|
| R5 G0/0 — before correction | 5 seconds | 20 seconds |
| Neighboring OSPF interfaces | 10 seconds | 40 seconds |

OSPF routers on the same link must agree on key Hello parameters, including the Hello and Dead intervals. R5 was sending Hellos using 5/20 while its neighbors expected 10/40. This prevented the expected adjacencies from forming.

## Correction

On R5 G0/0, I changed the Hello and Dead intervals to match the neighboring routers:

```cisco
interface gigabitEthernet 0/0
 ip ospf hello-interval 10
 ip ospf dead-interval 40
```

## Verification — Syslog shows both neighbors completing database loading

Immediately after the timer correction, R5 displayed OSPF adjacency-change messages for both neighbors:

```
%OSPF-5-ADJCHG: ... Nbr 192.168.245.1 ... from LOADING to FULL, Loading Done
%OSPF-5-ADJCHG: ... Nbr 192.168.245.2 ... from LOADING to FULL, Loading Done
```

This is direct device-generated evidence that the two neighbor relationships reached FULL after the timer correction. The **LOADING to FULL** transition means the routers completed OSPF database synchronization for those adjacencies.

![R5 timer correction and syslog showing both neighbors transition from LOADING to FULL](./Photos/Screenshot%202026-11-14%20172513.png)

I then checked the interface and neighbor table again. The output showed Hello 10 / Dead 40, a neighbor count of 2, and both neighbors in FULL state (one shown as BDR and the other as DROTHER).

![R5 post-correction timer values and both OSPF neighbors in FULL](./Photos/Screenshot%202026-11-14%20172956.png)

**Result:** R5's timers matched the other routers, both expected OSPF adjacencies formed, and the router confirmed successful database loading through its syslog messages.

---

# 4. Troubleshooting issue 3 — Internal PCs could not reach the external destination

## Symptom

PC1 and PC2 could not initially reach the external test destination **8.8.8.8**. PC2 returned a destination-host-unreachable message from its default gateway, 10.0.2.254.

![PC2 initial ping failure and unsuccessful traceroute](./Photos/Screenshot%202026-11-14%20173711.png)

## Evidence

The initial test showed that PC2 could not complete the ping to 8.8.8.8. The traceroute repeatedly reached or reported the local gateway but did not complete the path to the destination.

I checked the routing information and OSPF external route information on the routers. The internal routers needed a default route pointing toward the ISP-facing edge.

## Root cause

The internal OSPF routers needed a default route for destinations that were not present as more-specific routes in their routing tables. R5 was the edge router connected toward the ISP, so it needed a static default route toward the upstream next hop and had to originate that route into OSPF.

## Correction

On R5, I configured the static default route and advertised it through OSPF:

```cisco
ip route 0.0.0.0 0.0.0.0 <upstream-next-hop>

router ospf 1
 default-information originate
```

The placeholder represents the upstream next-hop address configured in the lab.

## Verification — Confirm the learned default route and test end-to-end reachability

On R3, the routing table showed:

```
Gateway of last resort is 192.168.34.2 to network 0.0.0.0
O*E2 0.0.0.0/0 via 192.168.34.2
```

This confirms that R3 learned an OSPF external Type-5 default route through R4.

![R3 routing table showing the OSPF external default route](./Photos/Screenshot%202026-11-14%20174448.png)

The final PC2 test shows successful replies from 8.8.8.8 and a completed traceroute. The visible path is:

1. 10.0.2.254 — PC2's default gateway / R3
2. 192.168.34.2 — R4
3. 192.168.245.3 — R5
4. 8.8.8.8 — external destination

![PC2 successful ping and completed traceroute to 8.8.8.8](./Photos/Screenshot%202026-11-14%20174522.png)

**Result:** The default route was distributed through OSPF, and PC2 successfully reached the external destination. The traceroute also showed the traffic traversing the expected internal routers and the edge router.

---

# 5. Additional verification — Inspecting the OSPF link-state database

After restoring routing, I inspected the OSPF database to see the link-state information maintained by R3.

Commands used:

```cisco
show ip ospf database
show ip ospf database router
show ip ospf database network
show ip ospf database external
show ip route
```

The R3 output shows:

- **Router Link States (Type 1):** Router LSAs for routers participating in Area 0.
- **Net Link States (Type 2):** A Network LSA for the shared multi-access segment, originated by its DR.
- **Type-5 AS External Link States:** An external LSA for 0.0.0.0, representing the default route originated into OSPF.

The same screenshot also contains OSPF syslog messages showing the R3–R4 neighbor reaching FULL, followed by the LSDB output.

![R3 OSPF adjacency syslog and OSPF database showing Type 1, Type 2 and Type 5 LSAs](./Photos/Screenshot%202026-11-14%20180059.png)

This links the control-plane evidence to the forwarding result: OSPF adjacencies formed, the external default route was represented in the LSDB, and the internal routing table installed the default route.

---

## 6. Troubleshooting summary

| Fault / task | Evidence | Root cause | Correction | Verification |
|---|---|---|---|---|
| R3 LAN 10.0.2.0/24 not learned remotely | R3 G0/1 was POINT-TO-POINT; R4 G0/1 was BROADCAST | OSPF network-type mismatch on R3–R4 transit link | Set R4 G0/1 to POINT-TO-POINT to match R3 | R4 syslog FULL → DOWN → LOADING → FULL; R5 learned 10.0.2.0/24 |
| R5 adjacencies missing | R5 G0/0 showed Hello 5 / Dead 20 and zero neighbors; peers used 10/40 | OSPF timer mismatch | Set R5 G0/0 Hello 10 / Dead 40 | Syslog showed both neighbors LOADING → FULL; neighbor table showed two FULL peers |
| PCs could not reach 8.8.8.8 | Initial ping/traceroute failed; later R3 showed O*E2 default route | No usable default route distributed to internal routers | Configure static default route on R5 and originate it into OSPF | R3 learned O*E2 0.0.0.0/0; PC2 ping and traceroute succeeded |
| OSPF database inspection | R3 LSDB output | Verification task, not a fault | No configuration change | Type 1, Type 2 and Type 5 LSAs visible |

---

## 7. What this lab demonstrates

- Comparing OSPF interface settings on both ends of a transit link
- Identifying and correcting OSPF network-type mismatches
- Reading OSPF adjacency syslog messages and understanding FULL, DOWN, and LOADING transitions
- Diagnosing Hello/Dead interval mismatches
- Verifying multiple OSPF neighbors after a timer correction
- Advertising an edge default route into OSPF
- Confirming route learning with the routing table and LSDB
- Using ping and traceroute to verify actual end-to-end reachability

The most important lesson was to use device output as evidence. The syslog messages are especially useful because they record the adjacency transition at the moment the configuration changes. I did not treat a successful command entry as proof; I checked the neighbor state, routing table, LSDB, and final packet path.
