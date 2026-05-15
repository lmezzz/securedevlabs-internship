# 🛡️ SecureDevLabs Cybersecurity Internship

This repository contains my documented work, notes, and practical write-ups from the **SecureDevLabs Cybersecurity Internship**. The internship focused on developing hands-on cybersecurity skills through weekly tasks involving lab setup, reconnaissance, web application mapping, vulnerability validation, exploitation, and VAPT reporting.

All work was performed in isolated, controlled lab environments using intentionally vulnerable systems and authorized practice targets.

> ⚠️ **Disclaimer:**  
> This repository is for educational and authorized cybersecurity training only. The techniques, tools, and findings documented here must not be used against real systems without explicit permission.

---

## 📌 Internship Overview

The internship was structured around weekly practical tasks that followed a realistic penetration testing workflow. Each week introduced a new area of cybersecurity and required hands-on implementation, testing, documentation, and reflection.

The main focus areas included:

- Setting up a safe virtual cybersecurity lab
- Performing network reconnaissance
- Mapping web application attack surfaces
- Intercepting and analyzing HTTP traffic
- Validating common web vulnerabilities
- Understanding exploitation workflows
- Preparing a structured VAPT report
- Documenting findings with evidence and remediation steps
- Practicing responsible and ethical security testing

---

## 📁 Weekly Progress Summary

| Week | Topic | Main Focus | Key Tools |
|------|-------|------------|-----------|
| [Week 1](./Week-1-Recon-and-Lab-Setup/README.md) | Lab Setup & Network Reconnaissance | Building the lab, identifying hosts, scanning services | `nmap`, `ping`, `ip` |
| [Week 2](./Week-2-Web-Application-Mapping/README.md) | Web Application Mapping & Input Discovery | Understanding web app structure, proxying traffic, discovering inputs | `OWASP ZAP`, `dig`, `Firefox` |
| [Week 3](./Week-3-Web-Vulnerability-Validation/README.md) | Web Vulnerability Validation | Testing SQL injection, XSS, filter bypasses, and mitigations | `sqlmap`, `OWASP ZAP`, `Firefox DevTools` |
| [Week 4](./Week-4-VAPT-Report/README.md) | VAPT Report & Exploit Documentation | Documenting vulnerability analysis, exploitation evidence, impact, and remediation | `KVM/libvirt`, `Docker`, `gcc`, `pkexec`, `PolicyKit` |

---

## 🧰 Lab Environment

The lab was built using local virtualization to ensure all testing remained isolated and controlled.

### Attacker Machine

- **Host:** `lexie@LexLuthor`
- **Operating System:** Arch Linux
- **Role:** Attacker / testing machine
- **Tools Used:** `nmap`, `OWASP ZAP`, `sqlmap`, browser developer tools, Linux networking utilities, Docker, GCC

### Target Machines

- **Metasploitable 2 VM**
  - Intentionally vulnerable Linux target
  - Used for reconnaissance, web mapping, and web vulnerability validation
  - Hosted vulnerable services and DVWA

- **Ubuntu 20.04 Lab VM**
  - Used for local privilege escalation validation
  - Deployed using KVM/libvirt
  - Used for the Week 4 VAPT report based on PwnKit / CVE-2021-4034

### Network

- **Type:** Isolated virtual lab network
- **Subnet:** `192.168.100.0/24`
- **Hypervisor:** KVM/libvirt
- **Virtual Bridge:** `virbr1`
- **Proxy Setup:** OWASP ZAP on `127.0.0.1:8080`
- **Browser:** Firefox manually configured to route traffic through ZAP

---

## 🎯 Internship Goals

The internship helped me work toward the following goals:

- Build confidence using a Linux-based security testing environment
- Understand how attackers discover and map targets
- Learn how to identify exposed services and application entry points
- Practice using professional security tools in a controlled way
- Validate vulnerabilities instead of only assuming their presence
- Understand the difference between scanning, exploitation, and reporting
- Learn how to support findings with evidence
- Improve technical documentation and write-up quality
- Build a stronger foundation for future work in VAPT, red teaming, and offensive security

---

## 🗺️ Methodology Followed

The internship followed a practical penetration testing workflow:

```text
Lab Setup
    ↓
Network Discovery
    ↓
Service Enumeration
    ↓
Web Application Mapping
    ↓
Input and Parameter Discovery
    ↓
Vulnerability Identification
    ↓
Vulnerability Validation
    ↓
Impact Analysis
    ↓
Remediation Planning
    ↓
Professional Reporting
