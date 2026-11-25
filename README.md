<div align="center">

# 🔄 Dynamic NAT: Pool + Access List

### Translating a Whole Private LAN on a Cisco Router · Packet Tracer

![Cisco](https://img.shields.io/badge/Cisco-IOS-1BA0D7?style=for-the-badge&logo=cisco&logoColor=white)
![Packet Tracer](https://img.shields.io/badge/Packet%20Tracer-Lab-049fd9?style=for-the-badge)
![NAT](https://img.shields.io/badge/NAT-Dynamic-red?style=for-the-badge)
![Pool](https://img.shields.io/badge/Pool-10%20addresses-orange?style=for-the-badge)
![ACL](https://img.shields.io/badge/ACL-Selects%20hosts-purple?style=for-the-badge)
![Level](https://img.shields.io/badge/Level-CCNA-green?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Verified-success?style=for-the-badge)

*A pool of public addresses, lent out on demand and returned when the traffic stops.*

</div>

---

## 📑 Table of Contents
1. [Overview](#-overview)
2. [Topology](#-topology)
3. [Addressing Table](#-addressing-table)
4. [Static vs Dynamic vs PAT](#-static-vs-dynamic-vs-pat)
5. [The Three Building Blocks](#-the-three-building-blocks)
6. [Configuration](#-configuration)
7. [Verification: Stage by Stage](#-verification-stage-by-stage)
8. [Reading the Debug Output](#-reading-the-debug-output)
9. [Address Assignment Across Test Runs](#-address-assignment-across-test-runs)
10. [Pool Size Matters](#-pool-size-matters)
11. [Key Takeaways](#-key-takeaways)
12. [Troubleshooting Notes](#-troubleshooting-notes)
13. [Next Labs](#-next-labs)
14. [Author](#-author)

---

## 🎯 Overview

In the previous lab, **Static NAT** mapped one private host to one permanent public address. That does not scale to a whole LAN.

**Dynamic NAT** solves this with a **pool** of public addresses. When an inside host sends traffic, the router borrows a free address from the pool for that host. When the traffic stops, the entry expires and the address **returns to the pool**.

| Item | Detail |
|---|---|
| NAT type | **Dynamic NAT** (pool + access list) |
| NAT router | Router0, hostname `NAT-ROUTER` |
| Inside interface | GigabitEthernet0/0 (`ip nat inside`) |
| Outside interface | Serial0/0/0 (`ip nat outside`) |
| Pool name | `test` |
| Pool range | 1.1.1.11 to 1.1.1.20 (**10 addresses**) |
| Hosts allowed | 192.168.1.0/24, selected by ACL 1 |
| ISP router | Router1, 1.1.1.2 |

---

## 🗺 Topology

```
   ┌────────────── Private AREA ──────────────┐        ┌─────── ISP ───────┐
   │                                          │        │                   │
   │  PC0 192.168.1.4  PC1 192.168.1.3  PC2 192.168.1.2│                   │
   │          \             |            /    │        │                   │
   │            ┌────────────────────┐        │        │                   │
   │            │ Switch0 (2950-24)  │        │        │                   │
   │            └─────────┬──────────┘        │        │                   │
   │                      │                   │        │                   │
   │              Gi0/0 192.168.1.1           │        │                   │
   │              ip nat INSIDE               │  WAN AREA                  │
   │                ┌─────┴──────┐  Se0/0/0 1.1.1.1 ━━━━⚡━━━ 1.1.1.2  ┌────────┐
   │                │  Router0   │  ip nat OUTSIDE   │        │        │Router1 │
   │                │ NAT-ROUTER │                   │        │        │  ISP   │
   │                └────────────┘                   │        │        └────────┘
   └──────────────────────────────────────────┘        └───────────────────┘

   Translation: 192.168.1.x  ◄══ dynamic ══►  pool 1.1.1.11 - 1.1.1.20
```

---

## 📋 Addressing Table

| Device | Interface | IP Address | Role |
|---|---|---|---|
| Router0 (NAT-ROUTER) | GigabitEthernet0/0 | 192.168.1.1/24 | Private gateway, **NAT inside** |
| Router0 (NAT-ROUTER) | Serial0/0/0 | 1.1.1.1 | WAN link, **NAT outside** |
| Router1 (ISP) | Serial | 1.1.1.2 | ISP side of the WAN link |
| PC0 | NIC | 192.168.1.4/24 | Inside host |
| PC1 | NIC | 192.168.1.3/24 | Inside host |
| PC2 | NIC | 192.168.1.2/24 | Inside host |
| **NAT pool `test`** | n/a | **1.1.1.11 to 1.1.1.20** | Public addresses handed out dynamically |

---

## 📚 Static vs Dynamic vs PAT

| | Static NAT | **Dynamic NAT (this lab)** | PAT (overload) |
|---|---|---|---|
| Mapping | 1 private ↔ 1 public, **fixed** | 1 private → 1 public from a **pool**, temporary | Many private → **1** public, by port |
| Created by | The configuration command | **Traffic** | Traffic |
| Expires? | ❌ Never | ✅ Yes, when idle | ✅ Yes |
| Same address each time? | ✅ Always | ❌ Not guaranteed | n/a |
| Limit | One public per host | **Pool size** (10 here) | Ports on one address |
| Typical use | Servers | Small number of hosts, public address pool | Normal Internet access |

---

## 🧱 The Three Building Blocks

Dynamic NAT needs three separate pieces, plus the inside/outside interface markings.

| # | Block | Command in this lab | Job |
|---|---|---|---|
| 1 | **Pool** | `ip nat pool test 1.1.1.11 1.1.1.20 netmask 255.255.255.0` | The public addresses available to hand out |
| 2 | **Access list** | `access-list 1 permit 192.168.1.0 0.0.0.255` | **Which inside hosts** may be translated |
| 3 | **NAT rule** | `ip nat inside source list 1 pool test` | Binds the list to the pool |

```
 inside host ──► matches ACL 1? ──yes──► take a free address from pool "test" ──► translate
                       └──no──► not translated
```

> In NAT, the access list **selects** traffic to translate. It does not block anything. `permit` means "translate this", and a host that does not match is simply left alone.

### The wildcard mask

| ACL line | Meaning |
|---|---|
| `192.168.1.0 0.0.0.255` | Match **any** host from 192.168.1.0 to 192.168.1.255 |
| `0.0.0.255` | The first three octets must match exactly, and the last octet is free |

---

## ⚙️ Configuration

### NAT-ROUTER (Router0)

```
NAT-ROUTER(config)# ip nat pool test 1.1.1.11 1.1.1.20 netmask 255.255.255.0

NAT-ROUTER(config)# access-list 1 permit 192.168.1.0 0.0.0.255
NAT-ROUTER(config)# ip nat inside source list 1 pool test

NAT-ROUTER(config)# interface gigabitEthernet 0/0
NAT-ROUTER(config-if)# ip nat inside
NAT-ROUTER(config-if)# exit

NAT-ROUTER(config)# interface serial 0/0/0
NAT-ROUTER(config-if)# ip nat outside
NAT-ROUTER(config-if)# end
```

### Basic connectivity
> Replace this block with the routing you actually used.

```
NAT-ROUTER(config)# ip route 0.0.0.0 0.0.0.0 1.1.1.2
```

### Debugging (used for the proof)

```
NAT-ROUTER# debug ip nat
IP NAT debugging is on
```
> Turn debugging off afterwards with `undebug all`.

### Config breakdown

| Command | Purpose |
|---|---|
| `ip nat pool test 1.1.1.11 1.1.1.20 netmask 255.255.255.0` | Defines pool `test` with 10 public addresses |
| `access-list 1 permit 192.168.1.0 0.0.0.255` | Selects the whole inside LAN for translation |
| `ip nat inside source list 1 pool test` | Translates sources matching list 1 using pool `test` |
| `interface gigabitEthernet 0/0` + `ip nat inside` | Marks the private side |
| `interface serial 0/0/0` + `ip nat outside` | Marks the public side |

---

## ✅ Verification: Stage by Stage

### 🟢 Stage 1: PC0 can reach the ISP router

```
C:\> ping 1.1.1.2

Reply from 1.1.1.2: bytes=32 time<1ms TTL=254
Reply from 1.1.1.2: bytes=32 time=1ms  TTL=254
Reply from 1.1.1.2: bytes=32 time<1ms TTL=254
Reply from 1.1.1.2: bytes=32 time<1ms TTL=254

Packets: Sent = 4, Received = 4, Lost = 0 (0% loss)
Minimum = 0ms, Maximum = 1ms, Average = 0ms
```

**Annotations**
- 4 sent, 4 received: the translated traffic gets there and back.
- **TTL 254**: the ISP router replies with 255, and the NAT router (one hop) lowers it to 254.

---

### 🔍 Stage 2: PC0's translation in the debug output

```
NAT-ROUTER# debug ip nat

NAT:  s=192.168.1.4->1.1.1.11, d=1.1.1.2 [10]
NAT*: s=1.1.1.2, d=1.1.1.11->192.168.1.4 [6]
NAT:  s=192.168.1.4->1.1.1.11, d=1.1.1.2 [11]
NAT*: s=1.1.1.2, d=1.1.1.11->192.168.1.4 [7]
NAT:  s=192.168.1.4->1.1.1.11, d=1.1.1.2 [12]
NAT*: s=1.1.1.2, d=1.1.1.11->192.168.1.4 [8]
NAT:  s=192.168.1.4->1.1.1.11, d=1.1.1.2 [13]
NAT*: s=1.1.1.2, d=1.1.1.11->192.168.1.4 [9]
```

**Annotations**
- **Outbound** (`NAT:`): the source 192.168.1.4 is rewritten to **1.1.1.11**, the **first free address in the pool**.
- **Inbound** (`NAT*:`): the reply's destination 1.1.1.11 is rewritten back to **192.168.1.4**.
- Nothing in the config names 1.1.1.11 for PC0. The router **chose** it from the pool.

---

### 🔍 Stage 3: PC1 and PC2 get their own pool addresses

```
NAT:  s=192.168.1.3->1.1.1.11, d=1.1.1.2 [7]     ← PC1
NAT*: s=1.1.1.2, d=1.1.1.11->192.168.1.3 [11]    ← reply
NAT:  s=192.168.1.3->1.1.1.11, d=1.1.1.2 [8]
NAT*: s=1.1.1.2, d=1.1.1.11->192.168.1.3 [12]
NAT:  s=192.168.1.3->1.1.1.11, d=1.1.1.2 [9]
NAT*: s=1.1.1.2, d=1.1.1.11->192.168.1.3 [13]

NAT:  s=192.168.1.2->1.1.1.12, d=1.1.1.2 [6]     ← PC2
NAT*: s=1.1.1.2, d=1.1.1.12->192.168.1.2 [14]    ← reply
NAT:  s=192.168.1.2->1.1.1.12, d=1.1.1.2 [7]
NAT*: s=1.1.1.2, d=1.1.1.12->192.168.1.2 [15]
NAT:  s=192.168.1.2->1.1.1.12, d=1.1.1.2 [8]
NAT*: s=1.1.1.2, d=1.1.1.12->192.168.1.2 [16]
NAT:  s=192.168.1.2->1.1.1.12, d=1.1.1.2 [9]
NAT*: s=1.1.1.2, d=1.1.1.12->192.168.1.2 [17]
```

**Annotations**
- The `->` is the **translation symbol**: private address on the left, public address on the right.
- The `NAT*:` lines are the **replies** from the ISP, translated back to the private host.
- **PC1 got 1.1.1.11 and PC2 got 1.1.1.12**, so different hosts receive different pool addresses.
- PC0 got 1.1.1.11 earlier in Stage 2. The pool address is **not tied to a host**.

---

### 📋 Stage 4: The translation table with all three hosts active

```
NAT-ROUTER# show ip nat translations
Pro   Inside global    Inside local      Outside local    Outside global
icmp  1.1.1.12:10      192.168.1.4:10    1.1.1.2:10       1.1.1.2:10
icmp  1.1.1.12:11      192.168.1.4:11    1.1.1.2:11       1.1.1.2:11
icmp  1.1.1.12:12      192.168.1.4:12    1.1.1.2:12       1.1.1.2:12
icmp  1.1.1.12:9       192.168.1.4:9     1.1.1.2:9        1.1.1.2:9
icmp  1.1.1.13:5       192.168.1.3:5     1.1.1.2:5        1.1.1.2:5
icmp  1.1.1.13:6       192.168.1.3:6     1.1.1.2:6        1.1.1.2:6
icmp  1.1.1.13:7       192.168.1.3:7     1.1.1.2:7        1.1.1.2:7
icmp  1.1.1.13:8       192.168.1.3:8     1.1.1.2:8        1.1.1.2:8
icmp  1.1.1.14:5       192.168.1.2:5     1.1.1.2:5        1.1.1.2:5
icmp  1.1.1.14:6       192.168.1.2:6     1.1.1.2:6        1.1.1.2:6
icmp  1.1.1.14:7       192.168.1.2:7     1.1.1.2:7        1.1.1.2:7
icmp  1.1.1.14:8       192.168.1.2:8     1.1.1.2:8        1.1.1.2:8
```

| Inside local (private) | Inside global (pool) |
|---|---|
| 192.168.1.4 (PC0) | **1.1.1.12** |
| 192.168.1.3 (PC1) | **1.1.1.13** |
| 192.168.1.2 (PC2) | **1.1.1.14** |

**Annotations**
- Three hosts, **three different public addresses, at the same time**. This is dynamic NAT working.
- These are **temporary entries created by traffic** (one per ICMP echo), unlike the permanent entry in Static NAT.
- All addresses are inside the pool range (1.1.1.11 to 1.1.1.20).

---

### 📊 Stage 5: Pool statistics after the entries expired

```
NAT-ROUTER# show ip nat statistics
Total translations: 0 (0 static, 0 dynamic, 0 extended)
Outside Interfaces: Serial0/0/0
Inside Interfaces: GigabitEthernet0/0
Hits: 28  Misses: 90
Expired translations: 28
Dynamic mappings:
-- Inside Source
access-list 1 pool test refCount 0
 pool test: netmask 255.255.255.0
        start 1.1.1.11 end 1.1.1.20
        type generic, total addresses 10 , allocated 0 (0%), misses 0
```

| Field | Value | Meaning |
|---|---|---|
| Total translations | **0** | All temporary entries have timed out. Compare Stage 4, taken a couple of minutes earlier |
| Expired translations | 28 | The entries that aged out |
| Outside / Inside Interfaces | Serial0/0/0 / GigabitEthernet0/0 | Marked correctly ✅ |
| `access-list 1 pool test` | refCount 0 | The rule binds ACL 1 to pool `test` ✅ |
| pool `test` | 1.1.1.11 to 1.1.1.20 | Range as configured |
| total addresses | **10** | Pool size |
| **allocated** | **0 (0%)** | No address is in use. **They all went back to the pool** |
| pool misses | **0** | No host was ever refused because the pool was empty |
| Hits / Misses (top line) | 28 / 90 | Hits are packets that matched an existing entry. Misses are packets with no entry yet, where each new flow triggers the creation of one |

> By default, IOS expires ICMP translations after about 60 seconds of inactivity.

---

## 🔎 Reading the Debug Output

```
NAT:  s=192.168.1.3->1.1.1.11, d=1.1.1.2 [7]
       │        │        │        │       └─ IP identification of the packet
       │        │        │        └─ destination (unchanged)
       │        │        └─ source AFTER translation (inside global, from the pool)
       │        └─ source BEFORE translation (inside local)
       └─ no asterisk: translated in the normal (process-switched) path

NAT*: s=1.1.1.2, d=1.1.1.11->192.168.1.3 [11]
 │                  │          └─ destination AFTER translation (back to the private host)
 │                  └─ destination BEFORE translation (the pool address)
 └─ asterisk: translated in the fast-switching path (this is the reply)
```

| Direction | What changes | Debug shows |
|---|---|---|
| Inside → Outside | **Source** address | `s=192.168.1.3->1.1.1.11` |
| Outside → Inside | **Destination** address | `d=1.1.1.11->192.168.1.3` |

---

## 🔁 Address Assignment Across Test Runs

The router hands out pool addresses **first come, first served**. The same host does not always receive the same address.

| Run (simulator clock) | PC0 (192.168.1.4) | PC1 (192.168.1.3) | PC2 (192.168.1.2) |
|---|---|---|---|
| ~00:20 | **1.1.1.11** | not shown | not shown |
| ~00:22 | not shown | **1.1.1.11** | **1.1.1.12** |
| ~00:25 (table) | **1.1.1.12** | **1.1.1.13** | **1.1.1.14** |

This is the main difference from Static NAT, where a host **always** gets the same public address. With Dynamic NAT you cannot predict which pool address a host will use, so it suits **outbound** traffic, not servers that outsiders must reach.

---

## 📏 Pool Size Matters

```
 Pool "test": 1.1.1.11 ... 1.1.1.20  =  10 addresses  =  10 hosts at the same time
```

| Situation | Result |
|---|---|
| Up to 10 hosts active | ✅ Each gets its own public address |
| An 11th host tries while all 10 are in use | ❌ No free address, the packet is dropped and the **pool misses** counter goes up |
| Entries expire | ✅ Addresses return to the pool and new hosts can use them |

In this lab the pool-level counter stayed at `misses 0`, so the pool never ran out. This limit is exactly why **PAT (overload)** exists: it lets many hosts share **one** public address by using port numbers.

---

## 💡 Key Takeaways

| # | Takeaway |
|---|---|
| 1 | Dynamic NAT uses a **pool** of public addresses, assigned on demand |
| 2 | Three blocks: **pool**, **access list**, and the **`ip nat inside source list ... pool ...`** rule |
| 3 | In NAT, the ACL **selects** hosts to translate and does not block anything |
| 4 | Entries are **created by traffic** and **expire** when idle |
| 5 | Expired entries return their address to the pool |
| 6 | A host's public address can **change** between sessions |
| 7 | The pool is a **limit**: pool size = number of hosts active at once |
| 8 | `debug ip nat`, `show ip nat translations` and `show ip nat statistics` prove it works |

---

## 🛠 Troubleshooting Notes

| Symptom | Check |
|---|---|
| Nothing is translated | Are `ip nat inside` and `ip nat outside` set on the correct interfaces? |
| Some hosts are not translated | Does ACL 1 cover their addresses? Is the wildcard mask correct? |
| Translation table is empty | Entries only exist while traffic flows, so ping again, then check |
| Only the first hosts work | The pool may be exhausted: check `show ip nat statistics` for pool misses |
| Wrong pool addresses | Is the pool range correct and inside the WAN network? |
| Ping translated but no reply | Does the ISP have a way back to the pool addresses? |
| Clear entries | `clear ip nat translation *` |
| Debug floods the screen | Run `undebug all` |

---

## 🚀 Next Labs

- [x] Static NAT ✅
- [x] Dynamic NAT with a pool and an access list ✅
- [ ] Dynamic NAT: what happens when the pool is exhausted
- [ ] PAT / NAT overload (many hosts, one public address)
- [ ] Static NAT for a server with inbound access

---

## 👨‍💻 Author

**Maaz Khan**
CCNA-certified Network & NOC Engineer · Lower Dir, KPK, Pakistan

[![LinkedIn](https://img.shields.io/badge/LinkedIn-maazkhanms-0A66C2?style=for-the-badge&logo=linkedin)](https://www.linkedin.com/in/maazkhanms)
[![GitHub](https://img.shields.io/badge/GitHub-maazkhanms-181717?style=for-the-badge&logo=github)](https://github.com/maazkhanms)

⭐ If this lab helped you, give the repo a star!
