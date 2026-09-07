# Application Security Vulnerability Assessment & Remediation

---

## The General Process

As a middleware/infra engineer, most vulnerability work falls into three buckets: SSL/TLS hardening, web server/app server configuration hardening, and patching known CVEs. Vulnerabilities usually surface from a Qualys/Nessus scan, a pen test finding, or an internal security audit — then remediation happens at the IHS/WAS/WebLogic config layer.

## Common Ways Vulnerabilities Get Identified

```bash
# SSL/TLS scan of a live endpoint
nmap --script ssl-enum-ciphers -p 443 app.example.com

# Or using testssl.sh (comprehensive TLS assessment tool)
./testssl.sh https://app.example.com

# Quick check of supported protocols
openssl s_client -connect app.example.com:443 -tls1
openssl s_client -connect app.example.com:443 -tls1_1
```

Most orgs run **Qualys, Nessus, or Rapid7** scans on a schedule, and those reports land on the middleware team's desk for remediation.

---

## Example 1: Weak SSL/TLS Protocol and Cipher Vulnerability

**The finding:** A Qualys scan flagged that the app server accepted **TLS 1.0/1.1** and weak ciphers (RC4, 3DES), vulnerable to attacks like **POODLE**, **BEAST**, and **Sweet32**.

**Root cause:** Default IHS/WAS SSL configuration hadn't been updated in years and still allowed legacy protocol negotiation for backward compatibility with old clients that no longer existed.

**Fix — IHS side (httpd.conf):**
```apache
SSLProtocolDisable SSLv2 SSLv3 TLSv1 TLSv1.1
SSLProtocolEnable TLSv1.2
SSLCipherSpec TLSv12 !aNULL:!eNULL:!EXPORT:!DES:!RC4:!3DES:!MD5:!PSK
```

**Fix — WebSphere side (via Admin Console):**
```
Security → SSL certificate and key management → SSL configurations → [NodeDefaultSSLSettings]
→ Quality of protection (QoP) → Protocol: TLSv1.2
→ Enabled cipher suites: remove weak ciphers, keep only AES-GCM/AES-CBC 256-bit suites
```

**Validation after fix:**
```bash
openssl s_client -connect app.example.com:443 -tls1_1
# Should now return: "no protocols available" / handshake failure — confirms it's disabled
```

---

## Example 2: Expired/Weak Certificate Chain Vulnerability

**The finding:** A cert using **SHA-1 signature algorithm** (deprecated, collision-vulnerable) or a cert with an incomplete chain causing browser trust warnings.

**Fix:** Regenerated the certificate with **SHA-256** signing, re-added the full intermediate chain into the keystore.

```bash
gskcmd -cert -create -db keyfile.kdb -pw <pw> -label newcert \
  -dn "CN=app.example.com,O=MyOrg,C=US" -sigalg SHA256WithRSA -size 2048
```

---

## Example 3: HTTP Security Headers Missing

**The finding:** Pen test flagged missing security headers — `X-Frame-Options`, `Strict-Transport-Security`, `X-Content-Type-Options` — exposing the app to **clickjacking** and **MIME-sniffing attacks**.

**Fix — added at the IHS/reverse-proxy layer** (centralizing it here instead of every individual app):
```apache
Header always set X-Frame-Options "SAMEORIGIN"
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
Header always set X-Content-Type-Options "nosniff"
Header always set Content-Security-Policy "default-src 'self'"
```

---

## Example 4: Session Fixation / Insecure Session Cookie Vulnerability

**The finding:** Session cookies missing the `Secure` and `HttpOnly` flags — exposing sessions to interception over non-HTTPS or theft via XSS.

**Fix — WebSphere session management config:**
```
Servers → Application Servers → [server] → Session Management
→ Enable "Set session cookies to HTTPOnly" = true
→ Enable "Restrict cookies to HTTPS sessions" = true
```

---

## Example 5: A Known CVE Requiring a Fixpack

**The finding:** A vulnerability scan flagged a specific IBM-published CVE affecting the installed WAS version (e.g., a deserialization vulnerability in a bundled component).

**Fix:** Followed the standard patching runbook — checked IBM Fix Central for the CVE's corresponding APAR/interim fix, applied it via `imcl`, validated, and documented in the change ticket.

---

## How to Answer "Tell Me About a Vulnerability You Fixed" (STAR-lite Format)

> **Situation:** During a quarterly Qualys scan, production IHS servers were found still accepting TLS 1.0 and 1.1, along with several weak ciphers like RC4 and 3DES.
>
> **Task:** Disable the legacy protocols and ciphers without breaking any client integrations that might still depend on them.
>
> **Action:** Ran a `testssl.sh` scan to get the full cipher/protocol inventory, cross-checked with client teams whether any integration still required TLS 1.1, confirmed none did, then updated `httpd.conf` to disable SSLv3 through TLS 1.1 and restrict cipher suites to strong AES-GCM options only, and made the equivalent change on the WAS SSL configuration for direct app server access. Tested in QA first, then rolled it through change management to prod.
>
> **Result:** Re-scan showed the vulnerability closed, no client integration broke, and the change was documented as the new SSL baseline standard going forward for all new server builds.

---

## Interview Soundbite

> The vulnerabilities I deal with most often fall into SSL/TLS hardening — disabling weak protocols and ciphers — certificate lifecycle issues like weak signature algorithms, missing security headers, insecure session cookie flags, and applying IBM fixpacks for published CVEs. My approach is always: confirm the finding with a scan tool, understand what's actually using the weak config before disabling it, make the change in QA first, validate with a rescan, then promote through change management to prod.