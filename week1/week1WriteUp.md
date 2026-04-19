# Week 1 — Lab Setup & Network Reconnaissance

**Internship:** SecureDevLabs  
**Topic:** Building a Vulnerable Lab Environment & Basic Recon  
**Tools Used:** VirtualBox/KVM, Metasploitable 2, `ip`, `ping`, `nmap`

---

## Overview

The first week focused on setting up a controlled penetration testing lab environment and performing foundational reconnaissance tasks. This involved spinning up a **Metasploitable 2** virtual machine (an intentionally vulnerable Linux system used for security practice), verifying network connectivity, discovering live hosts, and scanning for open ports and running services.

---

## Task 1 — Build the Lab Environment

### What is Metasploitable 2?
Metasploitable 2 is a deliberately vulnerable Linux VM maintained by Rapid7. It is designed to be attacked and is commonly used to practice exploitation techniques in a safe, legal environment. It should **never** be exposed to a public network.

### What was done
- Launched a Metasploitable 2 VM (username: `msfadmin@metasploitable`)
- Verified the network interfaces on both machines using the `ip a` command

### Key Observations
- The host machine had multiple interfaces: `wlp2s0` (Wi-Fi, IP: `192.168.0.61`) and `virbr1` — a **virtual bridge interface** used to connect the VM to the host network
- A `vnet0` interface appeared, which is the virtual NIC assigned to the Metasploitable VM by the hypervisor
- The Metasploitable VM was assigned IP `192.168.100.177` on the `eth0` interface within the isolated virtual network (`192.168.100.0/24`)
- The gateway/host-side bridge (`virbr1`) had IP `192.168.100.1`

### Commands Used
```bash
ip a          # Show all network interfaces and their IP addresses
```

---

## Task 2 — Confirming Connectivity

### Concept: What is `ping`?
`ping` uses **ICMP Echo Request/Reply** packets to test whether a host is reachable on a network. It also measures round-trip time (RTT), which tells you about latency.

### What was done
Two connectivity tests were performed:

**From Host → Metasploitable VM:**
```bash
ping 192.168.100.177
```
```
64 bytes from 192.168.100.177: icmp_seq=1 ttl=64 time=0.720 ms
64 bytes from 192.168.100.177: icmp_seq=2 ttl=64 time=0.823 ms
64 bytes from 192.168.100.177: icmp_seq=3 ttl=64 time=0.746 ms
64 bytes from 192.168.100.177: icmp_seq=4 ttl=64 time=0.850 ms
4 packets transmitted, 4 received, 0% packet loss
```

**From Metasploitable VM → Host bridge:**
```bash
ping -c 2 192.168.100.1
```
```
64 bytes from 192.168.100.1: icmp_seq=1 ttl=64 time=0.195 ms
64 bytes from 192.168.100.1: icmp_seq=2 ttl=64 time=0.683 ms
2 packets transmitted, 2 received, 0% packet loss
```

### Key Observations
- **0% packet loss** in both directions confirms the virtual network is functioning correctly
- TTL of **64** indicates the target is a Linux machine (Linux defaults to TTL 64; Windows uses 128)
- Low RTT (~0.7ms host→VM, ~0.4ms VM→host) is expected for a local virtual network
- `-c 2` flag limits ping to 2 packets — useful to avoid spamming in scripts

---

## Task 3 — Identifying Live Hosts

### Concept: Network Host Discovery
Before attacking or auditing a network, a penetration tester needs to know **which hosts are actually online**. Scanning every port on every IP is slow — host discovery narrows the target list first.

### What was done
Used `nmap` with the `-sn` flag (ping scan / no port scan) to discover live hosts on the entire `/24` subnet:

```bash
nmap -sn 192.168.100.0/24
```

### Output
```
Nmap scan report for 192.168.100.1   → Host is up (0.00027s latency)
Nmap scan report for 192.168.100.177 → Host is up (0.00083s latency)
Nmap done: 256 IP addresses (2 hosts up) scanned in 3.11 seconds
```

### Key Observations
- Only **2 hosts** were live on the network: the gateway (`192.168.100.1`) and the Metasploitable VM (`192.168.100.177`) — exactly as expected in our isolated lab
- The `-sn` flag tells Nmap to skip port scanning entirely and only check if hosts are up, making it much faster
- Nmap scanned all 256 possible addresses in the subnet in just 3.11 seconds

---

## Task 4 — Identifying Active Ports and Services

### Concept: Port Scanning
Every service running on a machine listens on a **port** (a numbered channel, 0–65535). By scanning ports, we can determine what software is running — this is a core step in any security assessment.

### What was done
Used `nmap` with the `-sT` flag (TCP Connect scan) to identify all open ports on the Metasploitable VM:

```bash
nmap -sT 192.168.100.177
```

### Output — Open Ports Found

| Port | Service | Notes |
|------|---------|-------|
| 21/tcp | FTP | File Transfer — often misconfigured, allows anonymous login on Metasploitable |
| 22/tcp | SSH | Secure Shell — encrypted remote access |
| 23/tcp | Telnet | Unencrypted remote access — **highly insecure** |
| 25/tcp | SMTP | Email sending protocol |
| 53/tcp | Domain (DNS) | Domain name resolution |
| 80/tcp | HTTP | Web server — unencrypted |
| 111/tcp | RPCbind | Remote Procedure Call — can expose NFS |
| 139/tcp | NetBIOS-SSN | Windows file sharing over older protocol |
| 445/tcp | Microsoft-DS | SMB — file sharing, famous for EternalBlue exploit |
| 512/tcp | exec | Remote execution — **dangerous** |
| 513/tcp | login | Remote login — **dangerous** |
| 514/tcp | shell | Remote shell — **dangerous** |
| 1099/tcp | RMIregistry | Java RMI — remote method invocation |
| 1524/tcp | ingreslock | Backdoor shell on Metasploitable |
| 2049/tcp | NFS | Network File System — can expose file system |
| 2121/tcp | ccproxy-ftp | Secondary FTP proxy |
| 3306/tcp | MySQL | Database server — should never be exposed publicly |

### Key Observations
- Metasploitable 2 has a **massive attack surface** — almost every service running is either outdated, misconfigured, or intentionally vulnerable
- Services like **Telnet (23), exec (512), shell (514)** transmit data in plaintext and should never be used in production
- **Port 1524** is a known **intentional backdoor** in Metasploitable — connecting to it gives a root shell directly
- **MySQL on 3306** being publicly accessible is a critical misconfiguration in real environments
- The `-sT` flag performs a full TCP 3-way handshake (SYN → SYN-ACK → ACK) — more reliable but also more detectable than a SYN scan (`-sS`)

---

## Key Takeaways

1. **Virtual bridges** (`virbr1`, `vnet0`) are how hypervisors create isolated networks for VMs without touching the physical network
2. **`ping`** is not just a connectivity check — TTL values reveal OS type, and RTT reveals latency/routing
3. **Host discovery before port scanning** is standard practice — don't waste time scanning dead hosts
4. **Open ports = potential attack surface** — every unnecessary service running is a risk
5. Metasploitable 2 is intentionally broken — it shows what a **worst-case misconfigured server** looks like, which is why it's such a good learning target

---

## Commands Cheatsheet

```bash
# View network interfaces
ip a

# Test connectivity (continuous)
ping <target-ip>

# Test connectivity (limited packets)
ping -c <count> <target-ip>

# Discover live hosts on a subnet (no port scan)
nmap -sn <subnet/CIDR>

# TCP connect scan for open ports
nmap -sT <target-ip>

# Faster SYN scan (requires root)
sudo nmap -sS <target-ip>

# Scan + detect service versions
nmap -sV <target-ip>

# Scan all 65535 ports
nmap -p- <target-ip>
```

---

*Part of my SecureDevLabs cybersecurity internship write-ups. New entries added weekly.*
