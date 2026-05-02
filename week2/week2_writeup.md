# Week 2 — Web Application Mapping and Input Discovery

## Overview

This week focused on understanding the structure of web applications before testing vulnerabilities. The goal was to build mental models around how HTTP works, how traffic can be intercepted, how identity and permissions are managed, and how DNS resolves domain names to IPs.

---

## Environment

- OS: Arch Linux
- Proxy Tool: OWASP ZAP
- Target: DVWA running on Metasploitable 2
- Browser: Firefox (manually proxied through ZAP on 127.0.0.1:8080)

---

## HTTP Request & Response

HTTP is a structured conversation between a browser and a server. Every interaction follows the same format.

**Request structure:**
```
GET /dvwa/vulnerabilities/sqli/?id=1 HTTP/1.1
Host: 192.168.100.177
Cookie: PHPSESSID=abc123
User-Agent: Mozilla/5.0
```

**Response structure:**
```
HTTP/1.1 200 OK
Content-Type: text/html
Set-Cookie: PHPSESSID=abc123
```

The request consists of a request line (verb + path + HTTP version), headers, and an optional body. The response consists of a status line, headers, and a body.

**Key insight:** Headers carry metadata that is often more valuable to an attacker than the body itself. Response headers like `Server: Apache/2.4.49` and `X-Powered-By: PHP/7.4` leak technology fingerprints. Request headers like `Cookie` and `Authorization` carry session credentials.

**Common HTTP status codes:**

| Code | Meaning |
|------|---------|
| 200 | OK |
| 301/302 | Redirect |
| 400 | Bad request |
| 401 | Unauthorized (not authenticated) |
| 403 | Forbidden (not authorized) |
| 404 | Not found |
| 500 | Internal server error |

---

## Cookie Security Flags

Two flags control cookie security:

- `HttpOnly` — cookie cannot be read by JavaScript. Blocks XSS-based cookie theft via `document.cookie`.
- `Secure` — cookie is only sent over HTTPS. Blocks interception over plain HTTP.

A well-configured session cookie requires both. Absence of either is a security finding.

---

## HTTPS and TLS

HTTPS is HTTP wrapped in TLS. The request/response structure is identical — TLS encrypts the connection before any HTTP is transmitted. A network observer sees ciphertext instead of plaintext.

---

## Intercepting Traffic with ZAP

ZAP operates as a TLS-terminating proxy using two separate tunnels:

```
Browser <— TLS #1 —> ZAP <— TLS #2 —> Server
```

- Tunnel 1: Browser trusts ZAP's certificate (installed manually into Firefox certificate store)
- Tunnel 2: ZAP establishes a fresh TLS connection to the real server

ZAP sits in the middle in plaintext — full visibility into every request and response. This is the same technique used by corporate firewalls for SSL inspection.

**Setup on Arch:**
```bash
sudo pacman -S zaproxy
zaproxy
```

Firefox proxy configured to `127.0.0.1:8080`. ZAP's CA certificate exported from Options → Network → Server Certificates and imported into Firefox certificate store.

**Captured login request:**
```
POST /dvwa/login.php HTTP/1.1
Host: 192.168.100.177
Content-Type: application/x-www-form-urlencoded

username=admin&password=pass&Login=Login
```

Credentials visible in plaintext — transmitted over HTTP, not HTTPS. The server responded with `Set-Cookie: PHPSESSID=...` confirming session creation after successful authentication.

---

## Attack Surface Mapping

The attack surface of a web application is every input the server accepts and processes. Inputs fall into the following categories:

- URL parameters — `?id=1&sort=asc`
- Form fields — login, search, file upload
- Cookies — server reads and acts on cookie values
- HTTP headers — `User-Agent`, `Referer`, `X-Forwarded-For`
- Path segments — `/user/123/profile` where `123` is user-controlled
- Request body — JSON or form-encoded POST data

**Methodology:** ZAP's Spider tool was used to crawl DVWA automatically. It parses HTML responses for `href` attributes in `<a>` tags, `action` attributes in `<form>` tags, and redirect headers — then recursively visits each discovered URL.

**Endpoints discovered on DVWA:**

| Endpoint | Method | User-controlled inputs |
|----------|--------|----------------------|
| /dvwa/login.php | POST | username, password, Login |
| /dvwa/setup.php | POST | create_db |
| /dvwa/vulnerabilities/sqli/ | GET | id, Submit |
| /dvwa/vulnerabilities/xss_r/ | GET | name |
| /dvwa/vulnerabilities/exec/ | POST | ip, Submit |
| /dvwa/vulnerabilities/fi/ | GET | page |
| /dvwa/vulnerabilities/sqli_blind/ | GET | id, Submit |
| /dvwa/phpinfo.php | GET | — (information disclosure) |

`phpinfo.php` being publicly accessible is itself a finding — it exposes PHP version, server configuration, and environment variables.

---

## Authentication vs Authorization

These are two distinct security checks that fail independently.

**Authentication (AuthN)** — verifies identity. Answers: *who are you?* Occurs once at login. Example: submitting username and password.

**Authorization (AuthZ)** — verifies permissions. Answers: *what are you allowed to do?* Occurs on every subsequent request. Example: checking whether the authenticated user can access `/admin`.

**How they fail differently:**

- Stolen session cookie → AuthN bypass. The server receives a valid session token and believes the attacker is the legitimate user. AuthZ is never reached.
- IDOR (Insecure Direct Object Reference) → AuthZ bypass. The attacker is legitimately authenticated but accesses another user's resource by manipulating a parameter: `/user/1337/invoice` → `/user/1338/invoice`.

---

## Sessions and Statefulness

HTTP is stateless — every request is anonymous by default. Sessions simulate statefulness:

1. User submits credentials → server verifies → creates a session server-side
2. Server responds with `Set-Cookie: PHPSESSID=abc123`
3. Browser stores the cookie and sends it automatically on every subsequent request
4. Server looks up the session ID and identifies the user

The cookie is a key. The actual session data lives server-side.

**JWT alternative:** Instead of a server-side session store, the server issues a signed token containing user data. The server verifies the signature on each request — stateless by design. Common in APIs.

---

## DNS Resolution

DNS translates human-readable domain names into machine-routable IP addresses.

**Resolution path:**
```
Browser cache → /etc/hosts → Recursive Resolver → Root Nameserver 
→ TLD Nameserver → Authoritative Nameserver → IP returned
```

Each step only occurs if the previous cache missed. Most lookups resolve at the recursive resolver.

**Practical — dig output for google.com:**
```
;; QUESTION SECTION:
;google.com.            IN    A

;; ANSWER SECTION:
google.com.    129    IN    A    142.250.184.110

;; SERVER: 192.168.0.1#53
;; Query time: 7 msec
```

- Query sent to local router (`192.168.0.1`) on port 53
- A record (IPv4) returned: `142.250.184.110`
- TTL of 129 seconds — cached answer expires after this
- Query resolved in 7ms

**Security relevance:**
- DNS spoofing — poisoning a resolver's cache with a fake IP to redirect traffic
- DNS enumeration — querying subdomains (`mail.target.com`, `dev.target.com`, `vpn.target.com`) to map a target's infrastructure

---

## Key Takeaways

- Every HTTP conversation has a defined structure. Understanding it is a prerequisite to manipulating it.
- Headers carry security-critical metadata in both directions.
- A proxy like ZAP makes the invisible visible — every field, parameter, and header becomes accessible and modifiable.
- Attack surface mapping must precede vulnerability testing. You cannot test what you have not found.
- AuthN and AuthZ are separate mechanisms and fail in different ways.
- DNS is a foundational protocol with its own attack surface.
