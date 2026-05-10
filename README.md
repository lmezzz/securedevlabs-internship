# 🛡️ SecureDevLabs Cybersecurity Internship

A collection of weekly write-ups, notes, and learnings from my online cybersecurity internship at **SecureDevLabs**. Each week covers a new topic through hands-on tasks — from setting up lab environments to active reconnaissance, exploitation, and beyond.

> ⚠️ All activities are performed in isolated, controlled lab environments. Nothing here is intended for use on real systems without authorization.

---

## 📁 Weekly Progress

| Week | Topic | Key Tools |
|------|-------|-----------|
| [Week 1](./Week-1-Recon-and-Lab-Setup/README.md) | Lab Setup & Network Reconnaissance | `nmap`, `ping`, `ip` |
| [Week 2](./Week-2-Web-Application-Mapping/README.md) | Web Application Mapping & Input Discovery | `zaproxy`, `dig` |
| [Week 3](./Week-3-Web-Vulnerability-Validation/README.md) | Web Vulnerability Validation | `sqlmap`, `zaproxy`, `firefox devtools` |

---

## 🧰 Lab Environment

- **Attacker Machine:** Host Linux system (`lexie@LexLuthor`) — Arch Linux
- **Target Machine:** Metasploitable 2 VM — an intentionally vulnerable Linux system running DVWA
- **Network:** Isolated virtual network (`192.168.100.0/24`) via KVM/libvirt
- **Hypervisor:** KVM with libvirt virtual bridge (`virbr1`)
- **Proxy:** OWASP ZAP listening on `127.0.0.1:8080`, Firefox manually proxied through it

---

## 🎯 Goals

- Build practical, hands-on cybersecurity skills
- Document learnings in a way that's useful to others
- Work toward a strong foundation in penetration testing methodologies

---

## 🗺️ Methodology So Far

```
Week 1: Environment setup → network scanning → host discovery
Week 2: HTTP fundamentals → traffic interception → attack surface mapping
Week 3: SQLi exploitation → XSS exploitation → filter bypass → mitigations
```

---

*Updated weekly throughout the internship.*
