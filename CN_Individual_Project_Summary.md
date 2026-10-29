# Firewall Network Security Project: Team Guide

**Course:** Computer Networks · **Tool:** Cisco Packet Tracer · **Technique:** Router-based firewall using an Extended ACL

## 1. What are we building?

We simulate a **small company network** with three parts: an employee LAN, an admin LAN and a private server LAN. Three routers connect them. The **central router (`Firewall_Router`) acts as the firewall** by applying **Extended ACL 101**, which blocks one specific employee PC from reaching the private server while everything else is allowed.

**The result we demonstrate:** `Employee_PC` cannot ping the server, but `Admin_PC` can.

**Important:** this is a *simulated, router-based ACL firewall*, not a dedicated hardware firewall.

**Why it matters:** in a company, not every user should reach every resource. This project shows how an administrator can restrict access to a private server based on IP addresses.

## 2. The big picture

```text
Employee_PC (192.168.1.10)
   |
Employee_Switch
   |
Employee_Gateway (LAN 192.168.1.1)
   |  10.0.12.1/30
   |  10.0.12.2/30  (G0/0)
Firewall_Router  ----- G0/2 (192.168.3.1)  [ACL 101 applied OUTBOUND here] ----> Private_Server (192.168.3.10)
   |  10.0.23.1/30  (G0/1)
   |  10.0.23.2/30
Admin_Gateway (LAN 192.168.2.1)
   |
Admin_Switch
   |
Admin_PC (192.168.2.10)
```

**Key idea:** all traffic to the server must leave `Firewall_Router` through **G0/2**, so that is the one place where the ACL is applied.

## 3. Devices and IP plan

### Equipment

- 3 Cisco 2911 routers
- 2 Cisco 2960 switches
- 2 PCs (Employee PC and Admin PC)
- 1 server (Private Server)

### Device roles

| Device | Role |
| --- | --- |
| `Employee_Gateway` (Router0) | Connects the employee LAN to the central router |
| `Firewall_Router` (Router1) | Routes traffic and applies ACL 101 |
| `Admin_Gateway` (Router2) | Connects the admin LAN to the central router |
| `Employee_Switch` | Connects `Employee_PC` to its gateway |
| `Admin_Switch` | Connects `Admin_PC` to its gateway |
| `Private_Server` | Represents the protected company resource |

### IP addressing

| Device / interface | IP address |
| --- | --- |
| `Employee_PC` | `192.168.1.10/24` |
| `Employee_Gateway` LAN | `192.168.1.1/24` |
| `Employee_Gateway` ↔ `Firewall_Router` | `10.0.12.1/30` ↔ `10.0.12.2/30` |
| `Firewall_Router` ↔ `Admin_Gateway` | `10.0.23.1/30` ↔ `10.0.23.2/30` |
| `Admin_Gateway` LAN | `192.168.2.1/24` |
| `Admin_PC` | `192.168.2.10/24` |
| `Firewall_Router` server interface (G0/2) | `192.168.3.1/24` |
| `Private_Server` | `192.168.3.10/24` |

**Default gateways:** `Employee_PC` = `192.168.1.1`, `Admin_PC` = `192.168.2.1`, `Private_Server` = `192.168.3.1`.

**Subnet masks:** `/24` = `255.255.255.0`. `/30` = `255.255.255.252`, used for two-device point-to-point router links (only 2 usable addresses, so nothing is wasted).

## 4. How a ping travels (worked examples)

**Example A: `Admin_PC` pings the server (allowed)**

1. `Admin_PC` sees `192.168.3.10` is outside `192.168.2.0/24`, so it sends the packet to its gateway `192.168.2.1`.
2. `Admin_Gateway` has a static route for `192.168.3.0/24` via `10.0.23.1` and forwards it to `Firewall_Router`.
3. `Firewall_Router` knows `192.168.3.0/24` is directly connected on G0/2. Before sending the packet out, it checks **ACL 101 (outbound)**. Rule 10 does not match (the source is not `192.168.1.10`), so rule 20 **permits** it.
4. The server replies. The reply returns via `Firewall_Router`, which has a static route to `192.168.2.0/24` via `10.0.23.2`.

**Example B: `Employee_PC` pings the server (blocked)**

1. `Employee_PC` sends the packet to its gateway `192.168.1.1`.
2. `Employee_Gateway` forwards it to `10.0.12.2` using its static route for `192.168.3.0/24`.
3. `Firewall_Router` is about to send it out G0/2 and checks ACL 101. Source `192.168.1.10` and destination `192.168.3.10` match **rule 10 (deny)**, so the packet is dropped. The ping fails (it may show as a timeout or "destination host unreachable").

**Takeaway:** the network is fully connected. The employee ping fails because of the **ACL policy**, not because of a routing problem. That is exactly what we want to prove.

## 5. How the firewall works (ACL 101)

An **Extended ACL** is an ordered list of rules that permit or deny packets. ACL 101 contains:

```text
10 deny ip host 192.168.1.10 host 192.168.3.10
20 permit ip any any
```

- **Rule 10 (deny):** blocks IP traffic from `Employee_PC` to `Private_Server`.
- **Rule 20 (permit):** allows any other IP traffic that reaches this rule.
- Rules are checked **top to bottom**, and the **first match wins**.
- ACL 101 is applied **outbound on `Firewall_Router` interface G0/2**, the interface leading to the server network.
- `host` matches one exact IP address. `any` matches every address.
- Cisco ACLs end with an **implicit deny**, so without rule 20 everything not explicitly permitted would be dropped.

### Equivalent configuration (typical syntax)

```text
access-list 101 deny ip host 192.168.1.10 host 192.168.3.10
access-list 101 permit ip any any
!
interface GigabitEthernet0/2
 ip access-group 101 out
```

### Why an Extended ACL?

An extended ACL can match **source and destination** addresses, and also protocols and ports. A standard ACL generally filters on the source address only, so it could not say "block this PC from reaching only this server."

## 6. Routing

Static routes tell each router how to reach networks that are not directly connected.

**`Employee_Gateway` (Router0):**

```text
ip route 192.168.2.0 255.255.255.0 10.0.12.2
ip route 192.168.3.0 255.255.255.0 10.0.12.2
```

**`Firewall_Router` (Router1):**

```text
ip route 192.168.1.0 255.255.255.0 10.0.12.1
ip route 192.168.2.0 255.255.255.0 10.0.23.2
```

`192.168.3.0/24` is directly connected to `Firewall_Router` through G0/2, so it needs no static route.

**`Admin_Gateway` (Router2):**

```text
ip route 192.168.1.0 255.255.255.0 10.0.23.1
ip route 192.168.3.0 255.255.255.0 10.0.23.1
```

**Format:** `ip route <destination network> <mask> <next-hop IP>`.

## 7. Testing and expected results

| Test | Expected result | What it proves |
| --- | --- | --- |
| `Employee_PC` → `Private_Server` (`ping 192.168.3.10`) | **Fails** | The ACL blocks the specified traffic |
| `Admin_PC` → `Private_Server` (`ping 192.168.3.10`) | **Succeeds** | Other traffic is permitted |
| Router interface check | Interfaces show `up/up` | Links and interfaces are operational |
| ACL check | ACL 101 shows deny, then permit | Rules are present in the right order |

**First-ping behavior:** the first ping may lose packets while ARP resolves the destination MAC address. Repeat before concluding anything is broken. In our tests the admin ping succeeded on the second attempt, while the employee ping failed as intended.

**Do not overclaim.** The ACL blocks only `192.168.1.10` → `192.168.3.10`. The general permit rule allows other IP traffic, so we must **not** say "all unauthorized traffic is blocked."

## 8. Key concepts in plain language

| Term | Meaning | Where we use it |
| --- | --- | --- |
| **Firewall** | Controls traffic using security rules | `Firewall_Router` |
| **ACL** | Ordered permit/deny rules | ACL 101 |
| **Extended ACL** | ACL that matches source, destination, protocol and ports | ACL 101 |
| **Implicit deny** | Hidden final rule that drops anything not permitted | End of every ACL |
| **Static route** | Manually entered path to a remote network | All three routers |
| **Default gateway** | Router a PC uses for outside destinations | `.1` address in each LAN |
| **ARP** | Finds the MAC address for an IP on the local network | Why the first ping may fail |
| **Switch / Router** | Switch connects devices in a LAN; router connects different IP networks | Switches and gateways |

## 9. Demo cheat sheet

| Step | Where | Command / action | What to point out |
| --- | --- | --- | --- |
| 1 | Topology | Identify the PCs, three routers, switches and server | The three segments |
| 2 | `Firewall_Router` | `show ip interface brief` | G0/0 = `10.0.12.2`, G0/1 = `10.0.23.1`, G0/2 = `192.168.3.1`, all `up/up` |
| 3 | `Firewall_Router` | `show ip route` | `C` and `L` routes are automatic; static routes are marked `S` |
| 4 | `Firewall_Router` | `show access-lists` | The deny rule, then the permit rule |
| 5 | `Firewall_Router` | `show ip interface gigabitEthernet 0/2` | ACL 101 is the **outgoing** access list |
| 6 | `Employee_PC` | `ping 192.168.3.10` | **Blocked** |
| 7 | `Admin_PC` | `ping 192.168.3.10` | **Allowed** |
| 8 | Wrap-up | Save the `.pkt` file | Keep both ping results ready |

**Tip:** after the pings, run `show access-lists` again. The deny rule's **match counter** goes up, which is direct proof that the ACL is what dropped the employee traffic.

## 10. Short presentation script

> "Good morning. My project is Implementation of Firewall Network Security System Using Cisco Packet Tracer.
>
> The objective is to demonstrate IP-based access control in a small company network. The topology has an employee network, an admin network and a private server network, connected through three routers.
>
> The central router uses Extended ACL 101. The first rule denies traffic from Employee\_PC to Private\_Server. The second rule permits other IP traffic. The ACL is applied outbound on interface G0/2, which connects to the server network.
>
> I verified the interface status and routing table, then tested with ping. Employee\_PC could not reach the private server, while Admin\_PC could. This shows how an ACL can enforce a specific access policy.
>
> This is a simulation of a router-based ACL firewall. Thank you."

## 11. Likely viva questions, with answers

**What is a firewall?** A system that controls network traffic according to security rules, allowing or blocking selected traffic.

**What is an ACL?** An Access Control List: an ordered set of rules used to permit or deny packets.

**Why an Extended ACL?** It matches both source and destination IP addresses, while a standard ACL typically matches only the source.

**Why is the deny rule first?** Rules are checked in order and the first match wins. The specific deny must come before the general permit, otherwise the permit would match first and the employee would never be blocked.

**What does `permit ip any any` mean?** It permits IP traffic from any source to any destination, if the packet reaches that rule.

**Why apply the ACL outbound on G0/2?** All traffic to the private server leaves the central router through G0/2, so an outbound ACL there filters it just before it reaches the server network, whichever path it took to get there.

**What would happen if the permit rule were removed?** The implicit deny at the end of every ACL would drop all traffic not explicitly permitted, including the admin's.

**What is a static route?** A manually configured route telling a router how to reach a remote network.

**Why can the first ping fail but the next succeed?** The devices may first need to resolve the destination MAC address using ARP.

**Is this a dedicated firewall?** No. It is a simulated router-based firewall using an Extended ACL in Packet Tracer.

**Does this ACL block all employee access to the server?** No. It blocks IP traffic from the specific address `192.168.1.10` to the specific address `192.168.3.10`. Another employee PC with a different address would not be blocked, and other traffic is allowed.

**What are the limitations?** It demonstrates basic IP-based filtering only. It has no application inspection, intrusion prevention or centralized security monitoring, and it is not stateful like a real firewall.

**What is the difference between a switch and a router?** A switch connects devices within one LAN. A router forwards packets between different IP networks.

**What is a subnet mask?** It identifies which part of an IPv4 address is the network and which part is the host.

## 12. Troubleshooting checklist

If a result is not what you expect, work bottom-up:

1. **PC / server:** run `ipconfig`. Are the IP, mask and gateway correct?
2. **Gateway reachable:** can each device ping its own gateway?
3. **Interfaces:** `show ip interface brief`. Are all interfaces `up/up` with the right IPs?
4. **Routes:** `show ip route`. Does each router have routes to the remote subnets, in both directions?
5. **ACL:** `show access-lists` and `show ip interface gigabitEthernet 0/2`. Is ACL 101 present, in the right order, and applied **out** on G0/2?
6. **Interpret the result:** if the **admin** ping also fails, suspect routing or addressing (not the ACL). If the **employee** ping *succeeds*, suspect the ACL is missing, mis-ordered, applied on the wrong interface or in the wrong direction, or the employee PC's address does not match `192.168.1.10`.

## 13. Final checks before presenting

- [ ] Save the final `.pkt` file.
- [ ] Confirm IPs, masks and gateways on every device.
- [ ] Confirm static routes on all three routers.
- [ ] Confirm ACL 101 exists and is applied **out** on G0/2.
- [ ] Repeat both pings before the demo and keep the results ready.
- [ ] Make sure every teammate can explain rule 10, rule 20 and why the two pings differ.

**If you forget an exact command in the viva,** explain the concept clearly rather than guessing.

## 14. Items to double-check

- The original notes call this an **individual project**, while the campus network guide describes a **group project**. Confirm whether this firewall scenario is a separate submission or part of the same deliverable.
- The two projects use different address ranges (`192.168.1.0`, `.2.0`, `.3.0` here vs. `192.168.10.0` to `.31.0` in the campus network), so do not mix up the IP tables between them.
- The notes list the ACL rules and placement but not the exact commands used to apply them. The "equivalent configuration" in Section 5 is standard Cisco syntax; check it against the real `show running-config`.
