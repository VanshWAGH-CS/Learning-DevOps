# ☸️ Kubernetes Study Notes

> Complete reference for Kubernetes concepts studied today — theory, diagrams, commands, and code.

---

## Table of Contents

1. [ConfigMap](#1-configmap)
2. [Persistent Volume (PV & PVC)](#2-persistent-volume-pv--pvc)
3. [CronJobs](#3-cronjobs)
4. [Scaling & Scheduling](#4-scaling--scheduling)
5. [StatefulSets](#5-statefulsets)
6. [Limits & Resource Management](#6-limits--resource-management)
7. [Taints, Tolerations & Node Affinity](#7-taints-tolerations--node-affinity)
8. [Resource Quotas](#8-resource-quotas)
9. [Probes (Liveness, Readiness, Startup)](#9-probes)
10. [Horizontal & Vertical Pod Autoscaler](#10-horizontal--vertical-pod-autoscaler)
11. [Load Generator](#11-load-generator)
12. [Kubernetes Cluster Autoscaler](#12-kubernetes-cluster-autoscaler)

---

## 1. ConfigMap

### Theory
A **ConfigMap** decouples configuration from container images, storing non-sensitive key-value data. Pods can consume ConfigMaps as:
- Environment variables
- Command-line arguments
- Files mounted in a volume

```
┌─────────────────────────────────────────┐
│               ConfigMap                 │
│  key: APP_ENV       value: production   │
│  key: LOG_LEVEL     value: debug        │
│  key: MAX_CONN      value: 100          │
└──────────────────┬──────────────────────┘
                   │  consumed by
         ┌─────────┴──────────┐
         │        Pod         │
         │  env: APP_ENV      │
         │  env: LOG_LEVEL    │
         └────────────────────┘
```

### Create ConfigMap

```bash
# From literal values
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=debug

# From a file
kubectl create configmap app-config --from-file=config.properties

# From a directory
kubectl create configmap app-config --from-file=./config-dir/

# View ConfigMap
kubectl get configmap app-config -o yaml
kubectl describe configmap app-config
```

### YAML Definition

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  LOG_LEVEL: debug
  MAX_CONN: "100"
  config.properties: |
    server.port=8080
    db.host=localhost
```

### Use in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  containers:
  - name: myapp
    image: nginx
    # Method 1: Individual env vars
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
    # Method 2: All keys as env vars
    envFrom:
    - configMapRef:
        name: app-config
    # Method 3: Mount as volume
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config
  volumes:
  - name: config-volume
    configMap:
      name: app-config
```

---

## 2. Persistent Volume (PV & PVC)

### Theory
Kubernetes separates **storage provisioning** (admin's job) from **storage consumption** (developer's job).

```
┌──────────────────────────────────────────────────────────────┐
│                    Storage Lifecycle                         │
│                                                              │
│  Admin creates:          Developer requests:                 │
│  ┌─────────────────┐     ┌──────────────────┐               │
│  │ PersistentVolume│◄────│PersistentVolume   │               │
│  │ (PV)            │ bind│Claim (PVC)        │               │
│  │ capacity: 10Gi  │────►│ request: 5Gi      │               │
│  │ hostPath/NFS/..│     │ accessMode: RWO   │               │
│  └─────────────────┘     └────────┬─────────┘               │
│                                   │ mounted by               │
│                          ┌────────▼─────────┐               │
│                          │       Pod         │               │
│                          │ /data → PVC       │               │
│                          └──────────────────┘               │
└──────────────────────────────────────────────────────────────┘
```

### Access Modes
| Mode | Short | Description |
|------|-------|-------------|
| ReadWriteOnce | RWO | One node read/write |
| ReadOnlyMany | ROX | Many nodes read-only |
| ReadWriteMany | RWX | Many nodes read/write |

### PersistentVolume YAML

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain   # Retain | Delete | Recycle
  storageClassName: manual
  hostPath:
    path: /mnt/data   # for local/testing; use NFS/cloud in prod
```

### PersistentVolumeClaim YAML

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
```

### Mount PVC in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-storage
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - mountPath: /data
      name: storage
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc
```

### Commands

```bash
kubectl get pv                          # list persistent volumes
kubectl get pvc                         # list claims
kubectl describe pvc my-pvc             # check binding status
kubectl delete pvc my-pvc               # delete claim
```

---

## 3. CronJobs

### Theory
**CronJob** creates Jobs on a repeating schedule. A **Job** runs a Pod to completion. CronJobs are ideal for backups, reports, cleanup tasks.

```
CronJob (schedule: "*/5 * * * *")
    │
    ├──[at T+0]──► Job-1 ──► Pod ──► runs & completes ✓
    ├──[at T+5]──► Job-2 ──► Pod ──► runs & completes ✓
    └──[at T+10]─► Job-3 ──► Pod ──► running...
```

### Cron Schedule Syntax

```
┌─────────── minute (0–59)
│ ┌───────── hour (0–23)
│ │ ┌─────── day of month (1–31)
│ │ │ ┌───── month (1–12)
│ │ │ │ ┌─── day of week (0–6, Sun=0)
│ │ │ │ │
* * * * *

Examples:
  */5 * * * *    → every 5 minutes
  0 2 * * *      → daily at 2:00 AM
  0 0 * * 0      → every Sunday midnight
  0 9 1 * *      → 1st of every month at 9 AM
```

### CronJob YAML

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
spec:
  schedule: "0 2 * * *"          # daily at 2 AM
  concurrencyPolicy: Forbid       # Allow | Forbid | Replace
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  startingDeadlineSeconds: 60
  jobTemplate:
    spec:
      completions: 1
      backoffLimit: 2
      template:
        spec:
          restartPolicy: OnFailure   # Never | OnFailure
          containers:
          - name: backup
            image: busybox
            command: ["/bin/sh", "-c", "echo Running backup at $(date)"]
```

### Commands

```bash
kubectl get cronjob                      # list cronjobs
kubectl describe cronjob db-backup       # inspect
kubectl get jobs                         # see generated jobs
kubectl get pods --selector=job-name=<name>   # pods from a job

# Manually trigger a cronjob
kubectl create job --from=cronjob/db-backup manual-backup-01

# Delete cronjob
kubectl delete cronjob db-backup
```

---

## 4. Scaling & Scheduling

### Manual Scaling

```bash
# Scale a deployment
kubectl scale deployment myapp --replicas=5

# Scale via patch
kubectl patch deployment myapp -p '{"spec":{"replicas":3}}'
```

### Scheduling Concepts

```
┌─────────────────────────────────────────────┐
│             Kubernetes Scheduler            │
│                                             │
│  1. Filtering (Predicates)                  │
│     - Sufficient CPU/Memory?                │
│     - Node selectors match?                 │
│     - Taints tolerated?                     │
│     - Port conflicts?                       │
│                                             │
│  2. Scoring (Priorities)                    │
│     - Least requested resources             │
│     - Pod affinity/anti-affinity            │
│     - Image locality                        │
│                                             │
│  3. Bind → Pod assigned to best node        │
└─────────────────────────────────────────────┘
```

### nodeSelector (simple scheduling)

```yaml
spec:
  nodeSelector:
    disktype: ssd       # node must have this label
```

```bash
# Label a node
kubectl label node node01 disktype=ssd

# View node labels
kubectl get nodes --show-labels
```

---

## 5. StatefulSets

### Theory
**StatefulSet** manages Pods with stable, unique identities. Unlike Deployments, each Pod has a persistent hostname and ordered creation/deletion.

```
StatefulSet: mysql
  ├── mysql-0  (created first, always)
  ├── mysql-1  (created after mysql-0 is Running)
  └── mysql-2  (created after mysql-1 is Running)

Each gets:
  ✓ Stable DNS: mysql-0.mysql-svc.default.svc.cluster.local
  ✓ Own PVC:    data-mysql-0, data-mysql-1, data-mysql-2
  ✓ Ordered scaling: scale up 0→1→2, scale down 2→1→0
```

### Use Cases
- Databases (MySQL, PostgreSQL, Cassandra)
- Message queues (Kafka, RabbitMQ)
- Any app needing stable network identity or persistent storage per Pod

### StatefulSet YAML

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  selector:
    matchLabels:
      app: mysql
  serviceName: "mysql-svc"       # headless service name
  replicas: 3
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          value: secret
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: data
          mountPath: /var/lib/mysql
  volumeClaimTemplates:           # each Pod gets its own PVC
  - metadata:
      name: data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi
---
# Required headless service
apiVersion: v1
kind: Service
metadata:
  name: mysql-svc
spec:
  clusterIP: None    # headless
  selector:
    app: mysql
  ports:
  - port: 3306
```

### Commands

```bash
kubectl get statefulset
kubectl describe statefulset mysql
kubectl scale statefulset mysql --replicas=5

# Pods are named predictably
kubectl get pods -l app=mysql
# → mysql-0, mysql-1, mysql-2

# Delete (pods deleted in reverse order: 2→1→0)
kubectl delete statefulset mysql --cascade=orphan   # keeps PVCs
```

---

## 6. Limits & Resource Management

### Theory
- **Requests**: minimum resources guaranteed to the container
- **Limits**: maximum resources the container can use

```
Node (8 CPU, 16Gi RAM)
│
├── Pod A
│    └── Container: request=1CPU/512Mi  limit=2CPU/1Gi
│
├── Pod B
│    └── Container: request=2CPU/1Gi   limit=4CPU/2Gi
│
└── Remaining schedulable: 5CPU/~13Gi (based on requests)
```

### QoS Classes
| Class | Condition |
|-------|-----------|
| **Guaranteed** | limits == requests for all containers |
| **Burstable** | at least one container has requests < limits |
| **BestEffort** | no requests or limits set |

> Eviction order: BestEffort → Burstable → Guaranteed

### Resource Limits in Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx
    resources:
      requests:
        memory: "128Mi"
        cpu: "250m"       # 250 millicores = 0.25 CPU
      limits:
        memory: "256Mi"
        cpu: "500m"
```

### LimitRange (namespace-level defaults)

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
spec:
  limits:
  - type: Container
    default:           # applied if no limit set
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:    # applied if no request set
      cpu: "100m"
      memory: "128Mi"
    max:
      cpu: "2"
      memory: "2Gi"
    min:
      cpu: "50m"
      memory: "64Mi"
```

```bash
kubectl describe limitrange default-limits -n dev
kubectl get limitrange -n dev
```

---

## 7. Taints, Tolerations & Node Affinity

### Taints & Tolerations

**Taint** = mark on a node repelling Pods  
**Toleration** = permission on a Pod to land on tainted nodes

```
Node: node01
  Taint: gpu=true:NoSchedule
         ↑ Only Pods tolerating this can schedule here

Pod A (no toleration)  ──╳──► node01  (rejected)
Pod B (has toleration) ──────► node01  (accepted)
```

#### Taint Effects
| Effect | Behavior |
|--------|----------|
| `NoSchedule` | New pods won't schedule; existing stay |
| `PreferNoSchedule` | Soft — avoid if possible |
| `NoExecute` | New pods rejected; existing evicted |

```bash
# Add taint
kubectl taint nodes node01 gpu=true:NoSchedule

# Remove taint (note the trailing -)
kubectl taint nodes node01 gpu=true:NoSchedule-

# View taints
kubectl describe node node01 | grep Taint
```

#### Toleration in Pod

```yaml
spec:
  tolerations:
  - key: "gpu"
    operator: "Equal"    # Equal | Exists
    value: "true"
    effect: "NoSchedule"
```

---

### Node Affinity

More expressive than `nodeSelector`. Uses label-based rules.

```yaml
spec:
  affinity:
    nodeAffinity:
      # Hard rule — must match
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In         # In | NotIn | Exists | DoesNotExist | Gt | Lt
            values:
            - ssd

      # Soft rule — prefer but not required
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: region
            operator: In
            values:
            - us-east
```

#### Pod Anti-Affinity (spread pods across nodes)

```yaml
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: myapp
        topologyKey: kubernetes.io/hostname
```

---

## 8. Resource Quotas

### Theory
**ResourceQuota** limits total resource consumption per namespace.

```
Namespace: dev
┌────────────────────────────────┐
│  ResourceQuota                  │
│  cpu:       max 10 cores        │
│  memory:    max 20Gi            │
│  pods:      max 20              │
│  services:  max 10              │
└────────────────────────────────┘
  All Pods in "dev" share this budget
```

### ResourceQuota YAML

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
    persistentvolumeclaims: "5"
    configmaps: "10"
    secrets: "10"
```

```bash
kubectl create namespace dev
kubectl apply -f quota.yaml -n dev
kubectl describe resourcequota dev-quota -n dev
kubectl get resourcequota -n dev
```

---

## 9. Probes

### Theory
Probes are health checks Kubernetes runs against containers.

```
Pod Lifecycle with Probes:
─────────────────────────────────────────────────────────►  time

[Container Starts]
      │
      ▼
  startupProbe ──── keeps checking until app is ready ────►
      │ (success)
      ▼
  readinessProbe ── is app ready to receive traffic? ──────►
  livenessProbe ─── is app still alive? ───────────────────►
```

### 9a. Liveness Probe
> Answers: **"Is the container alive?"**  
> On failure: container is **restarted**

```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 10    # wait before first check
  periodSeconds: 5           # check every 5s
  failureThreshold: 3        # restart after 3 failures
  timeoutSeconds: 2
```

### 9b. Readiness Probe
> Answers: **"Is the container ready to serve traffic?"**  
> On failure: Pod removed from Service endpoints (no restart)

```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  successThreshold: 1        # passes needed to mark ready
  failureThreshold: 3
```

### 9c. Startup Probe
> Answers: **"Has the app started yet?"** (for slow-starting apps)  
> Disables liveness & readiness until startup succeeds

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30       # 30 * 10s = 5 min max startup time
  periodSeconds: 10
```

### Probe Types Comparison

| Probe | Trigger | On Failure | Use Case |
|-------|---------|------------|----------|
| **Liveness** | Always running | Restart container | Detect deadlocks |
| **Readiness** | Always running | Remove from LB | DB not ready yet |
| **Startup** | Until success | Restart container | Slow JVM startup |

### Probe Methods

```yaml
# HTTP GET
httpGet:
  path: /health
  port: 8080
  httpHeaders:
  - name: Authorization
    value: Bearer token123

# TCP Socket
tcpSocket:
  port: 3306

# Exec command (exit 0 = healthy)
exec:
  command:
  - cat
  - /tmp/healthy
```

### Full Example with All Three Probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: app
    image: myapp:latest
    ports:
    - containerPort: 8080
    startupProbe:
      httpGet:
        path: /healthz
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 5
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /ready
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 5
      failureThreshold: 3
```

---

## 10. Horizontal & Vertical Pod Autoscaler

### Horizontal Pod Autoscaler (HPA)
> Scales the **number of Pod replicas** based on metrics (CPU, memory, custom)

```
         CPU > 70%?
              │
     ┌────────▼────────┐
     │       HPA        │
     │  current: 2 pods │
     │  desired: 4 pods │
     └────────┬────────┘
              │ scales
     ┌────────▼────────────────┐
     │ Deployment (4 replicas)  │
     │  Pod1 Pod2 Pod3 Pod4     │
     └──────────────────────────┘
```

#### Enable Metrics Server (required for HPA)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl top nodes
kubectl top pods
```

#### Create HPA

```bash
# Imperative
kubectl autoscale deployment myapp \
  --cpu-percent=70 \
  --min=2 \
  --max=10

# View HPA
kubectl get hpa
kubectl describe hpa myapp
```

#### HPA YAML

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: AverageValue
        averageValue: 500Mi
```

---

### Vertical Pod Autoscaler (VPA)
> Adjusts **CPU/memory requests & limits** of existing Pods (not replica count)

```
VPA observes usage over time:
  Container using 800m CPU but only requested 100m
            ↓
  VPA recommends / sets: request=800m, limit=1.2
            ↓
  Pod restarted with new resource config
```

#### VPA Modes
| Mode | Behavior |
|------|----------|
| `Off` | Only provides recommendations |
| `Initial` | Sets resources only at Pod creation |
| `Auto` | Continuously updates (restarts Pods) |
| `Recreate` | Updates by evicting and recreating Pods |

#### VPA YAML

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"    # Off | Initial | Recreate | Auto
  resourcePolicy:
    containerPolicies:
    - containerName: "*"
      minAllowed:
        cpu: 100m
        memory: 128Mi
      maxAllowed:
        cpu: 4
        memory: 4Gi
```

```bash
# Get VPA recommendations
kubectl describe vpa myapp-vpa

# Check recommendations section in output:
# Recommendation:
#   Container Recommendations:
#     Container Name: myapp
#       Lower Bound:  cpu: 100m, memory: 128Mi
#       Target:       cpu: 500m, memory: 512Mi
#       Upper Bound:  cpu: 2,    memory: 2Gi
```

#### HPA vs VPA

| | HPA | VPA |
|--|-----|-----|
| Scales | Replicas (horizontal) | Resources (vertical) |
| Restart needed | No | Yes (in Auto mode) |
| Best for | Stateless apps | Resource-intensive single pods |
| Use together? | Avoid on same metric; use HPA for CPU+VPA for memory |

---

## 11. Load Generator

### Theory
A load generator artificially increases traffic/CPU to test autoscaling behavior.

### Simple Load Generator Pod

```bash
# Start load against a service
kubectl run load-gen \
  --image=busybox \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://myapp-svc; done"

# Watch HPA respond
kubectl get hpa --watch

# Watch pods scale
kubectl get pods --watch

# Stop load generator
kubectl delete pod load-gen
```

### Apache Bench Load Test

```bash
kubectl run ab-load \
  --image=jordi/ab \
  --restart=Never \
  -- ab -n 100000 -c 100 http://myapp-svc/
```

### Locust Load Generator (YAML)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: locust
spec:
  replicas: 1
  selector:
    matchLabels:
      app: locust
  template:
    metadata:
      labels:
        app: locust
    spec:
      containers:
      - name: locust
        image: locustio/locust
        args: ["-f", "/locust/locustfile.py", "--host=http://myapp-svc"]
        ports:
        - containerPort: 8089
```

### Full HPA + Load Test Workflow

```bash
# 1. Deploy app with resource requests
kubectl apply -f deployment.yaml

# 2. Create HPA
kubectl autoscale deployment myapp --cpu-percent=50 --min=1 --max=10

# 3. Generate load
kubectl run load --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://myapp-svc; done"

# 4. Watch in separate terminal
watch kubectl get hpa,pods

# 5. Stop load and watch scale-down
kubectl delete pod load
```

---

## 12. Kubernetes Cluster Autoscaler

### Theory
The **Cluster Autoscaler (CA)** automatically adjusts the number of **nodes** in a cluster when:
- Pods can't be scheduled (scale up — add node)
- Nodes are underutilized (scale down — remove node)

```
┌─────────────────────────────────────────────────────────┐
│                    Cluster Autoscaler                   │
│                                                         │
│  Unschedulable Pod detected?                            │
│       YES → Request new node from cloud provider        │
│              Node joins cluster                         │
│              Pod scheduled ✓                            
│                                                         │
│  Node utilization < 50% for 10 mins?                    │
│       YES → Evict pods (respecting PodDisruptionBudget) │
│              Remove node from cluster                   │
│              Pods rescheduled on other nodes ✓          
└─────────────────────────────────────────────────────────┘
```

### CA vs HPA vs VPA

```
Traffic Spike
     │
     ▼
HPA detects CPU > threshold
     │
     ▼
Scales Pod replicas → but no node capacity!
     │
     ▼
Cluster Autoscaler detects Pending Pods
     │
     ▼
Adds new node → Pods scheduled → CPU normalizes
```

### Installation (AWS EKS example)

```bash
# Deploy Cluster Autoscaler
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# Annotate the deployment
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler \
  cluster-autoscaler.kubernetes.io/safe-to-evict="false"

# Patch with cluster name
kubectl -n kube-system patch deployment cluster-autoscaler \
  --type=json \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/command/-","value":"--node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster"}]'
```

### Node Group Annotations (AWS ASG tags)

```
k8s.io/cluster-autoscaler/enabled = "true"
k8s.io/cluster-autoscaler/<cluster-name> = "owned"
```

### CA Key Flags

```yaml
- --scale-down-enabled=true
- --scale-down-utilization-threshold=0.5   # scale down if < 50% used
- --scale-down-delay-after-add=10m         # wait 10m after adding node
- --scale-down-unneeded-time=10m           # node must be unneeded 10m
- --max-graceful-termination-sec=600
```

### Useful Commands

```bash
# View CA logs
kubectl -n kube-system logs -f deployment/cluster-autoscaler

# Check pending pods (CA responds to these)
kubectl get pods --all-namespaces --field-selector=status.phase=Pending

# View node capacity
kubectl get nodes
kubectl describe node <node-name>
kubectl top nodes

# View CA status
kubectl -n kube-system get configmap cluster-autoscaler-status -o yaml
```

### PodDisruptionBudget (works with CA)

```yaml
# Ensure CA doesn't evict too many pods at once
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2        # or use maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

---

## Quick Reference Cheatsheet

```bash
# ── ConfigMap ──
kubectl create configmap <name> --from-literal=key=val
kubectl get configmap / kubectl describe configmap <name>

# ── PV / PVC ──
kubectl get pv,pvc
kubectl describe pvc <name>

# ── CronJob ──
kubectl get cronjob / kubectl get jobs
kubectl create job --from=cronjob/<name> <job-name>

# ── Scaling ──
kubectl scale deployment <name> --replicas=N

# ── StatefulSet ──
kubectl get statefulset
kubectl scale statefulset <name> --replicas=N

# ── Resources ──
kubectl top pods / kubectl top nodes
kubectl describe limitrange / kubectl describe resourcequota

# ── Taints ──
kubectl taint nodes <node> key=val:Effect
kubectl taint nodes <node> key=val:Effect-         # remove

# ── HPA ──
kubectl autoscale deployment <name> --cpu-percent=70 --min=2 --max=10
kubectl get hpa --watch

# ── Probes ──
kubectl describe pod <name> | grep -A 10 Liveness
kubectl describe pod <name> | grep -A 10 Readiness

# ── Cluster Autoscaler ──
kubectl -n kube-system logs -f deployment/cluster-autoscaler
kubectl get pods --field-selector=status.phase=Pending -A
```

---

*Notes compiled: Kubernetes study session | Topics: ConfigMap, PV/PVC, CronJobs, Scaling, StatefulSets, Limits, Taints/Tolerations, Node Affinity, Resource Quotas, Probes, HPA, VPA, Load Generator, Cluster Autoscaler*
