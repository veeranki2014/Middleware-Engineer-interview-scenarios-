# Middleware Engineer Interview Prep — WebLogic & WebSphere (16+ yrs level)

> Answers written as a senior/lead middleware admin would frame them — process-oriented, with commands and reasoning, not just one-liners.

---

## 1. Day-to-Day Responsibilities

- Managing WebLogic and WebSphere domains/cells across Dev, QA, UAT, and Prod (health checks, JVM/heap monitoring, thread dumps, log review).
- Application/EAR-WAR deployments and rollbacks, coordinating with release management (CAB approvals, change tickets).
- Patch/upgrade planning (OPatch for WebLogic, IBM Installation Manager/Fix Central for WebSphere).
- Certificate lifecycle management (renewals, keystore/truststore updates) on WebLogic, WebSphere, Apache/IBM HTTP Server.
- Performance tuning: JVM args, connection pools, thread pools, GC tuning.
- Automation: writing WLST/wsadmin scripts, Ansible playbooks, and Terraform for provisioning.
- Incident/problem management: troubleshooting prod outages, RCA documentation.
- DR drills, capacity planning, and monitoring dashboard upkeep (Prometheus/Grafana, Splunk, AppDynamics).
- Security compliance: patching CVEs, SSL/TLS hardening, access reviews.

---

## 2. Patching / Upgrade Process

### WebLogic
1. **Pre-checks**: Review release notes/CVE bulletin, check OPatch/Fusion Middleware compatibility matrix.
2. **Backup**: Snapshot domain (`$DOMAIN_HOME`), export config, backup `oraInventory` and Middleware Home.
3. **Conflict check**: `opatch prereq CheckConflictAgainstOH -phBaseDir <patch_dir>`
4. **Apply patch (offline)**:
   ```bash
   cd $ORACLE_HOME/OPatch
   ./opatch apply <patch_location> -silent
   ```
5. **Post-patch**: Run `opatch lsinventory` to confirm, run smart update / WLS Patch Set Updates (PSU) validation scripts.
6. **Restart** AdminServer, then Managed Servers in rolling fashion (if clustered) to keep app availability.
7. **Validation**: Application smoke tests, check `AdminServer.log` / `.out` for errors, verify JDBC/JMS resources.
8. **Rollback plan**: `opatch rollback -id <patch_id>` if validation fails.

### WebSphere
1. **Pre-checks**: Confirm fix pack/interim fix compatibility via IBM Fix Central.
2. **Backup**: `backupConfig.sh` for the cell, plus a full profile backup.
3. **Apply using Installation Manager (IM)**:
   ```bash
   ./imcl install com.ibm.websphere.ND.v90 -repositories <repo_path> -installationDirectory <was_home> -acceptLicense
   ```
   or `updateInstaller`/`fixmanager` for interim fixes.
4. **Federate/sync nodes**: `syncNode.sh` on each node agent after Dmgr is updated.
5. **Restart** Dmgr → Node Agents → Application Servers in sequence.
6. **Validation**: `versionInfo.sh`, check SystemOut.log, run app health checks.
7. **Rollback**: `imcl rollback` or restore from `backupConfig`.

**Manual vs Automation**: Manual for the first pass in a new environment / when patch has unknowns; automated (scripted) for repeatable, well-tested patch cycles across many nodes to reduce human error and downtime window.

---

## 3. Automation / Scripting Implementation

- **WLST** (Jython) for WebLogic domain config, deployments, patch validation — run in offline or online mode via `java weblogic.WLST script.py`.
- **wsadmin** (Jython/Jacl) for WebSphere — equivalent automation for cell/node/server config, app install.
- **Ansible** playbooks to orchestrate: pre-checks → stop servers → backup → apply patch → restart → validate, across an inventory of hosts. Use roles per server type (weblogic, was, apache).
- **Shell scripts** wrap WLST/wsadmin calls, handle logging, error trapping (`set -e`, trap on exit), and Slack/email notification on completion.
- **Terraform** for infra provisioning (VMs, OpenShift namespaces, LB rules, DNS entries) — keeps environment reproducible; state file managed in remote backend (S3/Artifactory).
- CI/CD integration: Jenkins/GitLab pipeline triggers Ansible playbook against target inventory, artifacts pulled from Nexus/Artifactory, approvals gated for Prod.

---

## 4. WebLogic Kubernetes Operator Migration (OpenShift)

1. **Assessment**: Inventory existing WebLogic domains, JDBC/JMS resources, and app dependencies.
2. **Containerize**: Build WebLogic Docker image using Oracle's `WebLogic Image Tool` (WIT) — layer Oracle Home + PSU patches + domain config.
3. **Domain model**: Use **Model in Image (MII)** or **Domain in PV** strategy — MII preferred for OpenShift since config is baked into image/ConfigMap, simplifying immutable deployments.
4. **Install WebLogic Kubernetes Operator (WKO)** via Helm into the OpenShift cluster, scoped to target namespace(s).
5. **Define Domain CRD** (`domain.yaml`) — specifies image, cluster topology, replicas, resources, and startup policy.
6. **Networking**: Configure OpenShift Routes/Services for AdminServer console + managed server cluster; T3/T3S channel setup for WLST access if needed.
7. **Secrets**: Store DB credentials, keystores as OpenShift Secrets, mounted into pods.
8. **Cutover**: Run in parallel with legacy VM-based domain, validate via smoke tests and traffic mirroring, then switch DNS/LB to new OpenShift routes.
9. **Post-migration**: Set up HPA (horizontal pod autoscaling), liveness/readiness probes, and log forwarding to centralized logging (EFK/Splunk).

*(WebSphere equivalent: **WebSphere Liberty Operator** for OpenShift, using `liberty.yaml` CRDs and Liberty's `server.xml` packaged into container images — similar migration philosophy from traditional WAS ND cells to Liberty on OpenShift.)*

---

## 5. Webservers: Apache & Nginx

### Configuring Different URLs / Virtual Hosts
**Apache** (`httpd.conf` / `vhosts.conf`):
```apache
<VirtualHost *:443>
    ServerName app1.example.com
    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/app1.crt
    SSLCertificateKeyFile /etc/pki/tls/private/app1.key
    ProxyPass /app1 http://weblogic_cluster/app1
    ProxyPassReverse /app1 http://weblogic_cluster/app1
</VirtualHost>
```
Use `mod_proxy`/`mod_weblogic` (WebLogic Proxy Plugin) or `mod_jk`/IBM HTTP plugin (`plugin-cfg.xml`) for WebSphere.

**Nginx**:
```nginx
server {
    listen 443 ssl;
    server_name app2.example.com;
    ssl_certificate /etc/nginx/certs/app2.crt;
    ssl_certificate_key /etc/nginx/certs/app2.key;

    location /app2 {
        proxy_pass http://weblogic_upstream;
    }
}
```
Different URLs are handled via distinct `server_name`/`VirtualHost` blocks or `location`/`Location` path-based routing to different backend clusters.

### DR — Same Infra on East1 & East2
- Identical topology deployed in both data centers (active-passive or active-active depending on RTO/RPO requirements).
- Database replication (Data Guard/GoldenGate for Oracle DB) keeps East2 in sync.
- Config parity maintained via version-controlled Ansible/Terraform templates — same playbook deploys to either DC.
- Cutover: Update Global Server Load Balancer (GSLB)/DNS (e.g., F5 GTM, Route53 failover routing) to point to East2 VIPs; drain connections gracefully from East1; validate app health in East2 before full traffic switch; keep East1 as fallback until stability confirmed.
- Runbooks define step-by-step manual/automated failover with checklists and rollback triggers.

---

## 6. Authentication & Authorization

- **Integration**: WebLogic/WebSphere configured with an **LDAP Authentication Provider** (Active Directory/OpenLDAP) as the identity store; app-level roles mapped to LDAP groups via security realms (WebLogic) or Global Security/JAAS (WebSphere).
- **App-level authorization**: Applications use **JAAS** roles mapped in `web.xml`/`weblogic.xml` (WebLogic) or `ibm-application-bnd.xml` (WebSphere) to LDAP groups; fine-grained authorization can also be delegated to an external **OAuth2/OIDC** provider (Okta, Azure AD, PingFederate) using SAML/OIDC integration for SSO.
- **MFA Setup**: MFA typically isn't handled at the WebLogic/WebSphere layer itself — it's enforced at the **IdP** (Okta/Azure AD/PingFederate) via SAML/OIDC redirect flow, or at the **reverse proxy/WAF** layer (e.g., Apache with `mod_auth_openidc`) in front of the app servers. The app server trusts the assertion/token issued after MFA challenge.
- **Where configured**: Usually at Apache/reverse-proxy level (auth module intercepts request, redirects to IdP, validates token/cookie) rather than directly at WebLogic — keeps app servers stateless regarding auth mechanics and centralizes policy control.

---

## 7. Certificates

- **WebLogic**: Certificates stored in **Java KeyStore (JKS)** or **Oracle Wallet**; configured under Server → Keystores → Custom Identity/Trust, and SSL tab points to the private key alias. CLI: `keytool -importcert -keystore identity.jks ...`
- **Apache/IBM HTTP Server**: Certs in PEM format referenced by `SSLCertificateFile`/`SSLCertificateKeyFile`, or in a **kdb** file (GSKit) for IBM HTTP Server, managed via `gskcapicmd`/`ikeyman`.
- **Validity**: Typically issued for **180 days to 1 year**, depending on internal CA policy (many orgs now enforce 90–180 day rotation per CA/Browser Forum trends). Tracked via an internal cert-inventory tool with expiry alerts (e.g., 30/15/7-day warnings).
- **CA used**: Enterprise-internal CA (e.g., Microsoft ADCS, Venafi-issued, or DigiCert/Entrust for public-facing endpoints) — varies by environment; internal apps often use an internal PKI, external-facing apps use a public trusted CA.
- **Do you have certs on Apache?** Yes — SSL termination commonly happens at Apache/IBM HTTP Server layer; configure via `SSLCertificateFile/KeyFile` (Apache) or the **kdb keystore** referenced in `plugin-cfg.xml`/`httpd.conf` for IHS, then restart the webserver to pick up renewed certs. Automation via Ansible role that redeploys cert + key + restarts service is standard for renewal at scale.

---

## 8. Scripting Languages

| Language | Primary Use |
|---|---|
| WLST (Jython) | WebLogic domain automation, deployment, config |
| wsadmin (Jython) | WebSphere cell/node/app automation |
| Python | General scripting, REST API automation (WLS REST/WAS admin REST), log parsing |
| Ansible | Configuration management, patch orchestration, multi-node automation |
| Shell (bash/ksh) | OS-level automation, wrapper scripts, cron jobs |
| Terraform | Infra provisioning (compute, network, OpenShift resources) |

---

## 9. Recent Prod Issue & Troubleshooting Example

**Scenario**: Managed server unresponsive / high CPU, users getting 503s.
1. Checked `top`/`ps` for CPU-heavy JVM PID.
2. Took 3–5 thread dumps 10 seconds apart: `kill -3 <pid>` (output goes to `.out` log) or `jstack <pid> > threaddump.txt`.
3. Analyzed with **Thread Dump Analyzer (TDA)**/Samurai — identified stuck threads waiting on a JDBC connection pool (pool exhaustion).
4. Checked WebLogic Console → JDBC DataSource → Monitoring tab: active connections maxed out, "waiting for connection" count high.
5. Root cause: a slow downstream DB query holding connections + pool `Maximum Capacity` set too low for load.
6. **Fix**: Increased pool size temporarily, killed the long-running SQL session on DB side, worked with DBA to add missing index; long-term fix was a code review of that query and adding a statement timeout.
7. Documented RCA and added a Grafana alert on "connection pool wait count" to catch this earlier next time.

---

## 10. Linux / Shell Command Deep-Dive

**`which` vs `locate` vs `find`**
- `which <cmd>` — shows the full path of an executable as resolved from `$PATH` (used to confirm which binary will actually run).
- `locate <name>` — fast search using a prebuilt file-name index (`updatedb`); good for quick lookups, but index may be stale.
- `find <path> -name <name>` — real-time filesystem search, slower but always accurate; supports rich filters (size, time, permission, type).

**Delete files older than 30 days:**
```bash
find /path/to/dir -type f -mtime +30 -exec rm -f {} \;
```

**Find process running on port 8090:**
```bash
lsof -i :8090
# or
netstat -tulnp | grep 8090
# or (newer systems)
ss -tulnp | grep 8090
```

**Print 3 values from each row of a 4-column file:**
```bash
awk '{print $1, $2, $3}' filename
```

**IBM HTTP Server plugin (`plugin-cfg.xml`)**:
- Generated from the WebSphere Deployment Manager (Servers → Web Servers → plugin-cfg.xml → Generate), then **propagated** to the IHS server (`propagatePluginCfg` or manual copy) to route requests to the correct WebSphere application clusters. It defines routing rules, transports (HTTP/HTTPS), and load-balancing weights between IHS and the app servers.

**Validate modified Apache conf files:**
```bash
apachectl configtest
# or
httpd -t
```
For IBM HTTP Server: `apachectl -t` (IHS is Apache-based) before restarting, to avoid taking down the webserver on a syntax error.

---

## 11. Monitoring: Prometheus & Grafana

- **WebLogic**: Use the **WebLogic Monitoring Exporter** (deployed as a web app on the managed server) which exposes JVM/JMX metrics in Prometheus format at `/wls-exporter/metrics`. Prometheus scrapes this endpoint via a `scrape_config` job.
- **WebSphere**: Use **PMI (Performance Monitoring Infrastructure)** combined with a JMX exporter, or IBM's own `websphere_exporter`, to expose metrics to Prometheus.
- **Grafana**: Connect Prometheus as a data source; import/build dashboards for JVM heap usage, GC pause time, thread pool utilization, JDBC connection pool stats, request throughput/latency, and HTTP 5xx error rates.
- **What to monitor**:
    - Heap usage / GC frequency & pause times
    - Thread pool: active vs idle vs stuck threads
    - JDBC connection pool: active connections, wait count, wait time
    - Request throughput and response time percentiles (p95/p99)
    - HTTP error rates (4xx/5xx) at webserver and app server layer
    - CPU/Memory/Disk at OS level (via Node Exporter)
    - Certificate expiry (custom exporter/blackbox probe)
    - Cluster/member health and failover events
    - Alerting rules (Alertmanager) for thresholds — e.g., heap > 85%, pool wait count > threshold, server down.

---

*Tip: When answering in an interview, always frame responses as "process → tool → validation → rollback" — this signals operational maturity, not just tool familiarity.*