# Docker, Kubernetes & Terraform — Intermediate Tier (Kubernetes)

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Prerequisites:** Docker basics, comfortable with containers  
> **Estimated time:** 8–10 hours

---

## Table of Contents

1. Kubernetes Architecture
2. Pods
3. Workloads (Deployment, StatefulSet, DaemonSet, Job)
4. Services
5. Ingress
6. ConfigMaps and Secrets
7. Storage (PV, PVC, StorageClass)
8. Probes (Liveness, Readiness, Startup)
9. Resource Management
10. Namespaces
11. kubectl Essentials
12. Q&A

---

## 1. Kubernetes Architecture

```
┌──────────────────────────────────────────────────────┐
│                  Control Plane                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ API      │  │ Scheduler│  │ Controller        │   │
│  │ Server   │  │          │  │ Manager           │   │
│  └────┬─────┘  └──────────┘  └──────────────────┘   │
│       │ etcd (distributed key-value store)           │
│       └─────────────────────────────────────────┐    │
└─────────────────────────────────────────────────┼────┘
                                                  │
    ┌─────────────────────────────────────────────┼─────────┐
    │              Worker Node 1                  │         │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │         │
    │  │ kubelet  │  │ kube-proxy│  │ runtime  │  │         │
    │  └──────────┘  └──────────┘  └──────────┘  │         │
    │  ┌──────────┐  ┌──────────┐  ┌──────────┐  │         │
    │  │ Pod      │  │ Pod      │  │ Pod      │  │         │
    │  │ (app)    │  │ (sidecar)│  │ (app)    │  │         │
    │  └──────────┘  └──────────┘  └──────────┘  │         │
    └─────────────────────────────────────────────┘         │
                                                            │
    ┌─────────────────────────────────────────────┐         │
    │              Worker Node N                  │         │
    │  ... same structure ...                     │         │
    └─────────────────────────────────────────────┘         │
```

### Control plane components

| Component | Description |
|-----------|-------------|
| **API Server** | Entry point for all K8s operations (`kubectl apply` hits this). Authenticates, authorizes, validates requests. |
| **etcd** | Distributed key-value store. The single source of truth (cluster state, configs, secrets). |
| **Scheduler** | Assigns Pods to Nodes based on resource requirements, constraints, affinity/anti-affinity. |
| **Controller Manager** | Runs controllers (Deployment, ReplicaSet, Node, ServiceAccount, Endpoint). Watches state and reconciles desired → actual. |

### Worker node components

| Component | Description |
|-----------|-------------|
| **kubelet** | Agent running on every node. Registers node, manages Pod lifecycle, reports status to API server. |
| **kube-proxy** | Network proxy. Maintains network rules (iptables/IPVS). Forwards traffic to Pods. |
| **Container runtime** | Runs containers (containerd, CRI-O). |

> **Trap:** etcd is the single source of truth. If etcd is corrupted or unreachable, the cluster cannot operate. Always backup etcd. On EKS, AWS manages etcd for you.

---

## 2. Pods

A Pod is the smallest deployable unit in Kubernetes. It wraps one or more containers that share:

- Network namespace (same IP, port range)
- Storage volumes
- Lifecycle (start/stop together)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: api-pod
  labels:
    app: api
    environment: production
spec:
  containers:
    - name: api
      image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/api:1.2.3
      ports:
        - containerPort: 8080
          protocol: TCP
      env:
        - name: DB_HOST
          value: "database.internal"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: password
      resources:
        requests:
          memory: "256Mi"
          cpu: "250m"
        limits:
          memory: "512Mi"
          cpu: "500m"
      readinessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 5
        periodSeconds: 10
      livenessProbe:
        httpGet:
          path: /health
          port: 8080
        initialDelaySeconds: 15
        periodSeconds: 20
```

> **Trap:** Don't create Pods directly in production. Use Deployments or other controllers that self-heal (restart on failure, scale, rollback).

### Pod lifecycle

```
Pending → Running → Succeeded / Failed
  ↑          │
  └──────────┘ (CrashLoopBackOff → backoff restart)
```

| Phase | Description |
|-------|-------------|
| `Pending` | Image being pulled, node being scheduled |
| `Running` | All containers started |
| `Succeeded` | Containers exited with code 0 (Job) |
| `Failed` | At least one container exited with non-zero |
| `CrashLoopBackOff` | Container keeps crashing, K8s is backing off restart attempts |
| `ImagePullBackOff` | Can't pull image (wrong name, auth error, registry unreachable) |

---

## 3. Workloads

### Deployment

Stateless, self-healing, scalable. The most common workload.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app: api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1      # max pods unavailable during update
      maxSurge: 1            # max extra pods during update
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/api:1.2.3
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
```

### Deployment strategies

| Strategy | Description | Use case |
|----------|-------------|----------|
| **RollingUpdate** | Gradual replacement (default). `maxUnavailable` and `maxSurge` control speed. | General purpose |
| **Recreate** | Delete all pods, then recreate. Causes downtime. | Dev/test, DB migrations |

### StatefulSet

For stateful applications (databases) with stable network identity and persistent storage:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Gi
```

StatefulSet guarantees:
- Pods have stable hostnames: `postgres-0`, `postgres-1`, `postgres-2`
- Pods are created/deleted in order
- Each pod gets its own PVC (persistent storage)

> **Trap:** StatefulSet does NOT handle backups, failover, or replication. It just provides stable storage + identity. For a production DB, use an operator (PostgreSQL Operator, Zalando Postgres Operator) or managed service (RDS, Aurora).

### DaemonSet

Runs one pod per node (for logging, monitoring, networking):

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
        - name: fluentd
          image: fluent/fluentd:v1.16
          volumeMounts:
            - name: varlog
              mountPath: /var/log
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
```

### Job and CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
spec:
  schedule: "0 3 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
            - name: backup
              image: postgres:16-alpine
              command: ["pg_dump", "-h", "postgres", "-U", "app", "inventory"]
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: db-secret
                      key: password
          restartPolicy: Never
      backoffLimit: 3
```

---

## 4. Services

A Service provides stable networking to Pods (which are ephemeral and have changing IPs).

### Service types

| Type | External access | Internal DNS | Use case |
|------|----------------|--------------|----------|
| **ClusterIP** | No (cluster-internal only) | `service-name.namespace.svc.cluster.local` | Internal APIs, backend services |
| **NodePort** | `<NodeIP>:<NodePort>` (static port, 30000–32767) | Same | Dev/test, direct node access |
| **LoadBalancer** | Cloud LB (ALB/NLB) | Same | Production external services |
| **ExternalName** | CNAME to external DNS | `service-name.namespace.svc.cluster.local` → external host | In-cluster access to external services |

### ClusterIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  selector:
    app: api
  ports:
    - protocol: TCP
      port: 80         # service port
      targetPort: 8080  # container port
  type: ClusterIP
```

Other pods access this service via `http://api-service:80` (or full DNS: `api-service.default.svc.cluster.local`).

### NodePort

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: NodePort
  selector:
    app: api
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080  # optional (auto-assigned if omitted)
```

Access via `http://node-ip:30080`.

### LoadBalancer

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
    - port: 443
      targetPort: 8080
```

On EKS, this provisions an NLB automatically.

> **Trap:** Each LoadBalancer service creates a new cloud LB. This is expensive. For HTTP traffic, use a single Ingress with path-based routing to multiple services.

---

## 5. Ingress

Ingress provides HTTP/HTTPS routing to multiple services through a single Load Balancer.

### Ingress with ALB (EKS)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/abc-123
    alb.ingress.kubernetes.io/success-codes: "200"
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /v1/*
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
          - path: /admin/*
            pathType: Prefix
            backend:
              service:
                name: admin-service
                port:
                  number: 8080
    - host: internal.example.com
      http:
        paths:
          - path: /*
            pathType: Prefix
            backend:
              service:
                name: internal-service
                port:
                  number: 9090
```

> **Trap:** Without the ALB Ingress Controller (AWS Load Balancer Controller), Ingress resources on EKS do nothing. The controller watches Ingress resources and provisions ALBs accordingly.

---

## 6. ConfigMaps and Secrets

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  APP_DEBUG: "false"
  DB_HOST: postgres.default
  DB_PORT: "5432"
```

**Consumption options:**

```yaml
# 1. Environment variables
envFrom:
  - configMapRef:
      name: app-config

# 2. Specific keys
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: DB_HOST

# 3. Volume mount
volumes:
  - name: config
    configMap:
      name: app-config
```

> **Trap:** ConfigMap data is stored unencrypted in etcd. Anyone with etcd access can read it. For sensitive data, use Secrets.

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_PASSWORD: supersecret123
```

> **Trap:** Secrets in K8s are base64-encoded, NOT encrypted by default. Base64 is not encryption. Enable encryption at rest for etcd (KMS provider) or use external secret stores (External Secrets Operator, AWS Secrets Manager CSI driver).

### External Secrets with K8s

Better approach: use External Secrets Operator to sync AWS Secrets Manager → K8s Secrets:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: db-secret
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: db-secret
  data:
    - secretKey: DB_PASSWORD
      remoteRef:
        key: production/rds/password
```

This keeps secrets out of Git and syncs automatically.

---

## 7. Storage (PV, PVC, StorageClass)

### Conceptual model

```
Pod → PVC → PV → Storage → StorageClass
                                │
                           (AWS EBS / EFS / EBS CSI)
```

| Resource | Description |
|----------|-------------|
| **PV (PersistentVolume)** | Storage provisioned (EBS volume, EFS) |
| **PVC (PersistentVolumeClaim)** | Request for storage by a Pod |
| **StorageClass** | Defines storage type (gp3, io2, EFS) |

### PVC example

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  storageClassName: gp3
```

| Access mode | Meaning |
|-------------|---------|
| `ReadWriteOnce` (RWO) | Single node read-write (EBS) |
| `ReadOnlyMany` (ROX) | Multiple nodes read-only |
| `ReadWriteMany` (RWX) | Multiple nodes read-write (EFS) |

> **Trap:** EBS only supports RWO. If you need RWX (multiple pods reading/writing), use EFS or a distributed filesystem.

---

## 8. Probes (Liveness, Readiness, Startup)

| Probe | Purpose | On failure |
|-------|---------|------------|
| **Liveness** | Is the app running? (stuck, deadlocked) | Container restarts |
| **Readiness** | Is the app ready to serve traffic? | Removed from Service endpoints |
| **Startup** | Is the app started? (for slow-starting apps) | Delays liveness checks until ready |

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
    httpHeaders:
      - name: X-Custom-Header
        value: health-check
  initialDelaySeconds: 5   # wait before first check
  periodSeconds: 10         # check every 10s
  timeoutSeconds: 3         # probe timeout
  successThreshold: 1       # consecutive successes to become healthy
  failureThreshold: 3       # consecutive failures to become unhealthy

livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 15

startupProbe:
  httpGet:
    path: /startup
    port: 8080
  initialDelaySeconds: 3
  periodSeconds: 5
  failureThreshold: 30      # 30 × 5s = 150s to start
```

> **Trap:** A liveness probe that depends on an external service (DB, cache, external API) will restart your containers when that dependency is slow. Liveness should check LOCAL application health only. External dependency health goes in readiness.

---

## 9. Resource Management

### Requests vs Limits

| | Requests | Limits |
|---|---|---|
| **Purpose** | Scheduling guarantee | Throttling/OOM protection |
| **CPU behavior** | Guaranteed share | Throttled at limit |
| **Memory behavior** | Guaranteed | OOM killed if exceeded |
| **Scheduling** | Node must have available capacity | Ignored for scheduling |

```yaml
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"      # 1/4 CPU core
  limits:
    memory: "512Mi"
    cpu: "500m"       # 1/2 CPU core
```

**QoS Classes:**

| Class | Condition | Behavior |
|-------|-----------|----------|
| **Guaranteed** | `limits == requests` (both set for CPU + memory) | Highest priority, never OOM unless exceeds limit |
| **Burstable** | At least one resource with `requests < limits` | Medium priority, may be OOM-killed if node under memory pressure |
| **BestEffort** | No requests or limits set | Lowest priority, first to be OOM-killed |

> **Trap:** Not setting resource limits is dangerous. A memory leak in one pod can OOM-kill other pods on the same node. Always set both requests and limits.

### Resource Quotas (per namespace)

Limit total resources consumed by a namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
    services: "10"
```

---

## 10. Namespaces

Logical isolation for multi-tenant clusters:

```bash
# Create namespace
kubectl create namespace staging

# List resources in namespace
kubectl get pods -n staging

# Set default namespace
kubectl config set-context --current --namespace=staging
```

Common namespaces:
- `default` — resources without namespace
- `kube-system` — system pods (CoreDNS, kube-proxy, metrics-server)
- `kube-public` — readable by all users (ClusterInfo)
- `kube-node-lease` — node heartbeat leases

> **Trap:** Namespaces do NOT provide network isolation. Pods in different namespaces can communicate unless NetworkPolicies are applied.

---

## 11. kubectl Essentials

```bash
# Workloads
kubectl get pods -o wide                              # detailed pod info
kubectl get deployments                                # deployments
kubectl get services                                   # services
kubectl get events --sort-by='.lastTimestamp'          # recent events

# Describe (detailed info)
kubectl describe pod api-pod-7f4c5d6b9c-abc12          # pod details + recent events

# Logs
kubectl logs -f deployment/api                        # tail logs from deployment
kubectl logs pod/api-pod-7f4c5d6b9c-abc12 -c sidecar  # multi-container: specify container

# Exec
kubectl exec -it deployment/api -- bash                # shell into one pod of deployment
kubectl exec pod/api-pod-7f4c5d6b9c-abc12 -- php artisan cache:clear

# Port forwarding (for debugging)
kubectl port-forward service/api-service 8080:8080

# Apply/Delete
kubectl apply -f deployment.yaml                       # create or update
kubectl delete -f deployment.yaml                      # delete
kubectl delete pod --all -n staging                    # delete all pods in namespace
kubectl delete pod --field-selector=status.phase=Succeeded

# Rollout
kubectl rollout status deployment/api                  # watch rollout
kubectl rollout history deployment/api                 # revision history
kubectl rollout undo deployment/api --to-revision=2    # rollback

# Debug
kubectl run debug --image=nicolaka/netshoot:latest -it --rm -- /bin/bash  # network debug pod
kubectl get events -n production --sort-by='.lastTimestamp'
kubectl top pod                                        # resource usage
kubectl top node                                       # node resource usage

# Context
kubectl config get-contexts
kubectl config use-context production
kubectl config set-context --current --namespace=production
```

---

## 12. Q&A

### Intermediate

**Q: What's the difference between a Deployment and a StatefulSet?**
A: Deployment is for stateless apps (all pods are identical, any pod can serve any request). StatefulSet is for stateful apps (stable identity, ordered creation/deletion, each pod has its own PVC).

**Q: How does a Service route traffic to Pods?**
A: The Service has a label selector. kube-proxy creates iptables/IPVS rules to distribute traffic across matching pod IPs.

**Q: What's the difference between readiness and liveness probes?**
A: Readiness: is the app ready to serve traffic? (removed from Service). Liveness: is the app alive? (restart container). Readiness failures don't restart; liveness failures do.

**Q: What happens when a container exceeds its memory limit?**
A: It's OOM-killed by the kernel. K8s restarts it (restartPolicy: Always). If it keeps crashing → CrashLoopBackOff.

**Q: What's the purpose of a ServiceAccount?**
A: Identity for Pods to authenticate with the K8s API. Often paired with RBAC roles.

**Q: How do you update a Deployment without downtime?**
A: RollingUpdate strategy. New pods are created, old pods removed, all while maintaining at least `maxUnavailable` and at most `maxSurge` pods.

**Q: What's a DaemonSet used for?**
A: Run one pod per node for infrastructure: logging (Fluentd), monitoring (Prometheus node exporter), networking (Calico).

### Trap questions

**Q: You update a Deployment but the new Pods keep crashing. What happens?**
A: The Deployment controller detects crash loop (CrashLoopBackOff). Rolling update stops with old pods still running (if using RollingUpdate with maxUnavailable=0, the update stalls). Run `kubectl rollout undo` to rollback.

**Q: Two Pods need to communicate but get "connection refused." What do you check?**
A: (1) Are they in the same namespace? (2) Does the Service selector match the target Pod labels? (3) Are the Pods ready (readiness probe passing)? (4) Is there a NetworkPolicy blocking traffic? (5) Check `kubectl get endpoints` — does the Service have endpoints?

**Q: A Pod is stuck in `Pending` state. What do you check?**
A: (1) `kubectl describe pod` → events show why. (2) Node capacity — no node with enough CPU/memory. (3) Pod requests > available resources. (4) PVC not bound (PV pending). (5) Image pull error (wrong registry, no auth).

**Q: You scale a Deployment to 10 replicas but only 8 Pods are Running. What's the likely issue?**
A: Resource constraints — 8 Pods were scheduled, 2 are Pending because no node has enough CPU/memory. Add more nodes or reduce resource requests.

**Q: Can you have two containers in the same Pod on different ports?**
A: Yes, they share the same network namespace (same IP, can reach each other on localhost). Common pattern: app container + sidecar proxy.

**Q: What's the difference between Helm and Kustomize?**
A: Helm: package manager with templating (Go templates, values.yaml, dependency management). Kustomize: native YAML patching (overlays, no templating). Helm is more powerful but complex. Kustomize is simpler but limited.

### Follow-up questions

**Q: You mentioned etcd is the single source of truth. What happens if the API server goes down?**
A: Existing Pods continue running (kubelet keeps them alive). You cannot deploy changes, scale, or run `kubectl`. The cluster is "frozen." For production, run multi-instance API server behind a load balancer.

**Q: How would you design a zero-downtime deployment for your Laravel app on K8s?**
A: RollingUpdate with readiness probe. The probe checks that the app is warm (caches loaded, connections established). Set `maxSurge: 1`, `maxUnavailable: 0` (always have full capacity). PreStop hook for graceful shutdown.

**Q: What's the difference between `kubectl apply` and `kubectl create`?**
A: `kubectl create` is imperative — creates a resource, fails if it exists. `kubectl apply` is declarative — creates or updates to match the YAML. Always use `apply` in CI/CD.
