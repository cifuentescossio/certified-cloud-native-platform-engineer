# certified-cloud-native-platform-engineer

The industry shift, every few years the industry redefines how infrastructure teams operate

Sysadmin > DevOps > Platform Engineering

## Certification Overview

* Format: Performance-Based (hands-on tasks in a live environment)
* Questions: 15-20 tasks
* Duration 2 hours
* Language: English only
* Domains: 
    * Platform Architecture and Infrastructure (15%)
    * GitOps and Continouos Delivery (25%) --> Argo, Tekton, Flagger, Flux
    * Platform APIs and Self-Service Capabilities (25%) --> Crossplane, CRDs, Kubernetes Operators
    * Observability and Operations (20%) --> Prometheus, Grafana, Jaeger, OpenTelemetry, OpenCost
    * Security and Policy Enforcement (15%) --> Gatekeeper, OPA, Kyverno, Istio, Linkerd

## Platform Architecture and Infrastructure

### Flexibility vs Structure
* To be able to scale we need to stop building cluster and build platform instead

### Chaos at Scale
* Inconsistent Configuration
* No Boundaries
* Security Gaps

### Blueprint / Intentional Architecture

* In multi-cluster environments, a blueprint is a standardized, pre-configured template (typically written in Terraform, AWS CDK, or Pulumi) used to quickly and consistently provision fully functional Kubernetes clusters.

* Kubernetes is the infrastructure, the platform is what we build on top.

* Layered Platform Architecture:
    * Infrastructure
        * Nodes, Cloud, APIs, Hardware
        * Owner: Infrastructure / Cloud Team
    * Platform Core
        * Kubernetes itself (Networking, Storage, Access Control)
        * Owner: Platform Team
    * Platform Services
        * Observability, GitOps, Security
        * Owner: Platform Team
    * Application Layer
        * Business Workloads, Microservices, Jobs
        * Owner: Application / Dev Team
        * Interaction via golden paths and templates (no K8S YAML)

### Control Plane VS Data Plane

* Control Plane:
    * API Server
    * ETCD (state storage)
    * Scheduler
    * Controller Manager

* Data Plane
    * Worker Nodes
    * Kubelet
    * Container Runtime
    * Pods/Workloads

* Control Plane and Data Plane fail independently

### Cluster Topology: One vs Many

* Single:
    * [good] Simpler operations
    * [good] Better Resource Utilization
    * [good] Single Control Plane
    * [bad] Blast Radius is entire org
    * [bad] Control plane/etcd failure impacts all teams
    * [bad] Noisy neighbor teams

* Multi:
    * [good] Strong Isolation
    * [good] Independent lifecycle
    * [bad] Higher Operational Cost
    * [bad] Cross-Cluster Complexity

* Common Patterns:
    * By Environment
    * By Region
    * By Team/BU
    * Hybrid

### Right Sizing

* Kubernetes needs explicit resources hints:
    * Requests
        * Guaranteed minimum resources
        * Used by: Scheduler for placement
        * Meaning: "I need at least this much"
        * If Exceeded: Pod can use more (if available)
        * Without requests: Scheduler cannot make intelligent placement decisions
    * Limits
        * Maximum allowed resources
        * Used by: Kubelet for enforcement
        * Meaning: "Never use more than this"
        * If Exceeded: CPU throttled, Memory OOMKilled
        * Without limits: one runaway pod can crash an entire node

* To size properly:
    * Set sensible default CPU/memory requests
    * Maximum resource limits per namespace
    * Set memory limits
    * Right-sizing based on real usage (Prometheus, kubectl)
        * Set Request at P95
        * Set Limits at 2-3x
        * VPA (Vertical Pod Autoscaler) can recommend values based on observed usage

* Quality of Service (QoS) Classes
    * Guaranteed:
        * When requests and limits are equal (requests == limits)
        * Last to be evicted
        * Most Predictable Performance
    * Burstable:
        * When requests are less than limits (requests < limits>)
        * Evicted after BestEffort
        * Can burst when resources are available
    * BestEffort:
        * No request or limits set (requests  == none || limits == 0)
        * First to be evicted
        * No guarantees whatsoever

### Multi-Tenancy

>Isolation cost money, sharing requires discipline

* K8S is NOT Multi-Tenant by Default:
    * Communication: everything can communicate with everything
    * No Resource Limits
    * No RBAC configuration

* Shared infrastructure for separate teams - Failures Modes:
    * Secrets Exposure
    * Resource Exhaustion
    * Accidental Deletion
    * Network Segmentation

* Multi-Tenancy Models
    * Soft Multi-Tenancy: Isolation via Namespaces + RBAC + NetPol
    * Hard Multi-Tenancy: Separate clusters or virtual clusters
    
* Isolation levels:
    1. Namespace
    2. NS + RBAC + Quotas
    3. NS + RBAC + Quotas + NetPol
    4. Virtual Cluster
    5. Separate Cluster

* The Guardrails Stack
    1. Namespaces: Logical Isolation
        * Patterns:
            * By Team
            * By Environment
            * By App + Environment
    2. RBAC: Access Control
    3. ResourceQuotas: Fair resource allocation
    4. NetworkPolicies: Network Segmentation
    5. Pod Security Standards: Workload Hardening

### Resource Governance

* ResourceQuota: Namespace Budget
    * Enforced at admission time by the API Server (before schedule)
    * types:
        * Compute (cpu, memory)
        * Object Count (pods, services, configmaps, secrets)
        * Storage (volumeclaims, storageclass)

``` yaml
apiVersion: v1
kind: ReourceQuota
metadata:
    name: rq-example
spec:
    hard:
        requests.cpu: "32"
        requests.memory: "64Gi"
        limits.cpu: "64"
        limits.memory: "128Gi"
        pods: "10"
```

> if RequestQuota is set every pod must to have requests and limits

* LimitRange: Individual Pod/Container Budget
    * Configuration Options:
        * default: set default **limits**
        * defaultRequest: set default **requests**
        * min/max: defines min and max per container to enforce hard boundaries and prevent tiny ir oversized containers
    * Objects:
        * Container
        * Pod
        * PersistenceVolumeClaim

``` yaml
apiVersion: v1
kind: LimitRange
metadata:
    name: limitrange-example
spec:
    limits:
        - type: container
        min:
            cpu: 50m
            memory: 64Mi
        max: 
            cpu: "4"
            memory: 8Gi
        defaultRequest:
            cpu: 100m
            memory: 128Mi
        default:
            cpu: 500m
            memory: 512Mi 
```

### Persistent Storage

* Storage Stack
    * StorageClass: provisioning Template
    * PersistentVolume (PV): Actual Storage Resource
    * PersistentVolumeClaim (PVC): Request for Storage
    * Pod (mounts): regular directory

* Access Modes
    * ReadWriteOnce (RWO)
        * only pods in the specific node can use it
    * ReadOnlyMany (ROX)
        * Many nodes can mount read-only
    * ReadWriteMany (RWX)
        * Many nodes can mount read-write

* PersistentVolumeClaim

``` yaml
# PersistentVolumeClaim

apiVersion: v1
kind: PersistentVolumeClaim
metadata: 
    name: postgress-data
spec:
    accessModes:
        - ReadWriteOnce
    resources:
        requests:
            storage: 10Gi
    storageClassName: fast-ssd
```
``` yaml
# StorageClass

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
    name: fast-ssd
provisioner: ebs.csi.aws.com # CSI driver that creates the actual storage
parameters: # Provider-specific settings
    type: gp3
    iops: "3000"
reclaimPolicy: Delete # could be "Retain"
volumeBindingMode: WaitForFirstConsumer # could be "Immediate"

```

## GitOps and Continouos Delivery

## Platform APIs and Self-Service Capabilities

## Observability and Operations

## Security and Policy Enforcement

