# Middleware Engineer Interview Notes — Coforge

> Senior-level preparation guide for a middleware engineer with 16 years of experience. Adapt product versions, tools, scale, and incident details to your actual experience.

## 1. Tell me about your day-to-day activities

I support middleware platform engineering, operations, automation, security, and production reliability across WebLogic, Apache HTTP Server, IBM HTTP Server, Nginx, Kubernetes/OpenShift, and related integration components.

My responsibilities include:

- Reviewing alerts, health dashboards, overnight incidents, capacity, and certificate-expiry reports.
- Troubleshooting JVM, thread, JDBC, deployment, SSL, proxy, networking, and application-availability issues.
- Performing deployments, configuration changes, patching, upgrades, and certificate renewals.
- Automating repetitive activities using Ansible, WLST, Python, shell, and CI/CD pipelines.
- Monitoring heap, garbage collection, stuck threads, response time, HTTP errors, and dependencies.
- Supporting disaster-recovery exercises, failovers, and infrastructure resilience.
- Conducting root-cause analysis and implementing permanent corrective actions.
- Coordinating with application, database, network, security, cloud, and vendor teams.
- Mentoring engineers and reviewing operational procedures and architecture changes.

As a senior engineer, I focus on reducing manual work, standardizing configurations, improving recovery time, and preventing incidents—not only resolving tickets.

## 2. Explain your WebLogic patching or upgrade process

I treat patching as a controlled lifecycle rather than simply running OPatch.

### Preparation

- Confirm WebLogic, Java, OS, OPatch, and patch compatibility.
- Review prerequisites, known issues, and superseded patches.
- Check the existing inventory:

```bash
$ORACLE_HOME/OPatch/opatch lsinventory
```

- Back up the domain configuration, security files, deployment plans, Node Manager configuration, and Oracle Home as required.
- Validate free disk space, permissions, rollback procedures, and application dependencies.
- Test in a lower environment and capture baseline health and performance data.

### Execution

- Drain traffic or place the instance in maintenance.
- Use rolling patching when supported.
- Stop the affected WebLogic components in the required order.
- Apply the patch using OPatch or the applicable SPBAT procedure.
- Patch Java separately when required.
- Restart services and monitor the logs.

### Validation

- Confirm the updated patch inventory.
- Check Admin Server, Node Manager, and managed-server logs.
- Validate deployments, JDBC pools, JMS, clusters, SSL, and front-end routing.
- Run smoke tests and compare performance with the baseline.
- Maintain a documented rollback decision point.

In production, I prefer node-by-node patching behind a load balancer. If shared domain artifacts change or the patch is non-rolling, I schedule a controlled outage.

## 3. Is WebLogic patching manual or automated?

It can be both, but I automate repeatable steps in enterprise environments:

- Pre-checks and inventory collection.
- Traffic draining.
- Server shutdown and startup.
- Patch staging and checksum validation.
- Patch installation.
- Log and health validation.
- Smoke testing.
- Restoring the server to the load balancer.
- Producing an execution report.

Production still has approval gates. The workflow stops when validation fails. Automation improves consistency but does not remove operational control.

## 4. How do you implement middleware automation?

I identify a repeatable and measurable operation, then define its prerequisites, execution, validation, failure handling, and rollback before automating it.

- **Ansible:** orchestration and configuration management.
- **WLST/Jython:** WebLogic configuration and lifecycle operations.
- **Python:** API integration, complex validation, and reporting.
- **Shell:** focused Linux administration tasks.
- **Terraform:** declarative infrastructure provisioning.
- **Jenkins/GitLab CI/Azure DevOps:** pipeline orchestration.
- **Git:** version control and peer review.

Automation standards include idempotency, secrets management, environment-specific variables, pre-checks, post-checks, structured logging, safe retries, and documented rollback.

## 5. Explain a WebLogic migration to OpenShift/Kubernetes

We first assessed application compatibility, Java versions, file-system dependencies, session handling, JMS, JDBC, security providers, and external integrations.

We containerized the approved WebLogic runtime and application. Domain configuration was managed through WebLogic Deploy Tooling or an appropriate persistent-domain model. The WebLogic Kubernetes Operator managed the domain custom resource, server pods, lifecycle operations, scaling, and rolling restarts.

Kubernetes Secrets stored sensitive values, ConfigMaps stored non-sensitive settings, Services handled internal access, and OpenShift Routes or ingress handled external access.

Persistent state was moved outside ephemeral containers. We deliberately designed JMS stores, transaction logs, shared storage, database connectivity, and session persistence. We added readiness/liveness probes, resource requests and limits, disruption controls, anti-affinity, and monitoring.

The migration progressed through lower environments, performance tests, parallel validation, and controlled traffic cutover. Redirecting traffic to the original platform remained the rollback option until acceptance criteria were met.

Running WebLogic in containers does not automatically make a legacy application cloud-native. Storage, transaction recovery, startup time, state, and licensing still need deliberate design.

## 6. What is the WebLogic Kubernetes Operator?

It is a Kubernetes controller that observes WebLogic Domain custom resources and manages the associated WebLogic pods and services. It supports domain lifecycle, server start/stop, cluster scaling, rolling restarts, image or configuration updates, and domain-status reporting.

The operator manages the WebLogic lifecycle; Kubernetes provides scheduling, service discovery, secrets, storage, and pod recovery.

## 7. Apache HTTP Server versus Nginx

Both can provide web serving, reverse proxying, TLS termination, and load balancing.

- **Apache:** mature module ecosystem, flexible enterprise integration, and common use with traditional middleware.
- **Nginx:** event-driven architecture, efficient concurrency handling, and frequent use as a reverse proxy or ingress tier.

I choose based on required modules, architecture, support standards, operational maturity, and performance requirements.

## 8. How do you configure different URLs?

Use name-based virtual hosts/server blocks and path-based routing where required.

### Apache example

```apache
<VirtualHost *:443>
    ServerName app1.example.com

    SSLEngine on
    SSLCertificateFile /etc/pki/tls/certs/app1.crt
    SSLCertificateKeyFile /etc/pki/tls/private/app1.key

    ProxyPreserveHost On
    ProxyPass        /api http://app1_backend/api
    ProxyPassReverse /api http://app1_backend/api
</VirtualHost>
```

### Nginx example

```nginx
server {
    listen 443 ssl;
    server_name app1.example.com;

    ssl_certificate     /etc/nginx/certs/app1.crt;
    ssl_certificate_key /etc/nginx/certs/app1.key;

    location /api/ {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
        proxy_pass http://app1_backend;
    }
}
```

Also validate DNS, certificate SANs, forwarded headers, health checks, timeouts, request-size limits, and application context roots.

## 9. How do you design DR across two regions?

Start with business RTO and RPO. These determine active-active, active-passive, or backup-and-restore architecture.

A typical design includes:

- Equivalent or compatible infrastructure in both regions.
- Configuration generated from the same version-controlled source.
- Replicated artifacts, secrets, and required state.
- Database replication aligned with RPO.
- Global load balancer or DNS traffic management.
- Application-level health checks.
- Tested failover and failback runbooks.
- Split-brain protection.

Cutover sequence:

1. Validate the DR environment and dependencies.
2. Quiesce writes if data consistency requires it.
3. Confirm replication lag and transaction state.
4. Promote or activate the DR database.
5. Start or scale the middleware platform.
6. Run technical and business smoke tests.
7. Redirect global load-balancer or DNS traffic.
8. Monitor errors, latency, sessions, and transactions.
9. Retain a controlled rollback option.

For DNS cutovers, reduce TTL in advance. A DR solution is incomplete until failover and failback have been tested.

## 10. Authentication versus authorization

- **Authentication:** establishes who the user is.
- **Authorization:** determines what the authenticated user may do.

Common integrations include LDAP/Active Directory, SAML 2.0, OpenID Connect/OAuth 2.0, WebLogic security realms, application-role mapping, and mutual TLS for system identities.

I prefer centralized identity, least privilege, separated administrative roles, and auditable access.

## 11. How do you implement MFA?

MFA is normally enforced by the enterprise identity provider, such as Entra ID, Okta, or Ping. The application or middleware redirects authentication to that provider.

1. The user accesses the application.
2. The application or proxy redirects to the identity provider.
3. The provider validates the primary credential and second factor.
4. It returns a signed SAML assertion or OIDC token.
5. The application validates the response and maps claims/groups to roles.

Apache may participate through a SAML or OIDC module, but the identity provider generally controls MFA policy.

## 12. How do you configure certificates in WebLogic?

- The **identity keystore** holds the server private key and certificate chain.
- The **trust keystore** holds trusted CA certificates.

Typical process:

1. Generate a key pair and CSR.
2. Submit the CSR to the approved CA.
3. Import root and intermediate certificates.
4. Import the signed server certificate using the original alias.
5. Configure custom identity and trust keystores in WebLogic.
6. Configure the private-key alias and securely managed password.
7. Apply the approved TLS protocol and cipher policy.
8. Restart where necessary and validate the entire chain.

```bash
keytool -list -v -keystore identity.jks
openssl s_client -connect app.example.com:443 -servername app.example.com -showcerts
```

Validate SAN/hostname matching, expiry, chain order, key association, permissions, and client trust.

## 13. How do you configure certificates in Apache?

```apache
<VirtualHost *:443>
    ServerName app.example.com

    SSLEngine on
    SSLCertificateFile /etc/httpd/certs/app-fullchain.pem
    SSLCertificateKeyFile /etc/httpd/private/app.key

    SSLProtocol -all +TLSv1.2 +TLSv1.3
</VirtualHost>
```

Validation commands:

```bash
apachectl configtest
openssl x509 -in /etc/httpd/certs/app-fullchain.pem -noout -subject -issuer -dates
openssl s_client -connect app.example.com:443 -servername app.example.com
```

The exact directives depend on the installed Apache version and certificate format. Protect private keys using strict filesystem permissions.

## 14. How do you manage certificates with 180-day validity?

Maintain a central inventory and alert at multiple thresholds, such as 60, 30, 15, and 7 days before expiry.

- Discover certificates across load balancers, web servers, middleware, Kubernetes Secrets, and trust stores.
- Track owner, CA, expiry, SANs, environment, and deployment location.
- Automate renewal through enterprise PKI or ACME where permitted.
- Deploy and validate the replacement certificate.
- Reload services safely.
- Confirm the externally served certificate.
- Retain a controlled rollback copy under the security policy.

Do not rely only on calendar reminders because applications may terminate TLS at several layers.

## 15. Which certificate authority do you use?

Use your real experience. A safe formulation is:

> I have worked with enterprise internal PKI for internal services and publicly trusted CAs for internet-facing endpoints. The selection depends on whether clients are corporate-managed, partner-facing, or public, and follows the organization's approved certificate policy.

Do not invent a CA name.

## 16. Which scripting languages and automation tools do you know?

- WLST/Jython for WebLogic configuration and lifecycle.
- Python for APIs, validation, and reporting.
- Shell for focused operating-system tasks.
- Ansible for orchestration and configuration management.
- Terraform for infrastructure provisioning.

Choose the tool based on the task rather than forcing everything into one language.

## 17. Describe a recent production issue

Use a genuine incident and structure the answer using Situation, Task, Action, and Result.

### Sample answer

A production application started returning intermittent HTTP 503 errors during peak traffic. Web-tier response time increased, although the WebLogic JVM processes remained available.

I correlated Apache access logs, WebLogic logs, thread dumps, JDBC metrics, and database activity. Thread dumps showed execute threads waiting for database connections, and the JDBC pool had reached maximum capacity. A recently introduced query was holding connections much longer than expected.

For immediate recovery, we controlled incoming traffic, coordinated database-session cleanup, and adjusted capacity only within tested limits. The service recovered without a complete platform restart.

The application and database teams optimized the query and corrected connection handling. We added JDBC utilization, wait-time, and stuck-thread alerts, as well as performance checks in the deployment pipeline.

Restarting WebLogic might have provided temporary relief, but it would not have fixed the root cause. Replace this example with your actual figures and experience.

## 18. Difference between `which`, `locate`, and `find`

### `which`

Finds an executable through the current `PATH`:

```bash
which java
command -v java
type java
```

### `locate`

Searches a prebuilt filename database. It is fast, but results can be stale:

```bash
locate httpd.conf
```

### `find`

Searches the live filesystem and supports filters and actions:

```bash
find /etc -type f -name 'httpd.conf'
```

## 19. Delete files older than 30 days

Preview matches first:

```bash
find /required/path -type f -mtime +30 -print
```

After confirming the resolved path and results:

```bash
find /required/path -type f -mtime +30 -delete
```

`-mtime +30` uses completed 24-hour periods and has boundary semantics. Confirm the precise retention requirement before deletion.

## 20. Find the process listening on port 8090

```bash
ss -ltnp 'sport = :8090'
```

Alternatives:

```bash
lsof -nP -iTCP:8090 -sTCP:LISTEN
fuser -v 8090/tcp
```

Inspect the PID before taking action:

```bash
ps -fp <PID>
readlink -f /proc/<PID>/exe
```

## 21. Print the third value from every row

Whitespace-delimited file:

```bash
awk '{print $3}' file.txt
```

Simple comma-separated file:

```bash
awk -F',' '{print $3}' file.csv
```

Use a real CSV parser if fields can contain quoted commas.

## 22. Configure the IBM HTTP Server WebLogic plug-in

The Oracle WebLogic proxy plug-in forwards matching requests from IBM HTTP Server or Apache to WebLogic. The exact module name and supported directives depend on the installed plug-in version.

Conceptual cluster example:

```apache
LoadModule weblogic_module modules/mod_wl_ohs.so

<Location /myapp>
    SetHandler weblogic-handler
    WebLogicCluster apphost1:7001,apphost2:7001
    DynamicServerList ON
</Location>
```

Single backend example:

```apache
<Location /myapp>
    SetHandler weblogic-handler
    WebLogicHost apphost1
    WebLogicPort 7001
</Location>
```

Validate the module and directives against the installed Oracle plug-in documentation. Configure suitable timeouts, keep-alive behavior, secure backend communication, cluster discovery, and logging.

**Important distinction:** `plugin-cfg.xml` is normally associated with the IBM WebSphere web-server plug-in. WebLogic generally uses Oracle WebLogic proxy plug-in directives. Confirm whether the backend is WebSphere or WebLogic.

## 23. Validate modified Apache configuration files

```bash
apachectl configtest
```

or:

```bash
httpd -t
```

Additional diagnostics:

```bash
httpd -S
httpd -M
```

- `httpd -S` displays the parsed virtual-host configuration.
- `httpd -M` displays loaded modules.

After successful validation, reload gracefully and verify:

```bash
systemctl reload httpd
systemctl status httpd
journalctl -u httpd --since '10 minutes ago'
curl -vk https://app.example.com/health
```

For IBM HTTP Server, the executable may be under the IBM installation path, but the sequence remains syntax test, graceful reload, log validation, and smoke test.

## 24. Which monitoring tools do you use?

I use Prometheus and Grafana for metrics and visualization, together with enterprise logging and alerting platforms. Native diagnostics include the WebLogic Administration Console, WLST, JMX metrics, JVM tools, access logs, thread dumps, heap dumps, and Java Flight Recorder.

The important monitoring categories are availability, latency, traffic, errors, saturation, and dependency health.

## 25. How do you configure Prometheus and Grafana?

Prometheus collects metrics from instrumented endpoints or exporters. Kubernetes/OpenShift commonly discovers targets through services and supported monitoring resources. External systems can use service discovery or controlled static targets.

Grafana uses Prometheus as a data source. Dashboards and alerts should use consistent labels for application, environment, cluster, server, and region. WebLogic metrics can be exposed using an approved monitoring exporter or JMX integration.

Keep scrape configuration, dashboards, and alert rules in version control so changes can be reviewed and deployed consistently.

## 26. What do you monitor using Prometheus and Grafana?

### Web and application tier

- Request rate and response time.
- HTTP 4xx and 5xx rates.
- Backend connection errors.
- Active connections and queue depth.
- TLS certificate expiry.
- Health-check status.

### WebLogic and JVM

- Managed-server state.
- Heap usage and utilization.
- Garbage-collection count and pause time.
- Execute-thread utilization.
- Hogging and stuck threads.
- JDBC active connections, capacity, failures, and wait time.
- JMS queue depth, consumers, and oldest-message age.
- Deployment state.
- Open sockets and request backlog.
- Transaction failures and timeouts.

### Kubernetes/OpenShift

- Pod readiness and restart count.
- Crash loops and scheduling failures.
- CPU and memory versus requests and limits.
- Node and namespace capacity.
- Replica availability.
- Persistent-volume usage.
- Operator and domain health.

### Service objectives

- Availability.
- p95 and p99 latency.
- Error rate.
- Traffic.
- Resource saturation.
- Replication health related to RTO and RPO.

Alerts should be actionable and related to service impact. Dashboards can contain deeper diagnostic information.

## Final interview guidance

- Explain why you chose an approach, not only the commands used.
- Use real incident and migration examples.
- Quantify scale and outcomes where possible.
- Mention validation, rollback, security, and automation.
- Never claim experience with a product you have not used.
- Show that a senior engineer prevents recurring incidents and improves platform reliability.
