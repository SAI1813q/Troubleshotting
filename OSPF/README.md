# OSPF Multi-Router Troubleshooting Lab

A hands-on Cisco Packet Tracer exercise focused on diagnosing routing and connectivity faults in a multi-router network.

- **Platform:** Cisco Packet Tracer
- **Routing protocol:** OSPF
- **Topology:** Five internal routers (R1–R5), end-device LANs, serial and Ethernet links, and an ISP-facing connection.
- **Evidence:** [Troubleshooting screenshots](./Photos/)

## Troubleshooting process

### 1. Serial link and DCE clocking

Identified the DCE end of the R1–R2 serial link with `show controllers serial`. Configured the required clock rate of `128000` on R1's DCE interface, then checked interface status and addressing.

```cisco
show controllers serial 0/0/0
show ip interface brief
show interfaces serial 0/0/0
```

### 2. Missing 10.0.2.0/24 route

Investigated why the expected `10.0.2.0/24` route was not appearing. Comparing the OSPF interface settings on the connected link revealed a network-type mismatch:

- R3 FastEthernet 2/0: **BROADCAST**
- Connected interface on the other router: **POINT-TO-POINT**

Changed the OSPF network type on R3 FastEthernet 2/0 to match the other end:

```cisco
interface fastethernet 2/0
 ip ospf network point-to-point
```

Rechecked OSPF and the routing table after the correction.

### 3. OSPF neighbor adjacency failure

Checked neighbor states and compared Hello/Dead intervals. R5 was configured for Hello 5 / Dead 20, while the neighboring routers used Hello 10 / Dead 40. Corrected the affected timer settings so both ends matched, then verified that the adjacencies reached **FULL**.

```cisco
show ip ospf neighbor
show ip ospf interface
show ip protocols
```

### 4. Default route and external route advertisement

On R5, configured a static default route toward the upstream next hop and originated it into OSPF:

```cisco
ip route 0.0.0.0 0.0.0.0 <next-hop>
router ospf 1
 default-information originate
```

Verified on a downstream router that the default route was learned as an OSPF external route (for example, `O*E2 0.0.0.0/0`). Learning the route confirms OSPF advertisement; end-to-end reachability must still be tested separately.

### 5. OSPF database inspection

Inspected the link-state database and routing table to understand the information propagated through the OSPF domain.

```cisco
show ip ospf database
show ip ospf database router
show ip ospf database network
show ip ospf database external
show ip route
```

Reviewed Router LSAs (Type 1), Network LSAs (Type 2 where applicable), and AS External LSAs (Type 5) associated with the externally originated default route.

## Skills practised

- Serial DCE/DTE identification and clock-rate configuration
- Interface status and IP connectivity checks
- OSPF neighbor formation and adjacency states
- Hello/Dead interval troubleshooting
- OSPF interface network types
- Routing-table and LSDB interpretation
- Static default routing and OSPF default-route origination
- Layered troubleshooting from interface state through routing and reachability verification

## Evidence screenshots

Screenshots are listed in capture-time order to show the troubleshooting progression.

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

This lab reinforced checking both ends of a link, comparing OSPF parameters instead of guessing, and verifying changes through neighbor state, routing-table entries, and LSDB contents. The next step is to practise multi-fault scenarios and document each one using **symptom → evidence → root cause → fix → verification**.
