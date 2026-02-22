# L4: Infrastructure & DevOps

> The operational layer that runs everything. Understanding these lets you design systems that are deployable, observable, and resilient.

---

## 1. Containerization

### Docker Internals

**Linux Kernel Primitives**
- **Namespaces**: isolate processes from seeing each other
  - `pid`: isolated process IDs (init = PID 1 inside container)
  - `net`: separate network stack per container
  - `mnt`: isolated filesystem mount points
  - `uts`: independent hostname/domainname
  - `ipc`: isolated inter-process communication
  - `user`: map container root to non-root on host
  - `cgroup` (Linux 4.6+): separate cgroup hierarchy
- **cgroups (Control Groups)**: resource limits and accounting
  - CPU: shares, quotas, cpuset binding
  - Memory: hard limits, OOM kill, swap limits
  - Block I/O: bandwidth throttling
  - Network: traffic shaping (via tc)

**Container Filesystem**
- **OverlayFS**: union filesystem stacking layers
  - Lower layers: read-only (base image layers)
  - Upper layer: read-write (container's changes)
  - Copy-on-write: file copied to upper layer before modification
- **Image layers**: each Dockerfile instruction creates a layer
  - Layers are content-addressed (SHA256 hash)
  - Shared across containers → significant disk space savings

**Docker Networking Modes**
- `bridge` (default): private virtual network, NAT to host
- `host`: container shares host's network namespace (zero overhead)
- `overlay`: multi-host networking (Docker Swarm/Compose)
- `macvlan`: container gets its own MAC address on physical network
- `none`: no networking

**Container Security**
- Rootless containers: run entire daemon as non-root
- **seccomp profiles**: whitelist allowed system calls (default blocks ~44 syscalls)
- **AppArmor/SELinux profiles**: MAC security policies
- **Capabilities**: fine-grained privilege (don't run with NET_ADMIN unless needed)
- Image scanning: Trivy, Snyk, Clair — check for CVEs in base images

---

## 2. Kubernetes

### Architecture

```
Control Plane:                      Worker Nodes:
┌─────────────────────┐            ┌──────────────────┐
│  kube-apiserver     │◄──────────►│  kubelet         │
│  etcd               │            │  kube-proxy      │
│  kube-scheduler     │            │  Container Runtime│
│  controller-manager │            └──────────────────┘
└─────────────────────┘
```

**kube-apiserver**: REST API, validates requests, persists to etcd
**etcd**: all cluster state (pods, services, deployments, secrets, etc.)
**kube-scheduler**: assigns Pods to Nodes (considers resource requests, affinity, taints)
**controller-manager**: runs reconciliation loops (ReplicaSet controller, Deployment controller, etc.)
**kubelet**: node agent, ensures containers are running as described
**kube-proxy**: maintains iptables/IPVS rules for Service networking

### Core Workload Objects

**Pod**: smallest deployable unit, one or more containers sharing network/storage
- All containers in a pod share the same IP address
- Init containers: run before app containers start
- Sidecar containers: helper containers (logging, proxy, etc.)

**ReplicaSet**: ensures N pod replicas running at all times

**Deployment**: manages ReplicaSets, rolling updates, rollbacks

**StatefulSet**: for stateful apps (databases, caches)
- Stable network identity (pod-0, pod-1...)
- Stable persistent storage per pod
- Ordered, graceful deployment and scaling

**DaemonSet**: run one pod per node (log collectors, monitoring agents, CNI plugins)

**Job / CronJob**: run-to-completion workloads

### Services & Networking

**Service types**:
- `ClusterIP`: internal-only virtual IP (default)
- `NodePort`: expose on every node's IP at a port
- `LoadBalancer`: provision cloud load balancer (AWS ELB, GCP LB)
- `ExternalName`: CNAME alias to external service

**Ingress**: L7 HTTP routing — path-based, host-based routing, TLS termination
- Controllers: NGINX Ingress, Traefik, Istio Gateway, AWS ALB Ingress

**DNS in Kubernetes**:
- CoreDNS: cluster DNS server
- Service DNS: `svc-name.namespace.svc.cluster.local`
- Pod DNS: `pod-ip.namespace.pod.cluster.local`

**Network Policies**: define allowed pod-to-pod communication (requires CNI that supports it)

**CNI Plugins** (Container Network Interface):
- Flannel: simple VXLAN overlay
- Calico: BGP-based, supports NetworkPolicy, high performance
- Cilium: eBPF-based, L7 policies, service mesh without sidecar
- Weave: mesh networking

### Storage

**Persistent Volume (PV)**: cluster-level storage resource
**Persistent Volume Claim (PVC)**: pod's request for storage
**Storage Classes**: dynamic provisioning (creates PV on demand)
- StorageClass examples: `gp3` (AWS EBS), `pd-ssd` (GCP Persistent Disk), `local-path`

**Volume types**: emptyDir, hostPath, configMap, secret, NFS, CSI (Container Storage Interface)

### Autoscaling

**HPA (Horizontal Pod Autoscaler)**: scale pod count based on CPU/memory/custom metrics
**VPA (Vertical Pod Autoscaler)**: adjust pod resource requests/limits
**KEDA (Kubernetes Event-Driven Autoscaler)**: scale based on external events (queue depth, etc.)
**Cluster Autoscaler**: add/remove nodes based on pending pods

### RBAC

- **ServiceAccount**: identity for pods
- **Role / ClusterRole**: set of permissions (verbs on resources)
- **RoleBinding / ClusterRoleBinding**: grant role to user/group/serviceaccount
- **Principle of least privilege**: give only what's needed

### Kubernetes Scheduler Internals

1. **Filtering**: eliminate nodes that can't run the pod (resources, taints, node selectors, affinity)
2. **Scoring**: rank remaining nodes (spread pods, prefer node with image cached, etc.)
3. **Binding**: assign pod to highest-scoring node

**Node Affinity/Anti-affinity**: prefer/require nodes with certain labels
**Pod Affinity/Anti-affinity**: co-locate or spread pods based on other pod labels
**Taints and Tolerations**: mark nodes as special (GPU nodes, spot nodes), require pods to tolerate

### Advanced Kubernetes Patterns

**Operators**:
- Extend Kubernetes with custom controllers for complex stateful applications
- CRD (Custom Resource Definition) + Controller = Operator
- Examples: Prometheus Operator, Cert-Manager, Strimzi (Kafka), CloudNativePG

**Helm Charts**:
- Package manager for Kubernetes manifests
- Templates + values.yaml

**Multi-cluster Management**:
- **Argo CD**: GitOps continuous delivery tool
- **Flux CD**: GitOps operator
- **KubeVela**: multi-cluster application delivery

---

## 3. Service Mesh

### What It Solves
- Service-to-service communication: retries, timeouts, circuit breaking
- mTLS encryption without app code changes
- Distributed tracing without app code changes
- Traffic management: canary, blue-green, A/B testing

### Istio Architecture

**Data plane**: Envoy proxy injected as sidecar into every pod
- Intercepts all inbound/outbound traffic
- Handles: load balancing, circuit breaking, retries, TLS, telemetry

**Control plane** (Istiod):
- **Pilot**: pushes routing rules to Envoy (xDS protocol)
- **Citadel**: certificate authority (issues and rotates mTLS certs)
- **Galley**: configuration validation

**Key Resources**:
- `VirtualService`: routing rules (match header/path → route to specific version)
- `DestinationRule`: load balancing policy, circuit breaker, mTLS settings per destination
- `Gateway`: expose services outside mesh
- `AuthorizationPolicy`: L4/L7 authorization rules

### Linkerd (Lightweight Alternative)
- Much simpler than Istio, less features
- Rust-based sidecar (lighter than Envoy)
- Automatic mTLS, retries, timeouts
- No complex configuration language

### eBPF-Based (Cilium)
- No sidecar proxy — uses eBPF programs injected into Linux kernel
- Lower latency and overhead than sidecar model
- Deep visibility into network traffic at kernel level
- Works with or without full service mesh

---

## 4. Cloud Architecture

### AWS Core Services
- **Compute**: EC2, Lambda (serverless), ECS (containers), EKS (Kubernetes), Fargate (serverless containers)
- **Storage**: S3 (object), EBS (block), EFS (NFS), Glacier (archive)
- **Database**: RDS (managed SQL), Aurora (MySQL/PostgreSQL-compatible, 5× throughput), DynamoDB, ElastiCache (Redis/Memcached), Redshift (data warehouse)
- **Networking**: VPC, Route 53, CloudFront, ALB/NLB, API Gateway, Direct Connect
- **Messaging**: SQS (queue), SNS (pub/sub), Kinesis (streaming), EventBridge
- **Analytics**: EMR (Hadoop/Spark), Glue (ETL), Athena (query S3), QuickSight

### Availability Zones and Regions
- **Region**: geographic area with multiple AZs (us-east-1)
- **Availability Zone**: one or more discrete data centers in a region (us-east-1a, 1b, 1c)
- **Design for AZ failure**: spread across 2–3 AZs
- **Design for region failure**: use multi-region active-passive or active-active

### Multi-Region Patterns
- **Active-Passive**: all writes go to primary region, secondary is hot standby
  - Failover: update DNS to point to secondary (60s–5min depending on TTL)
- **Active-Active**: writes accepted in multiple regions, replicated async
  - Requires conflict resolution
  - Examples: DynamoDB Global Tables, CockroachDB multi-region

### Disaster Recovery
- **RTO (Recovery Time Objective)**: how long to restore service
- **RPO (Recovery Point Objective)**: how much data you can afford to lose
```
Strategy        | RTO      | RPO      | Cost
Backup/restore  | Hours    | Hours    | Low
Pilot light     | 10 min   | Minutes  | Medium
Warm standby    | Minutes  | Seconds  | High
Multi-region    | Seconds  | ~0       | Very High
```

### 12-Factor App (Heroku)
1. Codebase: one codebase, many deploys
2. Dependencies: explicitly declare and isolate
3. Config: store in environment (not code)
4. Backing services: treat as attached resources
5. Build, Release, Run: strictly separate stages
6. Processes: stateless, share-nothing
7. Port binding: export services via port binding
8. Concurrency: scale out via process model
9. Disposability: fast startup, graceful shutdown
10. Dev/Prod parity: keep environments similar
11. Logs: treat as event streams
12. Admin processes: run as one-off processes

---

## 5. Infrastructure as Code

### Terraform
- **Providers**: plugins for AWS, GCP, Azure, Kubernetes, etc.
- **Resources**: infrastructure objects (`aws_instance`, `kubernetes_deployment`)
- **Modules**: reusable infrastructure components
- **State**: tracks deployed resources (store in S3 + DynamoDB lock for teams)
- **Plan/Apply**: preview changes before applying
- **Workspaces**: multiple environments (dev, staging, prod)
- Remote state sharing between modules

### Pulumi
- General-purpose languages (TypeScript, Python, Go)
- Same Terraform concepts but with real programming logic
- Better for complex conditional infrastructure

### Ansible
- Agentless configuration management (SSH-based)
- YAML playbooks: tasks, roles, handlers
- Idempotent: running twice gives same result
- Use for: VM configuration, OS setup, application deployment (not infrastructure creation)

### GitOps
- Infrastructure state defined in Git
- Automated sync: cluster state should match Git state
- **Argo CD**: pull-based, watches Git repo, applies changes to K8s
- **Flux CD**: similar, CNCF project
- Benefits: audit log, rollback via Git revert, PR review for infra changes

---

## 6. CI/CD

### Deployment Strategies

**Blue-Green Deployment**:
- Two identical environments (blue = current, green = new)
- Route traffic to green after validation
- Instant rollback: flip back to blue
- Cost: doubles infrastructure during deployment

**Canary Deployment**:
- Gradually shift traffic to new version (1% → 5% → 25% → 100%)
- Monitor error rates at each step
- Automatic rollback if error budget exceeded
- Requires traffic splitting at load balancer level

**Rolling Update**:
- Replace old pods with new gradually
- Kubernetes default for Deployments
- maxSurge and maxUnavailable control speed

**Shadow Deployment (Traffic Mirroring)**:
- Mirror production traffic to new version without serving responses
- Validate new version behavior under real load
- Useful for database migrations, ML model changes

**Feature Flags**:
- Decouple deployment from release
- Enable for specific users/percentage
- A/B test in production
- Kill switch for problematic features

### Pipeline Security
- **SBOM** (Software Bill of Materials): list all dependencies and versions
- **SLSA** (Supply chain Levels for Software Artifacts): framework for supply chain integrity
  - Level 1: documented build process
  - Level 4: hermetic, reproducible builds, two-party review
- **Sigstore**: sign artifacts (container images, binaries) with ephemeral keys linked to OIDC identity
- Dependency vulnerability scanning in CI (Dependabot, Renovate)
- Secret scanning (git-secrets, GitGuardian)

---

## 7. Observability

> "The three pillars" — but they're not equal. Traces are most powerful for debugging, metrics for alerting, logs for investigation.

### Logs

**Structured Logging**:
- JSON format: machine-parseable, enables field-based queries
- Include: timestamp, severity, service name, trace ID, request ID, user ID, message
- Correlation ID: single ID threaded through all services for a request

**Log Aggregation**:
- **ELK Stack**: Elasticsearch + Logstash (pipeline) + Kibana (UI)
- **Grafana Loki**: log aggregation system, doesn't index content (only labels) — cheap
- **Fluentd / Fluent Bit**: log collector/forwarder (DaemonSet in K8s)
- **Vector**: high-performance log router (Rust-based)

**Log Levels**: TRACE → DEBUG → INFO → WARN → ERROR → FATAL

### Metrics

**Metric Types (Prometheus)**:
- **Counter**: monotonically increasing (requests_total, errors_total)
- **Gauge**: can go up or down (memory_usage_bytes, active_connections)
- **Histogram**: sample distribution with configurable buckets (request_duration_seconds)
- **Summary**: similar to histogram but client-side quantiles

**Prometheus Architecture**:
- Pull-based: scrapes `/metrics` endpoint at interval (default 15s)
- TSDB: custom time-series storage, ~1.3 bytes/sample
- **Recording rules**: pre-compute expensive queries for dashboards
- **Alerting rules**: fire alerts based on PromQL expressions
- **Alertmanager**: deduplication, grouping, routing, silencing

**PromQL Examples**:
```
# Request rate per second (5m window)
rate(http_requests_total[5m])

# 99th percentile latency
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Error ratio
rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m])
```

### Distributed Tracing

**OpenTelemetry** (OTel): vendor-neutral standard for traces, metrics, logs
- **Trace**: end-to-end request journey
- **Span**: single operation with timing, attributes, events
- **Context propagation**: trace ID passed between services via headers (W3C TraceContext standard)
- **Sampling**: head-based (decide at start), tail-based (decide after completion — keeps interesting traces)

**Backends**: Jaeger, Zipkin, Grafana Tempo, AWS X-Ray, Honeycomb

### SLI, SLO, SLA

**SLI (Service Level Indicator)**: measured metric (99th percentile latency, availability)
**SLO (Service Level Objective)**: target for SLI (latency < 200ms for 99% of requests)
**SLA (Service Level Agreement)**: contract with customers (often less strict than SLO, with financial penalties)

**Error Budget**:
- Error budget = 1 - SLO target (e.g., 99.9% → 0.1% = 43.8 min/month)
- Spend error budget on risky deployments, experiments
- When budget exhausted: freeze new features, focus on reliability

### Chaos Engineering

**Principles**:
- Define steady state (metrics that indicate normal behavior)
- Hypothesize that steady state continues under failure
- Introduce failure (chaos)
- Validate hypothesis

**Tools**:
- **Chaos Monkey**: randomly terminates EC2 instances (Netflix)
- **Chaos Mesh**: K8s-native chaos engineering (pod kill, network delay, disk I/O)
- **Litmus**: CNCF chaos framework
- **Gremlin**: commercial, comprehensive failure scenarios

**Game Days**: scheduled chaos experiments with team present to practice incident response
