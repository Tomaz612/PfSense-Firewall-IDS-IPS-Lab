# pfSense Firewall Lab — DoS Attack Simulation & Mitigation

## Overview

This project sets up a small virtualized network to simulate an external attacker targeting an internal host protected by a pfSense firewall. The lab consists of three virtual machines — pfSense (firewall), Kali Linux (attacker), and Ubuntu Desktop (victim) — used to demonstrate a Denial-of-Service (DoS) attack, capture it with Wireshark, and mitigate it using firewall rules.

**Status:** pfSense installation and network configuration complete. Ubuntu VM in progress. Kali VM, attack simulation, and mitigation still pending.

## Lab Architecture

| Device | Network | IP / Subnet | Role |
|---|---|---|---|
| Home Network | Physical | 192.168.1.0/24 | Provides internet access |
| pfSense — WAN | Bridged | 192.168.1.162/24 (DHCP) | Edge firewall interface facing home network |
| pfSense — LAN | Internal (`intnet`) | 192.168.2.1/24 | Internal firewall interface |
| Kali Linux (Attacker) | Bridged | TBD | External attacker host |
| Ubuntu Desktop (Victim) | Internal (`intnet`) | 192.168.2.100–199 (DHCP) | Internal victim host |

![Architecture](images/pfsense_dos_lab_architecture.png)


## Part 1 — Installation and Configuration

### 1.1 Virtual Machines

**pfSense**
- FreeBSD 64-bit, 2 vCPU, 2 GB RAM, 20 GB disk
- Adapter 1: Bridged (WAN)
- Adapter 2: Internal Network → `intnet` (LAN)
- Installed edition: pfSense CE (Community Edition)
- Filesystem: ZFS (Stripe, single disk)

**Ubuntu Desktop (Victim)**
- Ubuntu 64-bit, 2 vCPU, 2 GB RAM, 15 GB disk
- Adapter 1: Internal Network → `intnet`

**Kali Linux (Attacker)**
- Ubuntu 64-bit, 2 vCPU, 2 GB RAM, 15 GB disk
- Adapter 1: Bridged (WAN)

### 1.2 Interface Assignment

Interfaces were assigned via the pfSense console:
- WAN → `em0`
- LAN → `em1`

### 1.3 Network Addressing

The LAN interface was manually set to `192.168.2.1/24` to avoid a subnet conflict with the home network (which also uses `192.168.1.0/24`). IPv6 was disabled, and HTTPS was kept as the webConfigurator protocol.

### 1.4 Enabling GUI Access

The firewall was temporarily disabled from the console shell to allow first-time login:

```bash
pfctl -d
```

The web interface was then reached at `https://192.168.1.162` using the default credentials (`admin` / `pfsense`). A permanent rule was created to preserve GUI access going forward:

| Field | Value |
|---|---|
| Interface | WAN |
| Action | Pass |
| Protocol | TCP |
| Source | Network: 192.168.1.0/24 |
| Destination | WAN address |
| Destination port | 443 |
| Description | Allow GUI access from home network |


### 1.5 DHCP Server (LAN)

Configured under **Services ▸ DHCP Server ▸ LAN**:

![DHCP configuration](images/dhcp_config.png)

| Field | Value |
|---|---|
| Range | 192.168.2.10 – 192.168.2.199 |
| DNS server | 192.168.1.254 (home router) |

---

## Part 2 — Attack Simulation and Detection

### 2.1 Verifying Host Connectivity

**Ubuntu (Victim):**

```bash
sudo apt update
sudo apt install net-tools
ifconfig
ping 8.8.8.8
```

Confirms Ubuntu received an address in the `192.168.2.0/24` range via DHCP and has outbound internet access through pfSense.

![Ubunto Commands](images/ubunto_command_outputs.png)


**Kali (Attacker):**

```bash
ifconfig
ping 192.168.1.162
```
![Failed Pings](images/kali_command_outputs.png)


Pinging the pfSense WAN address initially failed. Two separate issues on pfSense were responsible:

1. **"Block private networks" on WAN.** By default, pfSense drops inbound traffic on the WAN interface that originates from RFC1918 private addresses, since WAN traffic is normally expected to come from the public internet. Since this entire lab runs on private ranges, this default rule silently dropped all of Kali's traffic. Fix: **Interfaces ▸ WAN** → uncheck *"Block private networks and loopback addresses"* under Reserved Networks.
2. **No explicit rule allowing Kali's traffic.** A rule was added on **Firewall ▸ Rules ▸ WAN** to allow ICMP from Kali's IP toward pfSense/Ubuntu.

![Successfull Pings](images/rule_to_pfsense.png)


Once both were addressed, the ping succeeded:

```bash
ping 192.168.1.162
```
![Successfull Pings](images/ping_kali_to_pfsense.png)


### 2.2 Allowing Kali → Ubuntu Traffic

A firewall rule was added on **Firewall ▸ Rules ▸ WAN** to allow traffic from Kali toward the Ubuntu victim:

![Firewall rule](images/rule_kali_to_ubunto.png)

| Field | Value |
|---|---|
| Action | Pass |
| Protocol | Any |
| Source | 192.168.1.125 (Kali) |
| Destination | 192.168.2.10 (Ubuntu) |
| Description | Allow Kali access to Ubuntu |

> **Note on protocol:** an initial rule restricted to ICMP allowed `ping` to succeed but silently blocked the later `hping3` TCP traffic. The rule protocol was widened to `Any` to allow the attack traffic through for this demonstration — a real-world equivalent of an attacker who has already found an open path to the target.

A static route was added on Kali so that return/forward traffic for the internal `192.168.2.0/24` subnet is correctly routed via the pfSense WAN address, since Kali has no direct knowledge of the internal LAN:

```bash
sudo ip route add 192.168.2.0/24 via 192.168.1.162
```
![Static Route](images/static_route.png)


Successfull pings now:
![Pings](images/ping_to_ubuntu_vm.png)


### 2.3 Installing Wireshark (Ubuntu)

```bash
sudo apt install wireshark
sudo wireshark
```
Capture was started on the primary interface (`enp0s3`) to observe incoming traffic during the attack.


### 2.4 Simulating the DoS Attack

From Kali, a SYN flood was launched against the Ubuntu victim on port 80:

```bash
sudo hping3 -S -p 80 --flood 192.168.2.10
```

```
--- 192.168.2.10 hping statistic ---
285256 packets transmitted, 0 packets received, 100% packet loss
```
![statistics](images/dos_command.png)

**On the reported packet loss:** in `--flood` mode, `hping3` disables response listening entirely in order to maximize send throughput — it never checks for replies, so `packets received` is always 0 and the reported loss is not a measure of whether the attack succeeded. Delivery of the attack traffic was instead confirmed independently through:

- **pfSense rule counters** (Firewall ▸ Rules ▸ WAN), showing the Kali → Ubuntu rule matching tens of thousands of packets during the attack window
- **Wireshark capture on Ubuntu**, showing a continuous stream of incoming SYN packets from Kali's IP

This is the expected behavior of a SYN flood: the goal isn't to receive replies, but to exhaust the victim's connection-handling resources with a high volume of half-open TCP connection requests.


### 2.5 Observing the Attack in Wireshark

![Wireshark capture](images/wireshark_packet.png)

The capture shows a sustained flood of `SYN` packets from Kali's IP (`192.168.1.125`) to Ubuntu's IP (`192.168.2.10`) on port 80, with no corresponding `SYN-ACK`/`ACK` handshake completion — consistent with a SYN flood pattern.

---


## Part 3 — Mitigation

### 3.1 Blocking Kali → Ubuntu TCP Traffic

To stop the attack, a **block** rule was added on **Firewall ▸ Rules ▸ WAN**:

| Field | Value |
|---|---|
| Action | Block |
| Protocol | TCP |
| Source | Kali IP |
| Destination | Ubuntu IP |

![Firewall block rule](images/block_tcp_rule.png)

pfSense evaluates rules top-to-bottom on the interface and applies the **first match**, so this block rule was placed above the earlier "Allow Kali access to Ubuntu" rule — otherwise the permissive rule would keep matching first and the block would never be reached.


### 3.2 Re-running the DoS Attack

The same attack was launched again from Kali to test the new rule:

```bash
sudo hping3 -S -p 80 --flood 192.168.2.10
```

This time, **396,968 packets** were transmitted:

![Attack repeated with block rule active](images/print_attack_again.png)

### 3.3 Verifying the Block in pfSense

Checking **Firewall ▸ Rules ▸ WAN** confirms the new rule caught the traffic: **396,987 packets blocked**, matching the volume sent by Kali. This confirms the rule is working as intended — the flood is being dropped at the firewall instead of reaching the internal network.

![Blocked packet count in pfSense](images/pfsense_gui_packets_blocked.png)

---

## Troubleshooting

A few issues came up during setup that are worth documenting, since they're common pitfalls in this kind of lab.

**DNS resolution failing despite working internet connectivity**

While verifying the Ubuntu VM, `sudo apt install net-tools` failed to run. Diagnosis steps:

1. Checked connectivity to the default gateway — worked
2. Checked `ping 8.8.8.8` — worked (confirms raw IP routing is fine)
3. Checked `ping google.com` — failed (points specifically to DNS resolution, not routing)

Since IP-based connectivity worked but name resolution didn't, the issue was narrowed down to an incorrect DNS server rather than a network/firewall problem. Running `ip route | grep default` on the host machine's own connection confirmed the actual home router address is `192.168.1.254` — not `192.168.1.1`, which had been assumed and configured as the DNS server in pfSense.

**Fix:** in the pfSense GUI, under **Services ▸ DHCP Server ▸ LAN**, the DNS Servers field was updated to `192.168.1.254`. Ubuntu picked up the corrected DNS server on its next DHCP renewal, and both `apt` and `ping google.com` started working.

> **Takeaway:** when IP-based tests succeed but name-based tests fail, the issue is almost always DNS — check the resolver configuration before suspecting routing or firewall rules.

## Next Steps

- [ ] Install and configure **Suricata** on pfSense as an IDS/IPS to detect and automatically respond to flood-style traffic patterns, rather than relying solely on manually created static block rules

---

## Key Takeaways

- pfSense's default WAN protections (block private networks/bogons) can silently interfere with fully-private lab topologies and must be adjusted deliberately, not disabled blindly in production.
- Firewall rules are protocol-specific — an ICMP allow rule does not imply TCP is allowed; each protocol needed for the test must be explicitly permitted.
- `hping3 --flood` statistics report transmission, not delivery — attack success must be verified independently (firewall counters, packet capture) rather than trusted from the tool's own summary.

## Repository

**Name:** `PfSense-DoS-Lab`
**Description:** Virtualized lab simulating a DoS attack from an external host through a pfSense firewall, with traffic capture in Wireshark and mitigation via custom firewall rules.


