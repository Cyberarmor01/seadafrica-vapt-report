# Network Scanning and Vulnerability Assessment

## 1. Network Scanning

Network scanning is the process of identifying all active devices, open ports, and services on a network in order to detect risks before attackers do. It is about discovering, mapping, and securing every device on a network.

### Tools Used for Network Scanning

**Discovery — who is on the network (find active devices/IPs)**

- **arp-scan** — fastest option for a local LAN:
  ```
  sudo arp-scan --localnet
  sudo arp-scan --interface=eth0 --localnet
  ```
- **nmap ping sweep** (`nmap -sn <IP address>`) — `-sn` means "no port scan." It only pings to see which hosts are up, using ICMP ping, ARP requests on the LAN, and TCP probes to ports 443/80:
  ```
  nmap -sn 10.10.10.7/24   {target IP range}
  ```

**Port and service scanning — what is running: find open ports, services, and OS**

- **Nmap** — the standard tool for network scanning. Service version detection probes open ports to identify what is running (e.g., Apache httpd 2.4.52):
  ```
  nmap -sV -O 10.10.10.7
  ```
- **ZMap** — a different tool from Nmap, purpose-built to scan the entire internet quickly. It is fast and stateless, well suited to scanning all of IPv4 on a single port (e.g., it can scan the whole internet on port 80 in about 45 minutes).
- **Intense scan plus UDP:**
  ```
  nmap -sS -sU -T4 -v 10.10.10.7/24
  ```
- **Ping scan:**
  ```
  nmap -sn 10.10.10.7/24
  ```
- **Regular scan:**
  ```
  nmap 10.10.10.7/24
  ```

---

## Full-Path VAPT Case Study: From OSINT to CVE-2026-93903 (CVSS 9.4) on a Live NGO Infrastructure

> This assessment was performed using only passive and non-intrusive techniques for academic purposes. No exploitation was attempted. Findings will be disclosed responsibly.

## 2. Vulnerability Scanning

A vulnerability is a weakness or flaw in a system, network, application, or process that can be exploited by a threat to cause harm.

**Concept of vulnerability — threat, vulnerability, and impact**

- **Threat:** something that wants to exploit it (hackers, malware, etc.)
- **Vulnerability:** the weakness itself
- **Impact:** what happens if it is exploited (data loss, downtime, etc.)

**Types of vulnerability:** network, system/OS, application, human, physical.

Scanning, penetration testing, vulnerability scanners, and bug bounty programs are the main ways vulnerabilities are found.

**Target:** seadafrica.org

### Risk Description

The vulnerabilities below exposed the target application to the risk of unauthorized access to confidential data and possible denial-of-service attacks. An attacker could search for an existing exploit, or develop one, for any of these vulnerabilities and use it against the system.

**SSH service exposed to the internet** — An SSH service using username/password authentication is publicly accessible. Administrators often use remote administration protocols to manage devices such as switches and routers; exposing these services to the internet increases the attack surface and creates opportunities for attack against the organization.

**FTP file service exposed** — The FTP service is publicly accessible. FTP allows clients to upload and download files but does not encrypt data in transit, leaving all transferred data exposed in plain text.

**Missing DMARC policy**

| Field | Value |
|---|---|
| Severity | Medium |
| CVSS | 5.3 |
| Category | Email Security / Authentication |

During this assessment, it was identified that the target domain does not have a DMARC TXT record configured in DNS. DMARC (Domain-based Message Authentication, Reporting and Conformance) works alongside SPF and DKIM to define how receiving mail servers should handle messages that fail authentication checks.

---

## 3. Active Vulnerability Scan — Nikto

| Field | Value |
|---|---|
| Tool | Nikto v2.6.0 (web server scanner) |
| Command | `nikto -h seadafrica.org` |
| Platform | Kali Linux (kali@kali) |

An active vulnerability assessment was performed using Nikto v2.6.0 against seadafrica.org. The scan identified the target IP 147.124.214.22 listening on port 80, with the platform header obfuscated (shown as "Unknown"), consistent with LiteSpeed WAF header filtering previously identified via WAFW00F. The exposure of port 80 indicates a potential lack of strict HTTPS enforcement, while the header obfuscation demonstrates active defensive measures.

**What Nikto does:** Nikto is an active scanner. Unlike SpiderFoot (passive) and WAFW00F (2 requests), Nikto sends thousands of requests to check for outdated software, default files (e.g. `/admin/`, `/phpmyadmin/`), misconfigurations, and insecure headers.

**Key findings:**

- **Target IP: 147.124.214.22** — the real server IP.
- **Target port: 80** — HTTP (unencrypted). The earlier WAFW00F scan was run against `https://` (port 443). This confirms the site still listens on port 80, likely redirecting to 443. If it is not forced to HTTPS, it remains vulnerable to downgrade/MITM attacks.

---

## 4. IP Reputation & Infrastructure Intelligence

| Field | Value |
|---|---|
| Tool | AbuseIPDB (abuseipdb.com/check/147.124.214.22) |
| Target IP | 147.124.214.22 (from the Nikto scan) |
| Hostname | seadafrica.org |
| Verdict | Low risk / clean — 0% confidence |

Infrastructure analysis via AbuseIPDB confirmed that the origin IP identified by Nikto is hosted by Majestic Hosting Solutions (AS396073) in Dallas, Texas, with reverse DNS `standard1.doveserver.com`. With an abuse confidence score of 0% and only one historical report from four weeks ago, the IP is considered clean and not currently part of botnet or spam infrastructure. This indicates the observed issues (open port 80 and an outdated LiteSpeed version) stem from misconfiguration rather than compromise — though the clean reputation does not offset the critical CVE-2026-93903 (CVSS 9.4) finding, which could allow full system takeover if exploited.

**Abuse history**

- Total reports: 1, from 1 distinct source
- Last reported: 4 weeks ago
- Confidence: 0% (green)

A 0% confidence score means AbuseIPDB does not consider this IP abusive. The single report from four weeks ago is marked as an "old report" — likely a false positive, or the IP was previously assigned to a different party and has since been cleaned. For comparison, a score of 80–100% with 50+ reports would indicate the server is actively sending spam, brute-forcing, or hosting malware; here it is clean.

---

## 5. OSINT / Reconnaissance Scan — SpiderFoot

| Field | Value |
|---|---|
| Tool | SpiderFoot (open-source OSINT automation) |
| Target | seadafrica.org |
| Date | 2026-10-07 |
| Status | Completed |

SpiderFoot is used for the reconnaissance phase. It does not attack the target; it gathers publicly available information about the domain from over 100 public sources. This scan was conducted purely for information gathering, with no active exploitation.

**Key findings — 17 data types discovered:**

*Affiliate information (high exposure)*
- Affiliate company names: 2 related companies found, indicating business affiliations.
- Affiliate domain names: 2 unique / 4 total related domains, meaning the organization uses multiple domains.
- Affiliate email addresses: 6 found — the most significant finding. Public emails can be used for phishing if not protected.
- Affiliate internet names: 3 unique / 5 total related hostnames/subdomains discovered.

*Infrastructure information*
- DNS SPF record, DNS TXT record, and email gateway (MX records): one of each found, showing email security configuration is present (strictness still needs review).
- HTTP headers: 1 captured, which can reveal server type and outdated software.
- Domain WHOIS and registrar: publicly available registration information.

*Organizational information*
- Company name, country, and affiliate description/category: business categorization is publicly discoverable.

| Category | Type | Unique / Total | Risk |
|---|---|---|---|
| Affiliate exposure | Affiliate email address | 6 / 6 | High |
| | Affiliate internet name | 3 / 5 | Medium |
| | Affiliate domain name | 2 / 4 | Medium |
| | Affiliate company name | 2 / 2 | Low |
| | Affiliate domain WHOIS | 2 / 2 | Low |
| Infrastructure | DNS SPF / TXT / MX records | 1 each | Info |
| | HTTP headers | 1 | Info |
| | Domain name / WHOIS / registrar | 1 each | Info |

**Most critical:** affiliate email addresses (6 found). Public emails are the top vector for business email compromise and spear-phishing.

**Infrastructure leakage:** DNS SPF/TXT/MX records and HTTP headers indicate that the site's email security posture and web server details are discoverable.

### Risk Analysis

This is an information-disclosure finding at the OSINT level. It is not a direct vulnerability like SQL injection, but it enables other attacks:

- **Phishing:** 6 affiliate emails are potential targets.
- **Attack-surface expansion:** 2 extra affiliate domains and 3 internet names represent additional points of attack.
- **Technology fingerprinting:** DNS and HTTP header data helps an attacker select appropriate exploits.

### Recommendations

- Reduce public footprint — use WHOIS privacy protection where possible.
- Review the SPF record to ensure a hard fail (`-all`) rather than a soft fail (`~all`); ensure SPF is `v=spf1 ... -all`, and enable DKIM and DMARC.
- Remove unnecessary emails from public-facing pages.
- Review HTTP security headers — implement `X-Frame-Options`, CSP, and HSTS.
- Train staff on phishing awareness, since emails are publicly discoverable.

---

## 6. Web Application Firewall (WAF) Detection

| Field | Value |
|---|---|
| Target | https://seadafrica.org |
| Tool | WAFW00F v2.3.2 |
| Command | `wafw00f seadafrica.org` |
| Platform | Kali Linux (kali@kali) |

A WAF fingerprinting scan was conducted using WAFW00F 2.3.2. The scan was passive, using only 2 requests. The result confirms that https://seadafrica.org sits behind a LiteSpeed Technologies WAF, meaning the application layer is filtered for common OWASP Top 10 attacks. This aligns with the SpiderFoot findings, where infrastructure exposure was low (only 1 HTTP header found), suggesting server header obfuscation by the WAF.

**Verdict: WAF detected.** LiteSpeed is not just a web server — its built-in WAF filters malicious requests (SQLi, XSS, LFI, etc.) before they reach the application.

**What WAFW00F did:** it sent only 2 requests, making it a non-intrusive tool — a baseline request, then a payload request to observe whether it was blocked and how the headers changed. It detects the WAF by analyzing response headers and behavior, not by attacking the target.

**Security implications**

- *For an attacker:* the presence of a LiteSpeed WAF means direct exploitation attempts will likely be filtered, pushing an attacker toward other vectors — such as the two affiliate domains identified by SpiderFoot.
- *For the defender (seadafrica.org):* having WAF protection in place is a positive sign. However, a WAF alone is not sufficient — if the origin IP is exposed, an attacker can bypass the WAF by connecting to the origin server directly. (Several S3 buckets were also identified, e.g. `chat.seadafrica.org` and `seadafricaorg-production.s3-ap-northeast-2.amazonaws.com`; these should be checked for public access.)

**Recommendations**

- Keep the LiteSpeed WAF enabled and up to date.
- Ensure the WAF is running in blocking mode, not detection-only mode.
- Hide the origin IP behind a reverse proxy/CDN (e.g. Cloudflare) and restrict direct access to the origin and to S3 buckets.
- Complement the WAF with secure coding practices — a WAF filters traffic but does not fix underlying vulnerabilities.

---

## 7. CVE Intelligence — LiteSpeed

| Field | Value |
|---|---|
| Target technology | LiteSpeed (identified via WAFW00F on seadafrica.org) |
| Source | CVE.org (MITRE) |
| Search query | LiteSpeed |
| Results | 48 CVEs found (showing 1–25) |

After identifying LiteSpeed as the WAF/web server technology, the official CVE database was searched to check for known public vulnerabilities that could affect the target if unpatched.

The search returned 48 results. Analysis of the top results revealed critical vulnerabilities in LSWS < 6.3.7 (CVE-2026-93903) and LiteSpeed Cache ≤ 7.9 (CVE-2026-84761, SSRF; CVE-2026-76579, XSS). These vulnerabilities carry a risk of server-side request forgery and cross-site scripting, which is especially relevant given the target's use of AWS S3 buckets identified during enumeration.

**Recommended passive follow-up:** check HTTP response headers (e.g. via the SpiderFoot results) for version indicators such as `X-LiteSpeed-Cache` or a server version string, without attempting active exploitation.

**Patch guidance:** update LSWS to ≥ 6.3.7 build 1, and the LiteSpeed Cache plugin to > 7.9.

### CVE-2026-93903 — Detail

The most critical vulnerability identified during this assessment is CVE-2026-93903, affecting LiteSpeed Web Server before 6.3.7 build 1. Classified as CWE-174 (Double Decoding), it carries a CVSS 4.0 score of 9.4 (Critical). The vulnerability stems from mishandling of internal redirect URL validation, which can enable WAF bypass via double-encoded payloads. Because seadafrica.org is confirmed to sit behind a LiteSpeed WAF, this CVE is directly applicable: if the server version is outdated, an unauthenticated remote attacker could bypass access controls and achieve high impact to confidentiality, integrity, and availability.

**Severity Analysis — Critical**

| Field | Value |
|---|---|
| CVSS Score | 9.4 / 10 — Critical |
| CVSS Version | 4.0 |
| Vector | `CVSS:4.0/AV:N/AC:L/AT:N/PR:L/UI:N/VC:H/VI:H/VA:H/SC:H/SI:H/SA:H` |

**Vector breakdown**

- `AV:N` — Attack Vector: Network. Exploitable remotely over the internet.
- `AC:L` — Attack Complexity: Low. No special conditions required.
- `AT:N` — Attack Requirements: None.
- `PR:L` — Privileges Required: Low (a low-privilege account, or in some cases none, depending on configuration).
- `UI:N` — User Interaction: None required.
- `VC:H / VI:H / VA:H` — Confidentiality, Integrity, Availability Impact: High (full compromise).
- `SC:H / SI:H / SA:H` — Subsequent System Impact: High (can affect systems beyond the vulnerable component).

This is rated 9.4/Critical because it is remotely exploitable, low complexity, and can lead to full system compromise plus impact on downstream systems, such as the S3 buckets identified above.

**Recommendation:** upgrade to LSWS 6.3.7 build 1 or later immediately, and implement strict input validation that decodes input only once.

---

## 8. WHOIS and DNS Reconnaissance

**NS records:** `ns5.doveserver.com`, `ns6.doveserver.com` — NameHero/DoveServer shared hosting, which is consistent with the hosting-related issues identified elsewhere in this report.

**MX record:** `0 seadafrica.org` (mail is hosted on the same server as the website). This means that if the website is compromised, email is also at risk — a single point of failure.

**TXT / SPF record:**
```
v=spf1 ip4:147.124.214.22 ip4:208.115.219.162 +a +mx
+ip4:67.220.184.146 +ip4:209.205.200.90
include:relay.mailchannels.net ~all
```

**Commands used:**
```
nslookup seadafrica.org
nslookup seadafrica.org 8.8.8.8
dig seadafrica.org ANY
```

**Findings:** the domain resolves to 147.124.214.22, using DoveServer name servers (ns5/ns6.doveserver.com). The MX record points to the same host, indicating shared hosting for both web and mail. The SPF record includes a MailChannels relay, showing an attempt at anti-spoofing protection, though it uses a soft fail (`~all`) rather than a hard fail.

**SOC relevance:** DNS data is critical for detecting domain hijacking and phishing infrastructure. Single-host dependency increases the impact of any compromise.

---

## 9. Vulnerability Scan Summary

The scan reports an overall risk level of **HIGH**, driven by one High-severity vulnerability found on port 22/tcp. The scan performed 27 tests in 4 minutes 45 seconds. The scanner notes this was a "Limited / Light Scan" only, meaning it did not perform deep checks across the full vulnerability database (20,000+ checks). The finding is marked **unconfirmed**, meaning it is based on the service version banner rather than active exploitation.

**Risk distribution**

| Severity | Count | Meaning |
|---|---|---|
| Critical | 0 | No immediate system-takeover flaw identified |
| High | 1 | Requires urgent patching |
| Medium | 1 | Needs review |
| Low | 6 | Hardening issues, information disclosure |
| Info | 7 | Best-practice recommendations |

### Finding #1: Outdated OpenSSH 8.0 on Port 22/tcp — High

| Field | Details |
|---|---|
| Port | 22/tcp — SSH, used for remote Linux administration |
| Service detected | OpenSSH 8.0 — released April 2019, ~7 years outdated |
| CVE | CVE-2023-38408 |
| CVSS Score | 9.8 / 10 — Critical |
| EPSS Score | 0.76768 — ~76% probability of exploitation in the wild |
| EPSS Percentile | 0.99491 — top 1% of most likely exploited vulnerabilities |
| CISA KEV | Not currently listed in the CISA Known Exploited Vulnerabilities catalog |
| Public exploit | Not required — the issue is an abuse of a legitimate feature |

**Technical summary:** the PKCS#11 feature in `ssh-agent` in OpenSSH before 9.3p2 has an insufficiently trustworthy search path, which can lead to remote code execution if an agent is forwarded to an attacker-controlled system — code in `/usr/lib` is not necessarily safe to load into `ssh-agent`. This stems from an incomplete fix for CVE-2016-10009.

### Analysis

1. **Severe version obsolescence** — OpenSSH 8.0 is end-of-life. Running a 2019 release in 2026 exposes the server to 30+ CVEs patched since, indicating a lack of patch management.
2. **High-risk configuration (agent forwarding)** — CVE-2023-38408 is only exploitable if `ForwardAgent yes` is enabled and the user connects to a malicious server. This setting is often enabled for convenience without a full understanding of the risk.
3. **Port 22 exposed to the internet** — SSH on a public-facing server should not be open to `0.0.0.0/0`; access should be restricted to admin IPs or a VPN.
4. **Unconfirmed result, not a false sense of security** — the finding is based on the banner string `SSH-2.0-OpenSSH_8.0` and does not confirm whether the vulnerable PKCS#11 code path is actually compiled in. Given a CVSS of 9.8 and an EPSS in the 99th percentile, it remains high priority regardless.
5. **Overall hygiene** — 1 Medium, 6 Low, and 7 Info findings exist beyond this one. In a typical assessment, Medium findings are often outdated TLS or missing security headers (ports 80/443); Low findings are typically weak SSH algorithms or missing cookie flags.

### Remediation

| Problem | Immediate Action | Long-Term Action |
|---|---|---|
| CVE-2023-38408 | Upgrade OpenSSH to 9.8p1 or later: `sudo apt update && sudo apt install openssh-server`; verify with `ssh -V` | Enable automatic security updates: `sudo apt install unattended-upgrades` |
| Agent forwarding risk | Disable in `/etc/ssh/sshd_config`: `AllowAgentForwarding no`; on the client, in `/etc/ssh/ssh_config`: `ForwardAgent no` | Train admins to use `ProxyJump` instead of agent forwarding; remove the PKCS#11 provider if unused |
| Port 22 public exposure | Restrict with a firewall: `sudo ufw allow from <ADMIN_IP> to any port 22` then `sudo ufw deny 22/tcp` | Move SSH behind a VPN or zero-trust access (e.g. Cloudflare Tunnel, Tailscale); change the default port and add fail2ban |
| Outdated system | Run a full system update and a deeper vulnerability scan (OpenVAS/Nessus) | Implement a patch-management policy (critical patches within 48 hours); use configuration management (e.g. Ansible) for fleet patching |
| Light-scan limitation | Re-run a deep scan to cover the full vulnerability database | Schedule weekly deep scans and a monthly penetration test |

### Recommended Action Plan for seadafrica.org

**Day 1 — immediate:**
- Confirm the OpenSSH version with `sshd -V` on the server.
- If it is confirmed as 8.0, upgrade immediately (take a snapshot/backup first).
- Disable agent forwarding and restart the service: `sudo systemctl restart sshd`.

**Day 2 — hardening:**
- Apply rate-limiting and brute-force protection: UFW rate limiting on port 22/tcp plus Fail2Ban (e.g., 3 failed attempts = 1-hour ban).
- Disable weak key-exchange algorithms in `sshd_config`: `KexAlgorithms -diffie-hellman-group1-sha1`.
- Re-scan to confirm the finding is resolved.

### Conclusion

This High-risk finding should not be dismissed as a false alarm. Even though the scan result is unconfirmed, the combination of a 9.8 CVSS score, a 99th-percentile EPSS score, and a 7-year-old OpenSSH version indicates a genuinely risky configuration. The remediation is both low-cost and straightforward: upgrade OpenSSH and restrict access to port 22.

This finding also validates the value of network monitoring: without an IDS such as Snort in place, an administrator would have no visibility into active exploitation attempts against this SSH service. Combining Snort, UFW, and Fail2Ban provides both detection and prevention.
