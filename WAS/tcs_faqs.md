# WebSphere / Middleware Admin — Interview Notes

## 1. Tell about yourself
- Summarize years of experience, primary tech stack (WebSphere, WebLogic, IHS, Apache, JBoss), domains supported, and 1–2 major projects (migrations, upgrades, DR setups) with measurable impact.

## 2. WebSphere Build (1–5) — Build & Support
Typical build/deploy lifecycle to describe:
1. **Plan** — capacity, topology (Deployment Manager, Node Agents, clusters), naming standards.
2. **Install** — IBM Installation Manager, response files for silent install.
3. **Profile creation** — `manageprofiles.sh` for Dmgr and managed nodes; federate nodes to the cell.
4. **Configure** — JVM, data sources, JMS, security (SSL, LTPA), virtual hosts.
5. **Deploy & Support** — application deployment (EAR/WAR), health checks, patching, ongoing incident/problem management.

## 3. WebSphere vs WebLogic vs JBoss

| Feature | WebSphere | WebLogic | JBoss / WildFly |
|---|---|---|---|
| Vendor | IBM | Oracle | Red Hat |
| Admin Console | ISC (Admin Console) | WebLogic Console / WLST | Management CLI / Console |
| Scripting | `wsadmin` (Jython/Jacl) | WLST (Jython) | JBoss CLI, Groovy |
| Clustering model | Cell → Node → Server | Domain → Cluster → Managed Server | Domain/Standalone → Server Group |
| Security | LTPA tokens, SAF/RACF integration | SAML, Oracle Identity | Elytron/PicketBox |
| **2-way (mutual) authentication** | SSL config with client cert auth enabled in Quality of Protection (QoP) settings; requires client cert trusted in keystore | Configured via SSL "Two Way Client Cert Behavior" on server SSL config | Configured via `verify-client="REQUIRED"` in Undertow HTTPS listener |
| Transactions | JTA via WebSphere Transaction Manager | JTA via WebLogic Transaction Manager | Narayana Transaction Manager |
| Typical use | Enterprise/legacy IBM shops | Oracle Fusion Middleware stack | Open-source/lightweight Java EE |

**2-way SSL (mutual auth)** in general: both client and server present certificates; server validates client cert against a trusted CA in its truststore before allowing the connection — used for service-to-service or high-security app access.

## 4. Why do we need a web server if there's no static web content?
Even without static content, a web server (Apache/IHS/Nginx) is used for:
- **Reverse proxy / load balancing** requests to app servers (via plugins like the WebSphere plugin, mod_wl for WebLogic).
- **SSL termination / offloading** so app servers don't handle raw TLS.
- **Session affinity** (sticky sessions) routing to the correct app server.
- **Security layer** — hides app server topology, can apply WAF rules, IP filtering.
- **Virtual hosting** — multiple domains/URLs routed from one entry point.

## 5. Five applications, Server3 (one JVM) not working — "HA Manager" issue
Troubleshooting approach:
- Check `SystemOut.log` / `SystemErr.log` on Server3 for HAManager (High Availability Manager) exceptions — common with **DCS (Data Replication Service)** communication failures between cluster members.
- Verify **DCS transport channel** connectivity (multicast/unicast) between nodes — firewall or network partition often the root cause.
- Check core group bridge configuration if servers span multiple core groups.
- Validate that the JVM isn't resource-starved (heap/CPU) causing HAManager heartbeat timeouts.
- Restart the affected JVM; if recurring, review core group policy and DCS logs (`ffdc` files) for the specific failure.

## 6. Auto-restart of WebSphere on OS reboot
- Configure via `/etc/init.d/<service_name>` (SysV) or a systemd unit (`/etc/systemd/system/websphere.service`) that calls `startServer.sh`/`startManager.sh`/`startNode.sh` at boot.
- Ensure proper run levels (`chkconfig` on RHEL) or `systemctl enable` so it starts automatically after OS reboot.

## 7. WAS Kernel
- The **WebSphere Application Server kernel** refers to the core runtime (OSGi-based since WAS 8.x) that bootstraps the server process, class loading, and the underlying container services before application-level components load.

## 8. Difference: Refresh Pack vs Fix Pack
| Type | Purpose |
|---|---|
| **Fix Pack** | Cumulative maintenance release with bug fixes and minor enhancements within the same version (e.g., 8.5.5.x) |
| **Refresh Pack** | Larger update that may add new features/functions within a version stream, typically bigger than a fix pack but smaller than a full version upgrade |

## 9. When is manual synchronization required?
- After **DB changes** that aren't tracked by the Dmgr repository (e.g., manual updates outside the admin console).
- After **configuration changes** made directly on a node while auto-sync is disabled, or if node agent sync fails.
- Command: `syncNode.sh <dmgr_host> <dmgr_port>` run from the node to manually pull the latest master configuration.

## 10. GC Policy & JVM Tuning
- **GC Policies**: `gencon` (generational concurrent — default, good for most workloads), `optthruput`, `optavgpause`, `balanced` (region-based, good for large heaps).
- **Generational tuning**:
    - **Nursery (Young Gen)**: `-Xmn` — sized for short-lived objects; too small causes frequent minor GCs.
    - **Tenured/Old Gen**: holds long-lived objects; sized via overall `-Xmx` minus nursery.
    - **PermGen/Metaspace**: class metadata; in IBM JVMs this is typically merged into a shared class cache rather than a separate PermGen like older Oracle JVMs.
- Monitor via verbose GC logs (`-verbose:gc`) and tools like IBM GC and Memory Visualizer or `verbosegc` analysis.

## 11. Permission denied error on WebSphere after server reboot
- Usually an **OS-level issue**: WAS process/user losing ownership on log/config directories, or `/tmp` cleared on reboot removing socket/lock files.
- Check ownership/permissions on `profiles/<profile>/logs`, `temp`, `wstemp` directories — should be owned by the WAS runtime user.
- Verify ulimits (open files, processes) reset after reboot.
- Confirm mount points (NFS) are available before WAS starts (race condition on boot).

## 12. TLS version changed on browser — how to upgrade at WebSphere level
- Update the **SSL configuration** in the Admin Console: `Security > SSL certificate and key management > SSL configurations > <config> > Quality of protection (QoP)` and set the protocol (e.g., `TLSv1.2`, `TLSv1.3`).
- Update JVM custom property `com.ibm.jsse2.overrideDefaultTLS=true` and `com.ibm.websphere.security.util.disableSSLv3=true` type properties as needed for the specific IBM JDK.
- Restart affected servers/nodes for the SSL config to take effect.

## 13. How to check which TLS version a browser is using
- Browser dev tools → **Security tab** (Chrome: DevTools > Security > View certificate/connection info shows protocol, e.g., TLS 1.2/1.3).
- Alternatively use `openssl s_client -connect host:port -tls1_2` from a terminal to confirm what the server/browser negotiate.

## 14. Delete a certificate from a keystore
```bash
# IBM keystore (.kdb) using GSKit
gskcapicmd -cert -delete -db <keystore.kdb> -stashed -label "<cert_label>"

# Java keystore (JKS)
keytool -delete -alias <cert_alias> -keystore <keystore.jks>
```
(Note: `keytool` is for JKS/PKCS12; IBM `.kdb` files use `gskcapicmd`/`ikeyman`, not keytool directly.)

## 15. Migration: WebSphere 8.5.5 → 9.0 — Steps
1. **Pre-migration assessment** — check app/JDK compatibility, deprecated APIs.
2. **Backup** current profiles/configuration (`backupConfig.sh`).
3. **Install WAS 9.0** binaries alongside (or on new servers).
4. **Run Migration Toolkit** (`WASPreUpgrade` / `WASPostUpgrade` commands) to migrate profile configuration.
5. **Validate** migrated configuration — data sources, security, SSL, apps.
6. **Redeploy/test applications** in the new environment (regression testing).
7. **Cutover** — switch traffic (via load balancer/web server plugin) to new servers.
8. **Decommission** old WAS 8.5.5 environment once stable.

## 16. Backup Config during upgrade/migration
```bash
backupConfig.sh /path/to/backup.zip -nostop
```
- Always take a `backupConfig` snapshot before any upgrade/migration/patching activity so configuration can be restored (`restoreConfig.sh`) if the migration fails.

## 17. SSO Configuration on WebSphere
- **SiteMinder**: integrated via the SiteMinder WebAgent plugin on the web server tier, passing an SM header/token that WebSphere trusts through **Trust Association Interceptor (TAI)** configuration.
- **SAML**: WebSphere supports SAML Web SSO via **SAML TAI** or federated SSO configuration, validating SAML assertions from an Identity Provider (IdP) to establish an authenticated WebSphere session (LTPA token issuance after SAML validation).

## 18. Inode
> The inode is a unique identifier in the file system, used by the OS to locate and manage files/directories. When a file or directory is created, the file system assigns it a unique inode number.

Check inode of a file:
```bash
ls -i <filename>
# or full details
ls -l <filename>
```

## 19. How to check request counts to app servers / web server
- Use **IHS/Apache access logs** and analyze with tools like `awk`, `grep -c`, or log analyzers.
- Reference: IBM Community doc — *"Tips for using the IHS access log"* covers log format customization and counting requests per app/server.
```bash
# Example: count requests per hour from IHS access log
awk '{print $4}' access_log | cut -d: -f1-2 | sort | uniq -c
```