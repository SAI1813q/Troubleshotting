# 🔧 RIP, DHCP, PAT, DNS & SSH Troubleshooting

Hands-on Cisco Packet Tracer troubleshooting covering five common network services and access issues: RIP, DHCP relay, PAT, DNS configuration, and SSH.

This exercise contains five faults. Each one was investigated by checking router or PC output, identifying the configuration issue, applying a correction, and testing again.

**Platform:** Cisco Packet Tracer  


## 🌐 Lab overview

The scenarios cover RIP route advertisement, DHCP relay, PAT, DHCP-provided DNS information, and SSH access.

![Lab topology](./Photos/Screenshot%202026-11-16%20131002.png)

---

## 1. 🧭 RIP – Missing routes on R2 and R3

### Symptom
R2 and R3 were not receiving the expected RIP routes from R1.

### 🔎 Investigation
I first checked R2 and R3 because both routers were missing routes. I then traced the issue back to R1 and inspected its RIP configuration. The problem was a passive interface on R1.

**Diagnostic commands**
```cisco
show ip route
show ip protocols
show running-config | section router rip
```

A passive RIP interface can still receive routing updates, but it does not send RIP updates out of that interface. This explains why R1 could continue learning routes while not advertising routes to its neighbor over the passive interface.

**Evidence**

![R2 – initial routing-table check](./Photos/Screenshot%202026-11-16%20130931.png)

![R3 – initial routing-table check](./Photos/Screenshot%202026-11-16%20130955.png)

![R1 – investigation](./Photos/Screenshot%202026-11-16%20131219.png)

![R1 – RIP configuration and passive-interface finding](./Photos/Screenshot%202026-11-16%20131250.png)

### Root cause
The relevant R1 interface was configured as passive under RIP, preventing R1 from advertising RIP updates through that interface.

### 🛠️ Correction
Removed the passive-interface setting from the R1 inter-router interface:

```cisco
router rip
 no passive-interface <R1-inter-router-interface>
```

Use the exact interface shown in the R1 configuration screenshot; I have kept the interface as a placeholder here rather than guessing it.

### ✅ Verification
Checked the routing information again after the correction to confirm that the missing RIP routes were being learned.

```cisco
show ip route
show ip protocols
```


![Post-troubleshooting RIP evidence](./Photos/Screenshot%202026-11-16%20131305.png)

---

## 2. 📡 DHCP – Hosts not receiving IP addresses

### Symptom
The hosts behind R2 were not receiving DHCP addresses.

### 🔎 Investigation
I checked the PC before and after attempting DHCP. Then I inspected R2's G0/0 interface, which acts as the gateway interface for that LAN. The interface did not have an IP helper address configured.

**Diagnostic commands**
```text
ipconfig
ipconfig /all
```

```cisco
show ip interface g0/0
show running-config interface g0/0
```

**Evidence**

![PC – before troubleshooting](./Photos/Screenshot%202026-11-16%20131504.png)

![R2 G0/0 – helper-address check](./Photos/Screenshot%202026-11-16%20131731.png)

### Root cause
R2 G0/0 was missing the `ip helper-address` configuration. DHCP client discovery messages are broadcasts and need a relay on the gateway interface to reach a DHCP server on another network.

### 🛠️ Correction
Configured the DHCP relay address on R2 G0/0:

```cisco
interface g0/0
 ip helper-address <DHCP-server-IP>
```

The helper-address IP is left as a placeholder rather than inventing an address; use the exact value visible in the configuration screenshot.


![R2 – helper-address configuration](./Photos/Screenshot%202026-11-16%20132020.png)

### ✅ Verification
Retested DHCP from the PC. The host successfully obtained an IP configuration after the helper address was added.

```text
ipconfig /release
ipconfig /renew
ipconfig /all
```


![PC – successful DHCP result](./Photos/Screenshot%202026-11-16%20132049.png)

![Additional DHCP verification](./Photos/Screenshot%202026-11-16%20132157.png)

---

## 3. 🔁 PAT – Incorrect ACL used by the NAT rule

### Symptom
PAT was configured but translations were not appearing for the inside traffic.

### 🔎 Investigation
I checked the NAT statistics and inspected the PAT configuration. The PAT rule was using ACL 2, which was the incorrect ACL reference for the intended inside traffic.

**Diagnostic commands**
```cisco
show ip nat statistics
show ip nat translations
show running-config | include ip nat
show access-lists
```

**Evidence**

![R1 – NAT statistics before generating test traffic](./Photos/Screenshot%202026-11-16%20132233.png)

![R1 – PAT and ACL configuration check](./Photos/Screenshot%202026-11-16%20132303.png)

### Root cause
The PAT overload rule referenced ACL 2 instead of the ACL intended to match the inside network.

### 🛠️ Correction
Corrected the ACL reference in the PAT overload rule. The rule was changed from ACL 2 to ACL 1:

```cisco
no ip nat inside source list 2 interface g0/1 overload
ip nat inside source list 1 interface g0/1 overload
```

The ACL reference and interface should match the configuration visible in the R1 screenshot.


![R1 – corrected PAT configuration](./Photos/Screenshot%202026-11-16%20133354.png)

### Verification
After changing the PAT rule, I checked again and initially no translations were displayed for the `192.168.2.0` traffic. I then pinged `30.0.0.100` from the PC to generate traffic through the router. After generating traffic, I checked the NAT translations again and the translations appeared.


![PC – ping to 30.0.0.100](./Photos/Screenshot%202026-11-16%20133335.png)

![R1 – translations after the ping generated traffic](./Photos/Screenshot%202026-11-16%20133533.png)

**Troubleshooting takeaway:** A dynamic PAT translation is created when matching traffic is generated. An empty translation table before the test traffic does not, by itself, mean the corrected PAT rule is failing.

**Verification commands**
```cisco
show ip nat translations
show ip nat statistics
```

```text
ping 30.0.0.100
```

---

## 4. 🧩 DNS – DHCP clients not receiving a DNS server

### Symptom
The PC did not have a DNS server address in its IP configuration, so hostname resolution was not working.

### 🔎 Investigation
I checked `ipconfig /all` on the PC and confirmed that the DNS server field was missing. I also tested DNS by trying to ping the server by its hostname.

```text
ipconfig /all
ping srv1
```

**Evidence**

![R1 – DHCP configuration before correction](./Photos/Screenshot%202026-11-16%20133840.png)

![PC – ipconfig /all showing no DNS server](./Photos/Screenshot%202026-11-16%20134129.png)

### Root cause
The DHCP pool on R1 did not include the DNS server option.

### 🛠️ Correction
Added the DNS server address to the relevant DHCP pool on R1:

```cisco
ip dhcp pool <R1-DHCP-pool-name>
 dns-server 30.0.0.100
```

The pool name is left as a placeholder because it should be copied exactly from the R1 configuration screenshot.


![R1 – adding the DNS server option](./Photos/Screenshot%202026-11-16%20133914.png)

### ✅ Verification
Released and renewed the PC's DHCP lease so it could receive the updated DHCP options. Then I checked the configuration and pinged `srv1` by hostname successfully.

```text
ipconfig /release
ipconfig /renew
ipconfig /all
ping srv1
```


![PC – release/renew and hostname ping to srv1](./Photos/Screenshot%202026-11-16%20134156.png)

---

## 5. 🔐 SSH – VTY transport permitted Telnet instead of SSH

### Symptom
An SSH login attempt from PC1 to R1 was rejected.

### 🔎 Investigation
I attempted to connect to R1 from PC1 and observed that the connection failed. I then checked the VTY configuration on R1 and found that the transport input setting permitted Telnet.

**Diagnostic commands**
```cisco
show running-config | section line vty
```

**Evidence**

![PC1 – initial SSH attempt rejected](./Photos/Screenshot%202026-11-16%20134254.png)

![R1 – VTY transport configuration](./Photos/Screenshot%202026-11-16%20134315.png)

### Root cause
The VTY lines were configured with Telnet as the permitted transport, rather than SSH.

### 🛠️ Correction
Changed the VTY transport setting to permit SSH:

```cisco
line vty <VTY-line-range>
 transport input ssh
```

The exact VTY line range is left as a placeholder; use the range shown in the R1 configuration screenshot.


![R1 – corrected VTY transport setting](./Photos/Screenshot%202026-11-16%20134400.png)

### ✅ Verification
Retried the SSH connection from PC1. This time the login succeeded and I reached the R1 command-line interface.

```text
ssh -l <username> <R1-management-IP>
```


![PC1 – successful SSH login to R1](./Photos/Screenshot%202026-11-16%20134433.png)

---

## 🧠 Skills practised

- Diagnosing RIP route-advertisement problems and passive-interface behavior
- Troubleshooting DHCP relay using PC lease information and router interface checks
- Inspecting PAT rules, ACL references, and dynamic NAT translations
- Configuring DHCP DNS options and validating client DNS settings
- Troubleshooting VTY access and verifying SSH connectivity from an end host
- Following a structured **symptom → investigation → root cause → correction → verification** workflow

