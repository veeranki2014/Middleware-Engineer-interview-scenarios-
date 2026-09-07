# SiteMinder Integration — WebSphere & WebLogic

---

## What SiteMinder Actually Is

CA SiteMinder (now Broadcom/Symantec SiteMinder, sometimes branded CA SSO) is a centralized Web Access Management (WAM) platform — unlike PingID (pure MFA layered on SAML), SiteMinder handles authentication, authorization, and SSO policy enforcement as a full platform. It has three core components:

| Component | Role |
|---|---|
| **Policy Server** | Central brain — stores policies, authenticates against LDAP/AD, makes authorization decisions |
| **Web Agent** | Sits on the web server (IHS/Apache/IIS) — intercepts requests, checks for the `SMSESSION` cookie |
| **Application Server Agent (ASA)** | Sits inside WAS/WebLogic — validates the SMSESSION cookie at the app server layer and asserts identity into the container's security context |

---

## The Authentication Flow

```
Browser → IHS/Apache (Web Agent installed)
        → No SMSESSION cookie? → Challenge for credentials
        → Web Agent sends creds to Policy Server
        → Policy Server authenticates against LDAP/AD
        → Policy Server issues SMSESSION cookie
        → Web Agent sets cookie, forwards request to App Server
        → App Server Agent (ASA) validates SMSESSION cookie
        → ASA asserts identity into WAS/WebLogic security context
        → Application processes request with authenticated user
```

---

## WebSphere Integration — Trust Association Interceptor (TAI)

WebSphere integrates SiteMinder using a Trust Association Interceptor — the same mechanism used for SAML SSO, just a SiteMinder-specific TAI implementation.

### Step 1: Install the SiteMinder Web Agent on IHS

Standard Web Agent install, registers a trusted host with the Policy Server via `SmHost.conf`.

### Step 2: Install the Application Server Agent (ASA) on WAS

The ASA installer deploys SiteMinder's TAI classes into WAS and registers the trust association interceptor.

### Step 3: Configure the TAI in WAS Admin Console

```
Security → Global security → Web and SIP security → Trust association → Interceptors
→ Add: com.netegrity.sso.was.WASSiteMinderTAI (or vendor-current class name)
```

Custom properties typically required:
```
SiteMinderTAI.SessionCookieName = SMSESSION
SiteMinderTAI.RemoteUserName = SM_USER
SiteMinderTAI.LoginID = SM_USERDN
```

### Step 4: Configure the Policy Server Resource Adapter

The SiteMinder Policy Server Resource Adapter validates the SMSESSION cookie, after which SiteMinder creates the user context.

```
WebSphere Admin Console:
Applications → Enterprise Applications → PolicyServerRA
→ Resource Adapter → Custom Properties
  PolicyServerHost = policyserver.example.com
  PolicyServerPort = 44441
  AgentName = WAS-Agent-01
  SharedSecret = <configured secret>
```

### Step 5: Restart WAS

Required for the TAI and resource adapter changes to take effect.

---

## WebLogic Integration — Identity Asserter (Not TAI)

WebLogic uses the Identity Asserter module that's part of the SiteMinder Application Server Agent, configured through the Security Realm — the same slot used for a SAML2IdentityAsserter, just a SiteMinder-specific implementation.

### Step 1: Install the SiteMinder Agent for WebLogic

Register a Trusted Host during install using an `SmHost.conf` file — the same host config file used for the WebSphere ASA, and can be shared/reused if generated for the same environment.

```bash
# Silent/console install example
./ca-wl-agent-install.bin -i console
```

### Step 2: Configure the Identity Asserter in the Security Realm

```
WebLogic Admin Console:
Domain → Security Realms → myrealm → Providers → Authentication
→ New → SiteMinderIdentityAsserter (or CA-provided asserter class)
```

Set:
```
Active Types: SM_USER
Provider Specific: Policy Server host, port, shared secret
```

### Step 3: Provider Ordering (Critical)

The Identity Asserter must be placed above the default authenticator in the provider list, since it's asserting an already-authenticated identity rather than validating a username/password itself.

### Step 4: Configure the Proxy Plug-in on the Web Tier

WebLogic requires the SiteMinder Web Agent's proxy plug-in to forward requests correctly from the web server into WebLogic — a separate install step from the ASA/Identity Asserter.

### Step 5: Restart WebLogic

Restarting the server for installation changes to take effect is required after agent configuration.

---

## Side-by-Side Comparison

| Aspect | WebSphere | WebLogic |
|---|---|---|
| Trust mechanism | Trust Association Interceptor (TAI) | Identity Asserter (Security Realm provider) |
| Host config file | `SmHost.conf` | `SmHost.conf` (same format) |
| Cookie validated | `SMSESSION` | `SMSESSION` |
| Config location | Global Security → Trust Association | Security Realm → Authentication Providers |
| Provider order matters? | N/A (single TAI) | Yes — Identity Asserter must precede default authenticator |
| Additional component | Policy Server Resource Adapter (`PolicyServerRA`) | Proxy plug-in on the web tier |

---

## SiteMinder vs PingFederate/SAML Approach

| | SiteMinder | PingFederate (SAML) |
|---|---|---|
| Protocol | Proprietary cookie (`SMSESSION`) + Policy Server API | Standards-based SAML 2.0 assertion |
| Where auth decision lives | Policy Server (centralized, proprietary) | IdP (standards-compliant, federatable across orgs) |
| Web tier component | SiteMinder Web Agent on IHS/Apache | Just a reverse proxy — no vendor agent required |
| Modern fit | Legacy enterprise WAM, still common in large regulated shops | Better fit for cloud/cross-domain federation, MFA vendors like PingID |

---

## Interview Soundbite

> SiteMinder is a full web access management platform, not just an authentication protocol — it has a Policy Server as the central decision-maker, a Web Agent on the HTTP tier that intercepts requests and manages the SMSESSION cookie, and an Application Server Agent that plugs into the app server itself. On WebSphere, that plugs in as a Trust Association Interceptor, conceptually the same mechanism used for SAML-based SSO with PingFederate, just a SiteMinder-specific TAI implementation. On WebLogic, it's an Identity Asserter registered in the Security Realm, and just like with SAML there, provider ordering matters — the Identity Asserter has to sit above the default authenticator since it's asserting an already-validated identity rather than checking a password itself.