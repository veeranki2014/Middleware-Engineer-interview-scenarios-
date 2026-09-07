# MFA Integration for WebSphere & WebLogic — Authentication vs Authorization

---

## Core Concept

Neither WebSphere nor WebLogic has native MFA capability. Both are Java EE containers designed to handle *authentication delegation* and *authorization enforcement* — they were never built to prompt a user for a push notification or OTP themselves. MFA always gets bolted on by delegating the entire login step to an external Identity Provider (IdP) like PingFederate, Okta, or ADFS, which owns the actual MFA challenge (PingID, Okta Verify, etc.).

> MFA isn't implemented inside the app server — it's implemented at the IdP, and the app server is configured to trust and consume the IdP's assertion via SAML or OIDC.

## Authentication vs Authorization

| | Authentication | Authorization |
|---|---|---|
| Question it answers | "Who are you?" | "What are you allowed to do?" |
| Where it happens | IdP (PingFederate/Okta) + MFA adapter | App server, using roles/groups from the assertion |
| Mechanism | SAML assertion / OIDC token, validated by TAI or SSO filter | `web.xml` security-constraints mapped to roles; WebLogic uses Deployment Descriptors + Security Realm |
| Example | User completes LDAP password + PingID push | User is in `AppAdmins` group → granted access to `/admin/*` |

MFA is purely an authentication-strengthening step. Once it succeeds, authorization proceeds exactly as it always did (role mapping from LDAP/AD groups), completely unchanged.

## Architecture Pattern (Same Shape for Both Servers)

```
User → App Server (SP) → Redirect to IdP (PingFederate/Okta)
                        → IdP: 1st factor (LDAP/AD password)
                        → IdP: 2nd factor (PingID push / OTP / FIDO2)
                        → IdP issues SAML assertion or OIDC token
     ← App Server validates assertion signature/trust
     ← App Server maps assertion attributes → J2EE/WebLogic roles
     ← Session established, authorization proceeds normally
```

The only real difference between WAS and WebLogic is which internal mechanism accepts and validates that assertion.

---

## WebSphere Application Server (v9) — Mechanism: SAML TAI

**Trust Association Interceptor (TAI)** is the WAS component that intercepts inbound requests, recognizes they've already been authenticated by an external trusted party, and skips WAS's normal login form.

**Enable TAI:**
```
Security → Global security → Web and SIP security → Trust association → Interceptors
→ com.ibm.ws.security.web.saml.SAMLTAI
```

**Key config pieces:**
- SP metadata exported from WAS (entity ID, ACS URL, signing cert) → given to PingFederate
- IdP metadata imported into WAS from PingFederate → establishes trust
- Assertion attributes mapped to WAS security roles via `ibm-application-bnd.xml`

```python
# wsadmin example
AdminTask.configureSAMLTai('[-ssoId sp1 -enabled true]')
AdminTask.importSAMLIdpMetadata('[-ssoId sp1 -metadataFile idp-metadata.xml]')
```

**Export SP metadata:**
```bash
wsadmin.sh -lang jython -c "print AdminTask.exportSAMLSPMetadata('[-ssoId sp1 -exportCertificate true]')"
```

---

## WebLogic Server — Mechanism: SAML 2.0 Identity Assertion Provider (Security Realm)

WebLogic's equivalent concept is the **Identity Assertion Provider** inside a **Security Realm**, configured via the WebLogic Admin Console.

**Add the provider:**
```
Domain → Security Realms → myrealm → Providers → Authentication
→ New → SAML2IdentityAsserter
```

**Key config pieces:**
- SAML 2.0 General settings: define WebLogic as the SP, generate SP metadata
- SAML 2.0 Identity Provider Partner: import PingFederate's IdP metadata, establish trust
- Identity Assertion Provider order: must be placed *above* the default authenticator in the realm's provider list, since it's asserting an already-authenticated identity rather than doing username/password validation itself
- Role mapping happens via WebLogic's Default Role Mapper, tied to groups pulled from the SAML assertion (or a subsequent LDAP lookup if only a username comes back)

```
# WLST example (conceptually)
cd('/SecurityConfiguration/mydomain/Realms/myrealm')
create('SAML2IdentityAsserter', 'AuthenticationProvider')
```

---

## Side-by-Side Summary

| Aspect | WebSphere | WebLogic |
|---|---|---|
| Trust mechanism | Trust Association Interceptor (TAI) | Identity Assertion Provider |
| SAML support built-in? | Yes, since 8.5.5+ | Yes, native SAML 2.0 support in realm |
| Config location | Admin Console / `wsadmin` → Global Security | Admin Console / WLST → Security Realm |
| Metadata exchange | Export/import SAML SP & IdP metadata | Export/import SAML SP & IdP metadata via Realm SAML2 settings |
| Role mapping | `ibm-application-bnd.xml`, group→role mapping | Default Role Mapper, group→role mapping |
| Provider ordering matters? | N/A (single TAI chain) | Yes — Identity Asserter must precede default authenticator |

---

## The One-Sentence Summary (Interview Soundbite)

> Both app servers are just Service Providers in a SAML federation — MFA lives entirely at the IdP layer via PingFederate and PingID. Once the IdP hands back a signed assertion, WebSphere validates it through a Trust Association Interceptor and WebLogic through an Identity Assertion Provider in its security realm — from there, authorization proceeds exactly like a normal LDAP-based deployment, just with a stronger, MFA-backed authentication event feeding it.

---

## Practical Caveats / What Goes Wrong in Real Deployments

- **Clock skew** between the app server and the IdP breaks SAML assertion validation (assertions have tight `NotBefore`/`NotOnOrAfter` windows) — NTP sync across all hosts is a prerequisite, not an afterthought.
- **Certificate rotation** on either side (SP signing cert or IdP signing cert) silently breaks SSO if not coordinated — same cert lifecycle discipline as ikeyman/gskcmd TLS cert management, just now applied to SAML trust instead of TLS termination.
- **Session timeout mismatch** between the IdP's SSO session and the app server's local session causes confusing "re-prompts for MFA mid-session" bugs if not aligned.