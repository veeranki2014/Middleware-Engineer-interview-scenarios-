# WebLogic SME Interview Notes

> Role: **WebLogic SME, Irving, TX** | Candidate: **Anka Veeranki** (16 yrs middleware)
> JD focus: Tomcat, iAS, WebLogic, Unix, capacity planning, server maintenance, monitoring, weekend night shifts. CI/CD, OpenShift and cloud are a plus.

## Table of Contents

1. [Prep checklist](#1-prep-checklist)
2. [Opening and fit](#2-opening-and-fit)
3. [WebLogic core administration](#3-weblogic-core-administration)
4. [Clustering, HA and DR](#4-clustering-ha-and-dr)
5. [Performance and troubleshooting](#5-performance-and-troubleshooting)
6. [Patching and upgrades](#6-patching-and-upgrades)
7. [Security and SSL](#7-security-and-ssl)
8. [Tomcat and web servers](#8-tomcat-and-web-servers)
9. [Unix and scripting](#9-unix-and-scripting)
10. [Capacity planning and maintenance](#10-capacity-planning-and-maintenance)
11. [Monitoring](#11-monitoring)
12. [CI/CD, OpenShift, cloud](#12-cicd-openshift-cloud)
13. [Scenario and behavioral](#13-scenario-and-behavioral)
14. [Questions to ask them](#14-questions-to-ask-them)
15. [Quick command cheat sheet](#15-quick-command-cheat-sheet)

---

## 1. Prep checklist

- [ ] **Recent WebLogic depth.** Resume shows WebLogic 10.3.5 and 9.x/10.x in detail; Freddie Mac lists it lightly. Prepare 12c/14c specifics (OPatch, unicast clustering, Java 8/11) and honest, concrete examples.
- [ ] **"IAS".** Ask what they mean. Likely Oracle iAS or IHS/OHS. Tie the answer to your IHS/OHS/Apache experience.
- [ ] **Shift availability.** Have a clear, confident answer on weekend night shifts (see [section 13](#13-scenario-and-behavioral)).
- [ ] Have 2 to 3 STAR stories ready (incident, automation, upgrade).

> Answers are in first person. Adapt specifics to what you have actually done.

---

## 2. Opening and fit

<details>
<summary><b>Walk me through your background.</b></summary>

I have 16 years in middleware. I started as a Java developer, moved into WebSphere administration on AIX at O2 supporting 24x7 production, then into WebLogic, JBoss and Tomcat across banking and financial clients. Recently I've focused on automation and platform work: Ansible-driven installs and patching, Tomcat at scale (500+ servers), JVM tuning, and supporting a multi-tenant OpenShift/Kubernetes cluster. That matches your need for WebLogic and Tomcat administration, capacity planning and monitoring.
</details>

<details>
<summary><b>Why are you a fit for this role?</b></summary>

The role needs 4+ years of Tomcat, iAS and WebLogic on Unix. I have far more than that. I've done installs, patching, tuning, clustering, SSL and security, and incident handling. I've also worked in 24x7 support models, and the CI/CD, OpenShift and cloud experience you list as an advantage is already on my resume.
</details>

---

## 3. WebLogic core administration

<details>
<summary><b>Explain WebLogic domain architecture.</b></summary>

A domain is the administrative unit. It has one **Admin Server**, which hosts the console and holds the master `config.xml`. **Managed Servers** run the applications. **Machines** represent physical or virtual hosts, and **Node Manager** on each machine starts, stops and restarts servers remotely. **Clusters** group Managed Servers for load balancing and failover. The Admin Server is not in the request path, so its failure doesn't stop running Managed Servers.
</details>

<details>
<summary><b>What happens if the Admin Server goes down?</b></summary>

Managed Servers keep running and serving traffic because of **Managed Server Independence (MSI) mode**. During the outage you can't make config changes or use the console. To recover, restart the Admin Server, using `boot.properties` for unattended startup. In production I keep a backup of the domain directory, and I've done Admin Server migration to another machine by copying the domain and updating the listen address.
</details>

<details>
<summary><b>What is Node Manager and how do you configure it?</b></summary>

It's a per-machine daemon that lets the Admin Server, WLST or the console control Managed Servers remotely. It also enables crash recovery and auto-restart. I configure it via `nodemanager.properties`, choosing SSL or plain, Java-based or script-based (`StartScriptEnabled`), and register it as a service so it comes back after reboot. Then I enroll it with `nmEnroll` in WLST.
</details>

<details>
<summary><b>What does WLST do for you?</b></summary>

WLST is the scripting tool for anything repeatable. I use **offline mode** to build domains from templates and **online mode** (`connect()`) to deploy, tune, and check runtime MBeans like thread pool and JDBC stats. Scripts live in Git and run from Ansible or Jenkins.

```python
# WLST online example: check server state and thread pool
connect('weblogic', password, 't3://adminhost:7001')
domainRuntime()
cd('ServerRuntimes/ms1/ThreadPoolRuntime/ThreadPoolRuntime')
print(get('ExecuteThreadTotalCount'), get('HoggingThreadCount'), get('StuckThreadCount'))
```
</details>

<details>
<summary><b>Explain deployment staging modes.</b></summary>

| Mode | How it works | When to use |
|---|---|---|
| Stage | Admin Server copies the archive to each server | Default, best for clusters |
| No-stage | Servers read from a shared location | Only with reliable shared storage |
| External stage | I place the files myself | Custom or scripted deploys |
</details>

<details>
<summary><b>How do you do a zero-downtime deployment?</b></summary>

**Production redeployment**: WebLogic runs versions side by side. New sessions go to the new version, existing sessions finish on the old one, which is retired afterward. The alternative is a **rolling deployment** across the cluster, taking one node out of the load balancer at a time, which I've done across Tomcat clusters.
</details>

---

## 4. Clustering, HA and DR

<details>
<summary><b>How does clustering work and how is session failover handled?</b></summary>

Servers communicate via **unicast** (default in 12c) or multicast. With in-memory replication, each session has a **primary** and a **secondary** on another server, and the proxy plugin holds that information in a cookie. If the primary dies, requests route to the secondary. **Replication groups** control secondary placement across machines or racks. Alternatives are JDBC or file persistence, which are slower but survive a full cluster restart.
</details>

<details>
<summary><b>Whole Server Migration vs Service Migration?</b></summary>

- **WSM** moves an entire server, IP included, to another machine after failure.
- **Service Migration** moves only pinned services such as JMS and JTA.

Both need a leasing mechanism (database or consensus) and Node Manager.
</details>

<details>
<summary><b>How do you front WebLogic with a web tier?</b></summary>

Oracle HTTP Server (`mod_wl_ohs`), Apache with the WebLogic plug-in, or IHS. Key settings are `WebLogicCluster`, SSL termination, `WLProxySSL`, and timeouts (`WLIOTimeoutSecs`, `ConnectTimeoutSecs`). I keep `KeepAliveEnabled` on and validate health checks.

```apache
<Location /app>
  SetHandler weblogic-handler
  WebLogicCluster ms1:7003,ms2:7003
  WLIOTimeoutSecs 120
  ConnectTimeoutSecs 10
  KeepAliveEnabled ON
</Location>
```
</details>

---

## 5. Performance and troubleshooting

<details>
<summary><b>Managed Server shows STUCK threads or WARNING/OVERLOADED state. How do you troubleshoot?</b></summary>

1. Take **three thread dumps ~10 seconds apart** (`kill -3 <pid>` or `jstack`).
2. Look for the same threads at the same stack across dumps. Common culprits: slow DB query, downstream call without a timeout, lock contention.
3. Check JDBC pool stats, GC logs, and CPU/IO on the host.
4. Fix the root cause, not the symptom.

I also tune `StuckThreadMaxTime` (default 600s) and Work Managers with max-thread constraints to protect critical apps. For **OutOfMemoryError**, I take a heap dump (`jmap`), analyze in Eclipse MAT, and look for classloader leaks, which I've found in hot-redeployment scenarios.
</details>

<details>
<summary><b>How do you tune JDBC data sources?</b></summary>

- Initial and Max Capacity matched to DB session limits and app concurrency
- **Test Connections on Reserve** with a lightweight query
- Connection Reserve Timeout, Inactive Connection Timeout (catches leaks), Statement Cache size
- Oracle RAC: Active GridLink or multi data sources
- Sizing check: total connections across all servers must stay below the DB limit
</details>

<details>
<summary><b>How do you tune the JVM and GC?</b></summary>

I start from GC logs, not guesses. `Xms` = `Xmx` to avoid resize pauses, heap sized to live set plus headroom, collector chosen for the workload. On Java 8/11 I've used **G1GC**, tuning pause targets and IHOP to reduce Full GC frequency. I watch old-gen occupancy after full GC, pause percentiles, and allocation rate, then re-test under load.
</details>

<details>
<summary><b>Production app is slow at 2 AM. What's your approach?</b></summary>

1. **Scope:** one server, one app, or everything?
2. **Walk the stack:** LB and web tier, WebLogic (thread pool, queue, stuck threads, JDBC waits), JVM (GC, heap), OS (CPU, memory, disk, network), DB and downstream.
3. **What changed?** Deploy, config, traffic.
4. **Capture evidence** (dumps, logs) before restarting.
5. Restore service, then RCA and a runbook entry.
</details>

---

## 6. Patching and upgrades

<details>
<summary><b>How do you patch WebLogic?</b></summary>

OPatch for PSUs and CPUs on the quarterly Oracle schedule.

```bash
# 1. Backup ORACLE_HOME and domain first
tar czf /backup/wl_home_$(date +%F).tgz $ORACLE_HOME
# 2. Check current state
$ORACLE_HOME/OPatch/opatch lsinventory
# 3. Stop servers and Node Manager, then check conflicts
$ORACLE_HOME/OPatch/opatch prereq CheckConflictAmongPatchesWithinSameOracleHome -phBaseDir /patches/<id>
# 4. Apply
$ORACLE_HOME/OPatch/opatch apply      # or napply for multiple patches
# 5. Restart and validate
```

For clusters I patch in a rolling fashion for zero downtime and automate with **Ansible Tower** across hosts. Earlier 10.x patch-sets used the Smart Update tool.
</details>

<details>
<summary><b>How would you approach a major upgrade, e.g., 12c to 14c?</b></summary>

Assess compatibility (JDK version, deprecated features, app libraries), build a parallel environment, use domain upgrade tools or recreate the domain from scripts, run regression and performance tests, then cut over with a rollback plan. Never upgrade in place without a tested rollback.
</details>

---

## 7. Security and SSL

<details>
<summary><b>How do you configure SSL in WebLogic?</b></summary>

Create an **identity keystore** (private key + cert) and a **trust keystore** (CA certs). Set *Custom Identity and Custom Trust* in the server's Keystores tab, specify alias and passphrases, enable the SSL port. Use `keytool` for the CSR and import the chain in order (root, intermediate, server cert), then verify with `openssl s_client`. I've handled domain-validated and wildcard certs and track certificate lifecycle so renewals don't cause outages.

```bash
keytool -genkeypair -alias server -keyalg RSA -keysize 2048 -keystore identity.jks
keytool -certreq -alias server -file server.csr -keystore identity.jks
openssl s_client -connect host:7002 -showcerts
```
</details>

<details>
<summary><b>What security realms and providers have you configured?</b></summary>

LDAP authentication providers (Active Directory, Sun One), SAML SSO, LTPA and NTLM in WebSphere contexts, and role/policy mapping. Hardening: remove default users and sample apps, enable hostname verification, restrict the admin console.
</details>

---

## 8. Tomcat and web servers

<details>
<summary><b>How do you tune Tomcat?</b></summary>

Connector settings (`maxThreads`, `acceptCount`, `connectionTimeout`, `maxConnections`), shared Executors across connectors, and JVM settings. Threads are sized from expected concurrency and latency, not just raised blindly. I use Access Log Valve output with awk/grep or ELK to find slow URIs and p99 degradation.
</details>

<details>
<summary><b>How did you put Tomcat behind NGINX?</b></summary>

NGINX terminates SSL and forwards `X-Forwarded-*` headers. In Tomcat I enable **RemoteIpValve** so the real client IP and scheme appear in logs and `request.isSecure()`.

```xml
<Valve className="org.apache.catalina.valves.RemoteIpValve"
       remoteIpHeader="X-Forwarded-For"
       protocolHeader="X-Forwarded-Proto" />
```
</details>

<details>
<summary><b>How do you harden Tomcat?</b></summary>

Remove default webapps, disable TRACE, hide the server version header, run as non-root, restrict the shutdown port, keep patches current.
</details>

<details>
<summary><b>Tell me about iAS / IHS / OHS.</b></summary>

Clarify with the interviewer first. Then: I've installed and configured IHS and the WebSphere plug-in, Oracle HTTP Server, and Apache, including virtual hosts, SSL, plug-in config, and log rotation.
</details>

---

## 9. Unix and scripting

<details>
<summary><b>Which Unix commands do you use for troubleshooting?</b></summary>

`top/htop`, `vmstat`, `iostat`, `sar`, `free -m`, `df -h`, `lsof`, `netstat`/`ss`, `ps -ef`, `tail -f`, `grep/awk/sed`, `strace`, and `kill -3` for thread dumps. For disk: `du -sh * | sort -h`, and `lsof +L1` for deleted-but-open files.
</details>

<details>
<summary><b>Give an example of automation you've done.</b></summary>

Shell scripts for log rotation (logrotate with copytruncate), multi-instance health checks, and disk usage alerts, which cut manual on-call intervention. Also Ansible playbooks for installs and Ansible Tower jobs for restarts and patch distribution.
</details>

---

## 10. Capacity planning and maintenance

<details>
<summary><b>How do you do capacity planning?</b></summary>

Gather baseline metrics (CPU, heap, thread usage, JDBC utilization, response time) over normal and peak periods, add expected growth or new-app load, and use load testing to get per-server throughput at acceptable latency. From that I calculate node count with **N+1 headroom**. Reviewed quarterly and before major events.
</details>

<details>
<summary><b>What does routine server maintenance look like?</b></summary>

Patch cycles, log and disk cleanup, certificate expiry checks, config backups, thread and heap trend reviews, capacity reports, DR tests.
</details>

---

## 11. Monitoring

<details>
<summary><b>Which tools have you used and what do you monitor?</b></summary>

Splunk, ELK, AppDynamics, Nagios, AWS CloudWatch.

| Layer | Metrics |
|---|---|
| WebLogic | Server state/health, thread pool, stuck threads, JMS queue depth |
| JVM | Heap, GC pauses, old-gen after full GC |
| JDBC | Active connections, wait counts, leaked connections |
| HTTP | Response times, error rates |
| OS | CPU, memory, disk, network |

I alert on trends and thresholds, not just failures.
</details>

---

## 12. CI/CD, OpenShift, cloud

<details>
<summary><b>How does CI/CD apply to middleware?</b></summary>

Jenkins and Azure DevOps pipelines build with Maven, store artifacts in Nexus or Artifactory, and deploy via WLST or Ansible. Config lives in Git, and I promote through dev, test and prod with approvals.
</details>

<details>
<summary><b>What's your OpenShift/Kubernetes experience?</b></summary>

I support a multi-tenant OpenShift/Kubernetes cluster and hold the **CKA**. I handle namespaces and quotas, deployments, troubleshooting CrashLoopBackOff and connection issues, and wrote runbooks that reduced MTTR. For WebLogic, I understand the **WebLogic Kubernetes Operator** approach: domains run as pods and the Operator handles lifecycle, scaling and rolling restarts.
</details>

---

## 13. Scenario and behavioral

<details>
<summary><b>Describe a tough production incident (STAR).</b></summary>

- **Situation:** intermittent outages in production
- **Task:** restore service and find the cause
- **Action:** thread dumps showed thread pool exhaustion from a slow downstream call with no timeout; restored service, worked with devs to add timeouts, tuned a Work Manager to isolate the app
- **Result:** no recurrence, documented runbook, lower MTTR
</details>

<details>
<summary><b>How do you handle change management?</b></summary>

Approved ticket with plan, lower-environment test, rollback plan and maintenance window. Verify after the change and update the record.
</details>

<details>
<summary><b>The JD requires weekend night shifts. Any concern?</b></summary>

No. I've supported 24x7 production before (O2 project) and I'm comfortable with on-call and shift work. Confirm the rotation and hours, and mention that automation reduces manual overnight intervention.
</details>

---

## 14. Questions to ask them

1. Which WebLogic version and JDK are in use, and is a migration planned?
2. How many domains and servers are in scope, and on which OS?
3. What does "iAS" refer to in your stack?
4. What are the shift rotation and on-call expectations?
5. What monitoring tools are in place, and is OpenShift a near-term target for these apps?

---

## 15. Quick command cheat sheet

```bash
# Thread dump / heap
kill -3 <pid>
jstack -l <pid> > td_$(date +%H%M%S).txt
jmap -dump:live,format=b,file=heap.hprof <pid>
jstat -gcutil <pid> 5000

# WebLogic
./startWebLogic.sh
./startNodeManager.sh
java weblogic.WLST script.py
java weblogic.Admin -url t3://host:7001 -username u -password p GETSTATE

# Patching
$ORACLE_HOME/OPatch/opatch lsinventory
$ORACLE_HOME/OPatch/opatch apply

# OS
vmstat 5 5; iostat -x 5 3; ss -tanp | grep :7001
lsof +L1
df -h; du -sh * | sort -h
```