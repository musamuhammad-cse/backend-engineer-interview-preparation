# Docker, Kubernetes & Terraform — Deep Dive Interview Preparation

> **Target:** Senior Backend Engineer at product-based / FAANG-type companies  
> **Your anchors:** Multi-tenant SaaS (Dockerized Laravel app deployed on ECS Fargate), trading platform (containerized microservices), Chronos (Go distributed scheduler containerized on ECS), CI/CD with GitHub Actions deploying Docker images to ECR → ECS  
> **Note:** Containers + IaC are expected knowledge for senior roles. The senior signal is **designing containerized systems for production** (health checks, graceful shutdown, resource limits, sidecar pattern, immutable infrastructure with Terraform).

---

## How to use this material

| Step | Action | Time |
|------|--------|------|
| 1 | Read a section, close the file, explain it out loud | 20 min/section |
| 2 | Type out the Dockerfiles/Terraform configs from memory | 15 min/section |
| 3 | Answer the section's Q&A without looking, then diff | 20 min/section |
| 4 | Design the architecture on paper (deployment diagram) | 15 min |

**The senior signal is container orchestration patterns and Infrastructure as Code mindset.** Knowing `docker run` flags is table stakes; designing multi-service deployments with health checks, resource limits, zero-downtime rollouts, and Terraform module composability is the differentiator.

---

## Files

| File | Contents | Approx. study time |
|------|----------|--------------------|
| [`01-basic.md`](./01-basic.md) | Docker: images vs containers, Dockerfile (FROM, RUN, CMD, ENTRYPOINT, COPY, EXPOSE, multi-stage builds), docker-compose, networking (bridge, host, none, overlay), volumes (bind mounts, named volumes, tmpfs), Docker registry (ECR, Docker Hub), docker build/run/exec/logs/ps, .dockerignore | 4–6 hours |
| [`02-intermediate.md`](./02-intermediate.md) | K8s: pods, deployments, replica sets, services (ClusterIP, NodePort, LoadBalancer), ConfigMaps, Secrets, Ingress, persistent volumes/claims, namespaces, probes (liveness, readiness, startup), resource quotas/limits, `kubectl` essentials | 8–10 hours |
| [`03-senior.md`](./03-senior.md) | K8s production patterns (sidecar, ambassador, init containers), Helm, service mesh (Istio, Linkerd), horizontal pod autoscaler, pod disruption budgets, network policies, RBAC, Terraform (state management, remote backend, modules, workspaces, `terraform plan/apply/destroy`, Terraform Cloud), Pulumi comparison, production EKS with IaC | 10–12 hours |
| [`04-question-bank.md`](./04-question-bank.md) | 150+ interview questions, debugging scenarios, architecture prompts, Terraform code puzzles | Ongoing drill |

---

## Coverage map

### Docker fundamentals
- Containers vs VMs (process isolation, cgroups, namespaces)
- Images: layers, union filesystem (overlay2), image cache
- Dockerfile instructions: FROM, RUN, CMD, ENTRYPOINT, COPY, ADD, WORKDIR, EXPOSE, ENV, ARG, USER, HEALTHCHECK, ONBUILD, SHELL, LABEL
- Multi-stage builds (minimize image size)
- Docker compose: services, networks, volumes, depends_on, healthcheck, env_file, profiles
- Docker networking: bridge (default), host, none, overlay (Swarm)
- Volumes: bind mounts, named volumes (managed by Docker), tmpfs
- Registry: Docker Hub, ECR, GCR, private registry
- Common commands: build, run, exec, logs, ps, images, rm, rmi, prune, inspect, stats, system df

### Kubernetes fundamentals
- Architecture: control plane (API server, etcd, scheduler, controller manager) + worker nodes (kubelet, kube-proxy, container runtime)
- Pods: atomic unit, containers in a pod share network/IP/storage
- Workloads: Deployment (stateless), StatefulSet (stateful), DaemonSet (per-node), Job/CronJob (batch)
- Services: ClusterIP (internal), NodePort (external on port), LoadBalancer (cloud LB)
- Ingress: path/host-based routing, TLS termination
- ConfigMaps & Secrets: environment variables, volumes
- Storage: PV/PVC, StorageClass, CSI drivers
- Probes: liveness (restart), readiness (traffic), startup (slow apps)

### Kubernetes production patterns
- Sidecar: logging agent (Fluentd), service mesh proxy (Envoy), config reloader
- Ambassador: proxy traffic to external services
- Adapter: normalize metrics (Prometheus adapter)
- Init containers: wait for dependencies, DB migrations
- Deployments: rolling update (maxUnavailable, maxSurge), blue-green, canary
- HPA: metrics-based auto-scaling (CPU, memory, custom metrics, external metrics)
- PDB: ensure minimum available pods during voluntary disruption
- NetworkPolicies: pod-level firewall (allow/deny ingress/egress)
- Resource management: requests (scheduling) vs limits (throttling/oom)
- Namespaces: isolation, resource quotas

### Terraform / IaC
- Core concepts: desired state, execution plan, resource graph, idempotence
- HCL syntax: blocks, arguments, expressions, functions, meta-arguments (count, for_each, depends_on, provider)
- State: local backend, remote backends (S3 + DynamoDB locking, Terraform Cloud)
- Modules: input variables, output values, module registry, composition
- Workspaces: managing multiple environments (dev/staging/prod)
- Provisioners: file, remote-exec, local-exec (last resort)
- `terraform init/plan/apply/destroy/fmt/validate/state/import`
- Remote backends: S3 (state) + DynamoDB (locking), Terraform Cloud, Terraform Enterprise
- CI/CD with Terraform: GitHub Actions, Atlantis, Terraform Cloud runs

### Docker vs container runtimes
- containerd (industry standard, used by Docker and K8s)
- CRI-O (Red Hat's K8s runtime)
- runc (low-level OCI runtime)
- gVisor, Kata Containers (sandboxed containers)

### K8s vs alternatives
- ECS (AWS native, simpler) vs EKS (K8s, portable, ecosystem)
- Nomad (HashiCorp, simpler scheduler)
- Docker Swarm (deprecated, avoid)

### Terraform vs alternatives
- Pulumi (general-purpose languages instead of HCL)
- AWS CDK (TypeScript/Python for AWS)
- CloudFormation (AWS native, YAML/JSON)
- Ansible (config management, not pure IaC)

---

## Study order recommendation

Focus on Docker first (your daily tool), then K8s patterns (senior signal), then Terraform for IaC.

```
Week 1:  01-basic.md          + Write Dockerfiles from scratch (Laravel + Go)
Week 2:  02-intermediate.md   + Deploy a simple app with Deployments/Services/Ingress
Week 3:  03-senior.md         + Terraform EKS module, Helm charts, HPA
Week 4+: 04-question-bank.md daily drill
```

**Next topic in skill order:** CI/CD.
