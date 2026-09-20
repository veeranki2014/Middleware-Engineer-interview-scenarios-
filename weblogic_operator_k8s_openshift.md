# WebLogic on Kubernetes & OpenShift with the WebLogic Kubernetes Operator

Personal reference notes: production-grade **Model-in-Image (MII)** deployment of an `orders.war` app on a 3-node WebLogic dynamic cluster, on vanilla Kubernetes and on OpenShift Container Platform (OCP).

> Versions move fast. Before using, verify operator, WDT, WebLogic image tags and OCP compatibility in the official docs: https://oracle.github.io/weblogic-kubernetes-operator/

## Contents

1. [Scenario and architecture](#1-scenario-and-architecture)
2. [Prerequisites](#2-prerequisites)
3. [Repo layout](#3-repo-layout)
4. [Part A: Vanilla Kubernetes](#4-part-a-vanilla-kubernetes)
5. [Part B: OpenShift (OCP) deltas](#5-part-b-openshift-ocp-deltas)
6. [Day-2 operations cheat sheet](#6-day-2-operations-cheat-sheet)
7. [Troubleshooting](#7-troubleshooting)
8. [Production hardening checklist](#8-production-hardening-checklist)
9. [Alternatives and next steps](#9-alternatives-and-next-steps)

---

## 1. Scenario and architecture

- **HA:** pods spread across nodes/zones, PodDisruptionBudget via `maxUnavailable`
- **Immutable config:** domain defined by WDT model files (no hand-patched domain home)
- **Zero-downtime changes:** rolling restarts for image, patch, or config changes
- **Scaling:** manual or HPA on CPU
- **Ops:** ingress/route with sticky sessions, Prometheus metrics, centralized logs, secrets kept out of images

```
Client -> Ingress/Route (TLS, cookie affinity)
           -> Service orders-domain-cluster-cluster-1 (:8001)
                 |- managed-server1
                 |- managed-server2
                 '- managed-server3
          Admin Server pod (internal only)
          Operator (ns: weblogic-operator-ns) watches Domain/Cluster CRs
          Oracle DB (external) <- JDBC datasource from model
```

**Why MII:** the base image holds patched WebLogic binaries; an *auxiliary image* holds the WDT model + app; the operator's introspector job builds the domain at startup. Model change means pods roll. No shared PV needed.

## 2. Prerequisites

- Kubernetes 1.26+ (or OCP 4.12+), 3+ worker nodes
- `kubectl` / `oc`, `helm` 3.x, Docker or Podman
- Oracle Container Registry account (accept WebLogic license at container-registry.oracle.com)
- Ingress controller (K8s) or router (OCP), metrics-server
- Private registry (or OCP internal registry) for the aux image
- Valid Oracle WebLogic license entitlement for production

## 3. Repo layout

```
weblogic-k8s-prod/
├── app/orders/                      # app source -> orders.war
├── aux-image/
│   ├── Dockerfile
│   └── models/
│       ├── domain.yaml              # WDT model
│       ├── domain.properties
│       └── archive.zip              # wlsdeploy/applications/orders.war
├── k8s/
│   ├── 00-namespace.yaml
│   ├── 01-secrets.sh
│   ├── 02-datasource-configmap.yaml
│   ├── 03-domain.yaml
│   ├── 04-cluster.yaml
│   ├── 05-ingress.yaml              # or 05-route.yaml on OpenShift
│   ├── 06-hpa.yaml
│   └── 07-networkpolicy.yaml        # OpenShift / default-deny clusters
└── operator/values.yaml
```

---

## 4. Part A: Vanilla Kubernetes

### Step 1: Install the operator

```bash
kubectl create namespace weblogic-operator-ns
kubectl create serviceaccount -n weblogic-operator-ns weblogic-operator-sa
helm repo add weblogic-operator https://oracle.github.io/weblogic-kubernetes-operator/charts --force-update
```

`operator/values.yaml`

```yaml
serviceAccount: weblogic-operator-sa
domainNamespaceSelectionStrategy: LabelSelector
domainNamespaceLabelSelector: weblogic-operator=enabled
replicas: 2                      # HA operator (leader election)
javaLoggingLevel: INFO
enableClusterRoleBinding: true
```

```bash
helm install weblogic-operator weblogic-operator/weblogic-operator \
  -n weblogic-operator-ns -f operator/values.yaml --wait
kubectl get pods -n weblogic-operator-ns
```

### Step 2: Namespace and secrets

`k8s/00-namespace.yaml`

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: orders-ns
  labels:
    weblogic-operator: enabled
```

`k8s/01-secrets.sh` (in production, source from Vault / External Secrets / cloud secret manager)

```bash
NS=orders-ns
kubectl apply -f k8s/00-namespace.yaml

kubectl -n $NS create secret generic orders-weblogic-credentials \
  --from-literal=username=weblogic --from-literal=password='<StrongPassword#1>'
kubectl -n $NS create secret generic orders-runtime-encryption \
  --from-literal=password='<AnotherStrongPassword#2>'
kubectl -n $NS create secret generic orders-db-secret \
  --from-literal=username=ORDERS_APP --from-literal=password='<DBPassword>'

for s in orders-weblogic-credentials orders-runtime-encryption orders-db-secret; do
  kubectl -n $NS label secret $s weblogic.domainUID=orders-domain
done

kubectl -n $NS create secret docker-registry ocr-secret \
  --docker-server=container-registry.oracle.com \
  --docker-username='<oracle-sso-user>' --docker-password='<token>'
kubectl -n $NS create secret docker-registry myreg-secret \
  --docker-server=<your-registry> --docker-username=<u> --docker-password=<p>
```

### Step 3: WDT model

`aux-image/models/domain.yaml`

```yaml
domainInfo:
  AdminUserName: '@@SECRET:__weblogic-credentials__:username@@'
  AdminPassword: '@@SECRET:__weblogic-credentials__:password@@'
  ServerStartMode: prod

topology:
  Name: orders-domain
  AdminServerName: admin-server
  ProductionModeEnabled: true
  Cluster:
    cluster-1:
      DynamicServers:
        ServerTemplate: cluster-1-template
        ServerNamePrefix: managed-server
        DynamicClusterSize: 6         # max capacity
        MaxDynamicClusterSize: 6
        CalculatedListenPorts: false
  Server:
    admin-server:
      ListenPort: 7001
  ServerTemplate:
    cluster-1-template:
      Cluster: cluster-1
      ListenPort: 8001
      GracefulShutdownTimeout: 120

resources:
  JDBCSystemResource:
    OrdersDS:
      Target: cluster-1
      JdbcResource:
        JDBCDataSourceParams:
          JNDIName: jdbc/ordersDS
          GlobalTransactionsProtocol: None
        JDBCDriverParams:
          DriverName: oracle.jdbc.OracleDriver
          URL: '@@PROP:DB_URL@@'
          PasswordEncrypted: '@@SECRET:orders-db-secret:password@@'
          Properties:
            user:
              Value: '@@SECRET:orders-db-secret:username@@'
        JDBCConnectionPoolParams:
          InitialCapacity: 5
          MinCapacity: 5
          MaxCapacity: 30
          TestConnectionsOnReserve: true
          TestTableName: SQL SELECT 1 FROM DUAL

appDeployments:
  Application:
    orders:
      SourcePath: wlsdeploy/applications/orders.war
      ModuleType: war
      Target: cluster-1
```

`domain.properties`

```
DB_URL=jdbc:oracle:thin:@//db-host:1521/ORDERSPDB
```

Archive (correct structure matters):

```bash
mkdir -p wlsdeploy/applications && cp app/orders/target/orders.war wlsdeploy/applications/
cd aux-image/models && zip -r archive.zip ../../wlsdeploy
```

### Step 4: Build the auxiliary image

`aux-image/Dockerfile`

```dockerfile
FROM busybox AS build
ARG WDT_VERSION=4.0.0
ADD https://github.com/oracle/weblogic-deploy-tooling/releases/download/release-${WDT_VERSION}/weblogic-deploy.zip /tmp/
RUN mkdir -p /auxiliary/models /auxiliary/weblogic-deploy \
 && unzip -q /tmp/weblogic-deploy.zip -d /auxiliary
COPY models/ /auxiliary/models/

FROM busybox
COPY --from=build /auxiliary /auxiliary
RUN adduser -D -u 1000 oracle && chown -R 1000:1000 /auxiliary
USER 1000
```

```bash
docker build -t <your-registry>/orders-aux:1.0.0 aux-image/
docker push <your-registry>/orders-aux:1.0.0
```

Use unique immutable tags (never `latest`). Pick a WDT version compatible with your operator release.

### Step 5: Optional ConfigMap overrides

`k8s/02-datasource-configmap.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: orders-wdt-config
  namespace: orders-ns
  labels:
    weblogic.domainUID: orders-domain
data:
  override.yaml: |
    resources:
      JDBCSystemResource:
        OrdersDS:
          JdbcResource:
            JDBCConnectionPoolParams:
              MaxCapacity: 40
```

### Step 6: Domain and Cluster

`k8s/03-domain.yaml`

```yaml
apiVersion: weblogic.oracle/v9
kind: Domain
metadata:
  name: orders-domain
  namespace: orders-ns
  labels:
    weblogic.domainUID: orders-domain
spec:
  domainUID: orders-domain
  domainHomeSourceType: FromModel
  image: container-registry.oracle.com/middleware/weblogic:14.1.1.0-11-ol8   # use your patched image
  imagePullPolicy: IfNotPresent
  imagePullSecrets:
    - name: ocr-secret
    - name: myreg-secret
  webLogicCredentialsSecret:
    name: orders-weblogic-credentials
  replicas: 3
  serverStartPolicy: IfNeeded
  restartVersion: "1"          # bump to force rolling restart
  introspectVersion: "1"       # bump to re-run introspection

  configuration:
    introspectorJobActiveDeadlineSeconds: 900
    model:
      domainType: WLS
      configMap: orders-wdt-config
      runtimeEncryptionSecret: orders-runtime-encryption
      auxiliaryImages:
        - image: <your-registry>/orders-aux:1.0.0
          imagePullPolicy: IfNotPresent
    secrets:
      - orders-db-secret

  adminServer:
    serverPod:
      resources:
        requests: { cpu: "500m", memory: "2Gi" }
        limits:   { memory: "2Gi" }

  clusters:
    - name: orders-domain-cluster-1

  serverPod:
    env:
      - name: JAVA_OPTIONS
        value: "-Dweblogic.StdoutDebugEnabled=false -XX:+UseG1GC -XX:MaxRAMPercentage=70"
      - name: USER_MEM_ARGS
        value: "-Djava.security.egd=file:/dev/./urandom"
    resources:
      requests: { cpu: "1", memory: "4Gi" }
      limits:   { memory: "4Gi" }        # requests=limits on memory avoids OOM surprises
    affinity:
      podAntiAffinity:
        preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              topologyKey: topology.kubernetes.io/zone
              labelSelector:
                matchLabels:
                  weblogic.clusterName: "$(CLUSTER_NAME)"
    readinessProbe:
      initialDelaySeconds: 30
      periodSeconds: 10
      failureThreshold: 3
    livenessProbe:
      initialDelaySeconds: 60
    shutdown:
      shutdownType: Graceful
      timeoutSeconds: 120
      ignoreSessions: false
    securityContext:
      runAsUser: 1000
      runAsNonRoot: true
```

`k8s/04-cluster.yaml`

```yaml
apiVersion: weblogic.oracle/v1
kind: Cluster
metadata:
  name: orders-domain-cluster-1
  namespace: orders-ns
  labels:
    weblogic.domainUID: orders-domain
spec:
  clusterName: cluster-1
  replicas: 3
  maxUnavailable: 1          # one pod at a time on rolling restarts; operator creates a PDB
```

```bash
kubectl apply -f k8s/02-datasource-configmap.yaml
kubectl apply -f k8s/03-domain.yaml -f k8s/04-cluster.yaml
```

### Step 7: Verify

```bash
kubectl -n orders-ns get domain,cluster
kubectl -n orders-ns get pods -w        # introspector -> admin -> managed servers
kubectl -n orders-ns describe domain orders-domain   # expect Available + Completed
```

### Step 8: Ingress (NGINX)

`k8s/05-ingress.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: orders-ingress
  namespace: orders-ns
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "ORDERS_ROUTE"
    nginx.ingress.kubernetes.io/session-cookie-samesite: "Lax"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "120"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [orders.example.com]
      secretName: orders-tls
  rules:
    - host: orders.example.com
      http:
        paths:
          - path: /orders
            pathType: Prefix
            backend:
              service:
                name: orders-domain-cluster-cluster-1
                port: { number: 8001 }
```

Never expose Admin Server (7001) or `/console` publicly. Use `kubectl port-forward` or an IP-restricted internal ingress.

### Step 9: Scaling

```bash
kubectl -n orders-ns scale cluster/orders-domain-cluster-1 --replicas=5   # max = DynamicClusterSize (6)
```

`k8s/06-hpa.yaml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orders-hpa
  namespace: orders-ns
spec:
  scaleTargetRef:
    apiVersion: weblogic.oracle/v1
    kind: Cluster
    name: orders-domain-cluster-1
  minReplicas: 3
  maxReplicas: 6
  metrics:
    - type: Resource
      resource:
        name: cpu
        target: { type: Utilization, averageUtilization: 65 }
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # WebLogic sessions are stateful; avoid flapping
```

### Step 10: Monitoring and logging

Monitoring exporter sidecar (operator 4.x):

```yaml
spec:
  monitoringExporter:
    image: ghcr.io/oracle/weblogic-monitoring-exporter:2.3.0
    configuration:
      metricsNameSnakeCase: true
      queries:
        - key: name
          keyName: location
          prefix: wls_servlet_
          servlets: { key: name, values: [invocationTotalCount, executionTimeAverage] }
```

Scrape port 8080 `/metrics` with Prometheus (ServiceMonitor). Logs: collect stdout with Fluent Bit / Promtail; for file logs use `logHome` + PV or `fluentdSpecification`.

---

## 5. Part B: OpenShift (OCP) deltas

Everything above applies; only the differences are listed here.

| Topic | Vanilla K8s | OpenShift |
|---|---|---|
| Pod UID | You set `runAsUser: 1000` | `restricted-v2` SCC assigns a random UID; fixed UID is rejected |
| Namespace | `kubectl create namespace` | Project: `oc new-project` |
| External access | Ingress + NGINX | Route (HAProxy router) |
| Registry | External | Internal registry + ImageStreams |
| Network | Optional NetworkPolicy | Often default-deny |
| Monitoring | Self-installed Prometheus | Built-in stack + user-workload monitoring |

### B1: Operator install

```bash
oc login https://api.<cluster>:6443 -u <user>
oc new-project weblogic-operator-ns
oc create serviceaccount weblogic-operator-sa -n weblogic-operator-ns
helm repo add weblogic-operator https://oracle.github.io/weblogic-kubernetes-operator/charts --force-update
```

`operator/values.yaml` (adds `kubernetesPlatform`)

```yaml
serviceAccount: weblogic-operator-sa
kubernetesPlatform: OpenShift
domainNamespaceSelectionStrategy: LabelSelector
domainNamespaceLabelSelector: weblogic-operator=enabled
replicas: 2
enableClusterRoleBinding: true
javaLoggingLevel: INFO
```

```bash
helm install weblogic-operator weblogic-operator/weblogic-operator \
  -n weblogic-operator-ns -f operator/values.yaml --wait
oc get crd | grep weblogic.oracle
oc get pod -n weblogic-operator-ns -o jsonpath='{.items[0].metadata.annotations.openshift\.io/scc}{"\n"}'   # expect restricted-v2
```

### B2: Project and secrets

```bash
oc new-project orders-ns
oc label namespace orders-ns weblogic-operator=enabled
# then create the same secrets as Step 2 using `oc -n orders-ns create secret ...`
```

### B3: Internal registry for the aux image

```bash
oc patch configs.imageregistry.operator.openshift.io/cluster --type=merge -p '{"spec":{"defaultRoute":true}}'
REG=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')

podman login -u $(oc whoami) -p $(oc whoami -t) $REG
podman build -t $REG/orders-ns/orders-aux:1.0.0 aux-image/
podman push  $REG/orders-ns/orders-aux:1.0.0
```

In-cluster reference:

```
image-registry.openshift-image-registry.svc:5000/orders-ns/orders-aux:1.0.0
```

For images in other projects, grant `system:image-puller` to the pulling service account.

### B4: Domain changes

Same as `03-domain.yaml`, except:

- `auxiliaryImages[].image` uses the internal registry address above
- **Remove** `runAsUser` / `runAsNonRoot` (let the SCC assign the UID)
- Add container security context and topology spread

```yaml
  serverPod:
    topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            weblogic.domainUID: orders-domain
            weblogic.clusterName: cluster-1
    containerSecurityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop: ["ALL"]
```

If your org requires the fixed UID 1000, use a dedicated ServiceAccount plus a narrowly scoped custom SCC (avoid `anyuid` / `privileged`):

```bash
oc create sa weblogic-domain-sa -n orders-ns
oc adm policy add-scc-to-user <your-custom-scc> -z weblogic-domain-sa -n orders-ns
# then set spec.serverPod.serviceAccountName: weblogic-domain-sa
```

### B5: Route (replaces Ingress)

`k8s/05-route.yaml`

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: orders
  namespace: orders-ns
  annotations:
    haproxy.router.openshift.io/timeout: 120s
    haproxy.router.openshift.io/balance: leastconn
    router.openshift.io/cookie_name: ORDERS_ROUTE
spec:
  host: orders.apps.<cluster-domain>
  path: /orders
  to:
    kind: Service
    name: orders-domain-cluster-cluster-1
    weight: 100
  port:
    targetPort: default          # verify: oc get svc -n orders-ns -o yaml
  tls:
    termination: edge            # or reencrypt
    insecureEdgeTerminationPolicy: Redirect
```

- Edge/reencrypt routes give cookie session affinity; passthrough uses source-IP affinity.
- For `reencrypt`: configure an SSL channel in the WDT model and supply `destinationCACertificate`.
- Never create a Route for the Admin Server; use `oc port-forward pod/orders-domain-admin-server 7001:7001`.

### B6: NetworkPolicy

`k8s/07-networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: orders-allow
  namespace: orders-ns
spec:
  podSelector:
    matchLabels:
      weblogic.domainUID: orders-domain
  policyTypes: [Ingress]
  ingress:
    - from:                                   # OpenShift router
        - namespaceSelector:
            matchLabels:
              network.openshift.io/policy-group: ingress
      ports: [{ port: 8001 }]
    - from:                                   # operator
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: weblogic-operator-ns
      ports: [{ port: 7001 }, { port: 8001 }]
    - from:                                   # intra-domain
        - podSelector:
            matchLabels:
              weblogic.domainUID: orders-domain
    - from:                                   # user-workload monitoring
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: openshift-user-workload-monitoring
      ports: [{ port: 8080 }]
```

### B7: Monitoring

```bash
oc apply -f - <<'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    enableUserWorkload: true
EOF
```

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: orders-wls
  namespace: orders-ns
spec:
  selector:
    matchLabels:
      weblogic.domainUID: orders-domain
  endpoints:
    - port: metrics        # confirm port name in your exporter setup
      interval: 30s
```

Metrics appear under **Observe -> Metrics** in the console.

---

## 6. Day-2 operations cheat sheet

| Change | How | Operator behavior |
|---|---|---|
| New app version | Build `orders-aux:1.1.0`, update `auxiliaryImages.image`, apply | Introspects, then rolling restart (`maxUnavailable`) |
| WebLogic patch (PSU) | Build patched base image, update `spec.image` | Rolling restart |
| Model/ConfigMap change | Edit ConfigMap, bump `introspectVersion` | Online update if `configuration.model.onlineUpdate.enabled: true`, else rolling restart |
| Force restart | Bump `restartVersion` | Rolling restart |
| JVM args / resources | Edit `serverPod` | Rolling restart |
| Scale | `kubectl scale cluster/... --replicas=N` or HPA | Starts/stops managed servers |

```bash
# Rolling update to a new aux image
kubectl -n orders-ns patch domain orders-domain --type=merge -p \
  '{"spec":{"configuration":{"model":{"auxiliaryImages":[{"image":"<your-registry>/orders-aux:1.1.0"}]}}}}'
kubectl -n orders-ns get pods -w
```

Operator upgrades: `helm upgrade` the operator first (running domains keep running), then roll domains onto new images. Test in staging.

## 7. Troubleshooting

```bash
kubectl -n orders-ns get events --sort-by=.lastTimestamp
kubectl -n orders-ns logs job/orders-domain-introspector     # model errors show here
kubectl -n weblogic-operator-ns logs deploy/weblogic-operator
kubectl -n orders-ns describe domain orders-domain
```

| Symptom | Likely cause / fix |
|---|---|
| Introspector fails | WDT model syntax, missing secret, wrong `@@SECRET:name:key@@` |
| `ImagePullBackOff` | Missing `imagePullSecrets`; on OCP wrong registry hostname or missing `system:image-puller` |
| Managed servers won't start beyond N | `replicas` exceeds `DynamicClusterSize` |
| `OOMKilled` | Heap exceeds container limit; lower `MaxRAMPercentage` |
| OCP: "unable to validate against any SCC" | Hard-coded `runAsUser`/`fsGroup`; remove it or use custom SCC + dedicated SA |
| OCP Route 503 | No ready endpoints, or wrong `targetPort` name |
| OCP sessions bounce | Passthrough route or cookies disabled; use edge/reencrypt |
| Introspector times out (OCP) | NetworkPolicy blocking operator namespace |
| Permission denied under `/u01` (OCP) | UID's group is not 0 or `fsGroup` overridden |

## 8. Production hardening checklist

- [ ] **Networking:** NetworkPolicies (ingress -> 8001, operator -> 7001/8001, pods -> DB)
- [ ] **TLS:** custom identity keystore, or terminate at ingress/route and re-encrypt to pods
- [ ] **RBAC:** operator restricted to labeled namespaces; lock down Domain CR access
- [ ] **Secrets:** external secret manager (Vault / External Secrets / CSI); rotate runtime encryption secret carefully
- [ ] **Pod security:** non-root, drop capabilities, `readOnlyRootFilesystem` where feasible, PSA `restricted`/`baseline`
- [ ] **Capacity:** memory requests = limits, heap 60-70% of limit, avoid tight CPU limits
- [ ] **HA:** 2+ operator replicas, multi-zone spread, PDB via `maxUnavailable`, DB failover (RAC / Active Data Guard service URL)
- [ ] **Backup/DR:** model + images in Git/registry; JMS/JTA file stores on PVs (or JDBC stores)
- [ ] **Supply chain:** scan and sign images, pin digests
- [ ] **GitOps:** `k8s/` in Git, deployed via Argo CD / Flux (OpenShift GitOps on OCP)
- [ ] **Dedicated project/namespace per domain**; developers get `edit` on the domain project only

## 9. Alternatives and next steps

- **Domain-on-PV** (`domainHomeSourceType: PersistentVolume` / `initializeDomainOnPV`): use when you must keep an existing domain with runtime-persisted state. MII remains the recommended immutable approach.
- Ideas to add later: Tekton / GitHub Actions pipeline (build aux image, push, patch Domain), Argo CD Application manifest, Istio/Traefik variant.

## References

- WebLogic Kubernetes Operator docs: https://oracle.github.io/weblogic-kubernetes-operator/
- WebLogic Deploy Tooling: https://github.com/oracle/weblogic-deploy-tooling
- WebLogic Monitoring Exporter: https://github.com/oracle/weblogic-monitoring-exporter