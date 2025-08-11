# Kubernetes Learning Guide
*A comprehensive curriculum for understanding Kubernetes from basics to advanced concepts*

---

## Chapter 1: Introduction to Kubernetes
*Start here to understand what Kubernetes is and why it matters*

### What is Kubernetes?
- **Container Orchestration Platform**
  - Automates deployment, scaling, and management of containerized applications
  - Originally developed by Google, now maintained by CNCF
  - Also known as "K8s" (8 letters between K and s)

### Why Use Kubernetes?
- **Scalability**: Automatically scale applications up/down based on demand
- **High Availability**: Ensures applications stay running even when servers fail
- **Portability**: Run anywhere - on-premises, cloud, hybrid environments
- **Resource Efficiency**: Optimal resource utilization across your infrastructure

### Key Benefits
- Self-healing capabilities
- Automated rollouts and rollbacks
- Service discovery and load balancing
- Storage orchestration
- Secret and configuration management

---

## Chapter 2: Core Architecture
*Understanding how Kubernetes is structured*

### Cluster Overview
*A Kubernetes cluster consists of a control plane and worker nodes*

### Control Plane Components
*The brain of the cluster that makes all the decisions*

#### API Server
- **Central communication hub** for all cluster operations
- Validates and processes REST API requests
- Authentication and authorization gateway
- Single source of truth for cluster state

#### etcd
- **Distributed key-value store** that holds all cluster data
- Stores configuration, state, and metadata
- Provides strong consistency and reliability
- Backup of etcd = backup of entire cluster

#### Scheduler
- **Assigns pods to nodes** based on resource requirements, scheduling rules, and priorities.
- Considers factors such as:
  - Resource availability (CPU, memory)
  - Hardware/software constraints
  - **Priority**: Pods can be assigned a `priorityClassName` which determines their scheduling priority. Higher-priority pods are scheduled before lower-priority ones, and in cases of resource pressure, lower-priority pods may be preempted (evicted) to make room for higher-priority pods.
    - **Example**:
      ```yaml
      apiVersion: scheduling.k8s.io/v1
      kind: PriorityClass
      metadata:
        name: high-priority
      value: 100000
      globalDefault: false
      description: "This priority class is for critical workloads."
      ---
      apiVersion: v1
      kind: Pod
      metadata:
        name: important-app
      spec:
        priorityClassName: high-priority
        containers:
        - name: app
          image: myimage
      ```
    - **Preemption**: If the cluster is full, the scheduler may evict lower-priority pods to make space for higher-priority ones.
  - **nodeSelector**: A simple way to constrain pods to run only on nodes with specific labels. For example, you can use `nodeSelector: disktype: ssd` in your pod spec to ensure the pod only runs on nodes labeled with `disktype=ssd`.
  - **Affinity and Anti-Affinity rules**:  
    - **Node Affinity**: Lets you specify rules for scheduling pods onto nodes with certain labels, using `requiredDuringSchedulingIgnoredDuringExecution` or `preferredDuringSchedulingIgnoredDuringExecution`.
    - **Pod Affinity**: Ensures pods are scheduled on the same node (or zone/rack) as other specified pods.  
    - **Pod Anti-Affinity**: Ensures pods are *not* scheduled on the same node (or zone/rack) as other specified pods.
    - **Example**:
      ```yaml
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchLabels:
                  app: my-app
              topologyKey: "kubernetes.io/hostname"
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - my-app
            topologyKey: "kubernetes.io/hostname"
      ```
    - **Scheduler's Role**: The scheduler considers priority, nodeSelector, affinity/anti-affinity, taints/tolerations, resource requirements, and policies to determine the best node for each pod.
  - **Data locality**: Placing pods close to the data they need (e.g., on nodes with attached storage).
- **Node Management Operations**:
  - **cordon**: Mark a node as unschedulable so that no new pods will be scheduled on it, but existing pods are not affected. Useful for maintenance.
  - **drain**: Safely evict all pods from a node (except DaemonSets and mirror pods) and mark it unschedulable. Used before node maintenance or removal to ensure workloads are gracefully moved elsewhere.

#### Controller Manager
- **Runs controller loops** that watch and respond to cluster state
- Node Controller: Monitors node health
- Replication Controller: Maintains desired pod replicas
- Endpoints Controller: Manages service endpoints
- Service Account Controller: Creates default service accounts

### Worker Node Components
*Where your applications actually run*

#### kubelet
- **Node agent** that communicates with the control plane
- Manages pod lifecycle on the node
- Reports node and pod status back to API server
- Executes container health checks

#### kube-proxy
- **Network proxy** that maintains network rules
- Implements Services abstraction
- Handles load balancing for Services
- Manages iptables/IPVS rules

#### Container Runtime
- **Runs containers** (Docker, containerd, CRI-O)
- Pulls container images
- Starts, stops, and manages container lifecycle
- Interface through Container Runtime Interface (CRI)

---

## Chapter 3: Fundamental Building Blocks
*The essential objects you need to understand first*

### Pods
*The smallest and most basic deployable object in Kubernetes*

#### What is a Pod?
- **Group of one or more containers** that share:
  - Network namespace (same IP address)
  - Storage volumes
  - Lifecycle (created/destroyed together)

#### Pod Characteristics
- **Ephemeral**: Pods come and go, don't rely on pod IPs
- **Single responsibility**: Usually one main container per pod
- **Shared context**: Containers can communicate via localhost

#### Pod Lifecycle States
- **Pending**: Pod accepted but not yet running
- **Running**: At least one container is running
- **Succeeded**: All containers terminated successfully
- **Failed**: At least one container failed
- **Unknown**: Pod state cannot be determined

#### Common Pod Patterns
- **Sidecar**: Helper container alongside main container
- **Init Containers**: Run before main containers start
- **Multi-container**: Tightly coupled containers working together

---

## Chapter 4: Workload Management
*How to deploy and manage applications at scale*

### Deployments
*The standard way to deploy stateless applications*

#### Purpose
- **Manages ReplicaSets** and provides declarative updates
- **Rolling updates** with zero downtime
- **Rollback capability** to previous versions
- **Scaling** applications up and down

#### Key Features
- Replica management (desired vs actual state)
- Update strategies (RollingUpdate, Recreate)
- Revision history for rollbacks
- Pausing and resuming deployments

### ReplicaSets
*Ensures a specified number of pod replicas are running*

#### Responsibilities
- **Maintains pod count** according to desired state
- **Creates new pods** when some are deleted
- **Uses selectors** to identify which pods to manage
- Usually managed by Deployments, not directly

### StatefulSets
*For applications that need persistent identity*

#### Use Cases
- **Databases** (MySQL, PostgreSQL, MongoDB)
- **Distributed systems** (Kafka, Elasticsearch)
- **Applications requiring**:
  - Stable network identities
  - Persistent storage
  - Ordered deployment and scaling

#### Features
- Pods get persistent identifiers (app-0, app-1, app-2)
- Stable network identity (predictable hostnames)
- Ordered deployment and termination

### DaemonSets
*Ensures a pod runs on every (or selected) node*

#### Common Use Cases
- **Log collectors** (Fluentd, Filebeat)
- **Monitoring agents** (Node Exporter, Datadog agent)
- **Network plugins** (CNI agents)
- **Storage plugins** (CSI drivers)

### Jobs and CronJobs
*For batch processing and scheduled tasks*

#### Jobs
- **Run pods to completion** (not continuously)
- Ensure specified number of successful completions
- Handle pod failures and retries
- Examples: Data migration, backup tasks

#### CronJobs
- **Schedule Jobs** using cron syntax
- Run recurring tasks at specified times
- Examples: Daily backups, weekly reports

---

## Chapter 5: Networking Fundamentals
*How pods communicate with each other and the outside world*

### Kubernetes Networking Model
- **Every pod gets its own IP address**
- **Pods can communicate without NAT**
- **Nodes can communicate with all pods**
- **Container Network Interface (CNI)** provides networking

### Services
*Provide stable network endpoints for pods*

#### Why Services?
- **Pod IPs are ephemeral** (change when pods restart)
- **Load balancing** across multiple pod replicas
- **Service discovery** through DNS names
- **Abstraction layer** over pods

#### Service Types

##### ClusterIP (Default)
- **Internal cluster communication only**
- Accessible only from within the cluster
- Gets a virtual IP address
- Most common type for microservices

##### NodePort
- **Exposes service on each node's IP**
- Accessible from outside the cluster
- Port range: 30000-32767
- Not recommended for production

##### LoadBalancer
- **Cloud provider integration**
- Creates external load balancer
- Assigns external IP address
- Works with AWS ELB, GCP Load Balancer, Azure Load Balancer

##### ExternalName
- **Maps service to external DNS name**
- No load balancing or proxying
- Returns CNAME record

### Ingress
*HTTP/HTTPS load balancer for services*

#### Purpose
- **Single entry point** for multiple services
- **Host-based routing** (api.example.com, app.example.com)
- **Path-based routing** (/api/, /app/)
- **SSL/TLS termination**
- **More cost-effective** than multiple LoadBalancers

#### Ingress Controllers
- **NGINX Ingress Controller**: Most popular
- **Traefik**: Dynamic configuration
- **HAProxy**: High performance
- **Cloud-specific**: AWS ALB, GCP Ingress

### Network Policies
*Firewall rules for pod-to-pod communication*

#### Default Behavior
- **All pods can communicate** with each other by default
- **No network isolation** between namespaces
- Network Policies provide security through isolation

#### Policy Types
- **Ingress**: Controls incoming traffic to pods
- **Egress**: Controls outgoing traffic from pods
- **Both**: Can specify both ingress and egress rules

### CNI Plugins
*Provide networking capabilities to Kubernetes*

#### Popular CNI Plugins
- **Flannel**: Simple overlay network
- **Calico**: Network policies + networking
- **Weave Net**: Easy setup, DNS-based discovery
- **Cilium**: eBPF-based, advanced security features

---

## Chapter 6: Storage Management
*How to handle data persistence in Kubernetes*

### Understanding Kubernetes Storage
- **Containers are stateless by default**
- **Data disappears when container restarts**
- **Volumes provide persistent storage**

### Volume Types

#### Ephemeral Volumes
- **emptyDir**: Temporary storage, deleted with pod
- **configMap**: Mount configuration files
- **secret**: Mount sensitive data

#### Persistent Volumes
- **hostPath**: Mount directory from node (for testing only)
- **Cloud storage**: AWS EBS, GCP Persistent Disk, Azure Disk
- **Network storage**: NFS, iSCSI, Ceph

### Persistent Volume (PV)
*Cluster-level storage resource*

#### Characteristics
- **Independent of pod lifecycle**
- **Provisioned by admin or dynamically**
- **Has its own lifecycle**
- **Can be reused by different pods**

#### Access Modes
- **ReadWriteOnce (RWO)**: Single node read-write
- **ReadOnlyMany (ROX)**: Multiple nodes read-only
- **ReadWriteMany (RWX)**: Multiple nodes read-write

### Persistent Volume Claim (PVC)
*User's request for storage*

#### How it Works
1. **User creates PVC** specifying size and access mode
2. **Kubernetes finds matching PV** or creates new one
3. **PVC binds to PV**
4. **Pod uses PVC** in volume specification

### Storage Classes
*Templates for dynamic storage provisioning*

#### Benefits
- **Automatic PV creation** when PVC is created
- **Different storage tiers** (fast SSD, slow HDD)
- **Cloud provider integration**
- **Custom parameters** for each storage type

#### Common Parameters
- Storage type (SSD, HDD)
- Replication factor
- IOPS (Input/Output Operations Per Second)
- Encryption settings

---

## Chapter 7: Configuration Management
*How to manage application configuration and secrets*

### The Configuration Challenge
- **Applications need configuration** (database URLs, API keys)
- **Hard-coding is bad practice**
- **Need different configs** for dev/staging/production
- **Secrets require special handling**

### ConfigMaps
*Store non-sensitive configuration data*

#### Use Cases
- **Application configuration files**
- **Environment variables**
- **Command-line arguments**
- **Any non-sensitive key-value data**

#### How to Use ConfigMaps
1. **Create ConfigMap** with configuration data
2. **Reference in pod** as:
   - Environment variables
   - Volume mounts (files)
   - Command arguments

### Secrets
*Store sensitive information securely*

#### What Goes in Secrets
- **Passwords and API keys**
- **TLS certificates**
- **Docker registry credentials**
- **SSH keys**

#### Secret Types
- **Opaque**: Generic secret (default)
- **kubernetes.io/dockerconfigjson**: Docker registry credentials
- **kubernetes.io/tls**: TLS certificates
- **kubernetes.io/service-account-token**: Service account tokens

#### Security Considerations
- **Base64 encoded** (not encrypted by default)
- **Enable encryption at rest** in etcd
- **Use RBAC** to control access
- **Consider external secret management** (Vault, AWS Secrets Manager)

---

## Chapter 8: Security Essentials
*Protecting your cluster and applications*

### Security Layers
1. **Cluster Security**: Secure the infrastructure
2. **Network Security**: Control traffic flow
3. **Pod Security**: Secure individual workloads
4. **Application Security**: Secure the code and data

### Role-Based Access Control (RBAC)
*Control who can access what resources*

#### Core Components
- **Users and Service Accounts**: Who is making the request
- **Roles and ClusterRoles**: What actions are allowed
- **RoleBindings and ClusterRoleBindings**: Connect users to roles

#### RBAC Best Practices
- **Principle of least privilege**: Give minimum required permissions
- **Use service accounts** for applications
- **Regular access reviews**: Remove unused permissions
- **Separate roles** by function (dev, ops, read-only)

### Pod Security
*Secure individual workloads*

#### Pod Security Standards
- **Privileged**: Unrestricted (not secure)
- **Baseline**: Minimally restrictive (good starting point)
- **Restricted**: Heavily restricted (most secure)

#### Security Contexts
- **Run as non-root user**
- **Drop unnecessary capabilities**
- **Use read-only root filesystem**
- **Set resource limits**

### Network Security
- **Network Policies**: Control pod-to-pod traffic
- **Ingress security**: Use TLS, authentication
- **Service mesh**: Advanced traffic management (Istio, Linkerd)

---

## Chapter 9: Management Tools
*Essential tools for working with Kubernetes*

### kubectl
*Your primary tool for interacting with Kubernetes*

#### Essential Commands
```bash
# View resources
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>

# Create/Update resources
kubectl apply -f deployment.yaml
kubectl create configmap app-config --from-file=config/

# Troubleshooting
kubectl exec -it <pod-name> -- /bin/bash
kubectl port-forward <pod-name> 8080:80
```

#### Configuration
- **kubeconfig file**: Stores cluster connection info
- **Contexts**: Switch between different clusters
- **Namespaces**: Organize resources

### Helm
*Package manager for Kubernetes*

#### What is Helm?
- **Package Kubernetes applications** into Charts
- **Template engine** for YAML files
- **Dependency management**
- **Release management** with versioning

#### Helm Concepts
- **Chart**: Package of Kubernetes resources
- **Release**: Instance of a chart running in cluster
- **Values**: Configuration parameters for charts
- **Repository**: Store and share charts

### Kustomize
*Configuration customization tool*

#### Purpose
- **Customize YAML files** without templates
- **Environment-specific configurations**
- **Built into kubectl** (kubectl apply -k)
- **GitOps friendly**

---

## Chapter 10: Monitoring and Observability
*Understanding what's happening in your cluster*

### The Three Pillars of Observability
1. **Metrics**: What is happening (numbers)
2. **Logs**: Why it's happening (events)
3. **Traces**: Where it's happening (request flow)

### Metrics
*Numerical data about system performance*

#### Prometheus Stack
- **Prometheus**: Metrics collection and storage
- **Grafana**: Visualization and dashboards
- **AlertManager**: Alert routing and management
- **Node Exporter**: System metrics
- **kube-state-metrics**: Kubernetes object metrics

#### Key Metrics to Monitor
- **Resource usage**: CPU, memory, disk, network
- **Application metrics**: Request rate, latency, errors
- **Cluster health**: Node status, pod status
- **Business metrics**: User signups, transactions

### Logging
*Collecting and analyzing log data*

#### Logging Architecture
- **Log collectors**: Fluent Bit, Fluentd, Filebeat
- **Log storage**: Elasticsearch, Loki, CloudWatch
- **Log analysis**: Kibana, Grafana, Splunk

#### Best Practices
- **Structured logging**: Use JSON format
- **Centralized logging**: Collect all logs in one place
- **Log retention**: Set appropriate retention policies
- **Security**: Don't log sensitive information

### Health Checks
*Ensuring applications are running correctly*

#### Probe Types
- **Liveness Probe**: Is the application running?
- **Readiness Probe**: Is the application ready to serve traffic?
- **Startup Probe**: Has the application started successfully?

#### Probe Methods
- **HTTP**: Check HTTP endpoint
- **TCP**: Check TCP socket
- **Exec**: Run command in container

---

## Chapter 11: Advanced Topics
*For experienced users ready to dive deeper*

### Custom Resources and Operators
*Extend Kubernetes with custom functionality*

#### Custom Resource Definitions (CRDs)
- **Define new resource types**
- **Extend Kubernetes API**
- **Store custom objects in etcd**

#### Operators
- **Automate application management**
- **Encode operational knowledge**
- **Handle complex lifecycle operations**
- **Examples**: Database operators, monitoring operators

### Advanced Networking
#### Service Mesh
- **Istio**: Full-featured service mesh
- **Linkerd**: Lightweight service mesh
- **Features**: Traffic management, security, observability

#### Advanced Ingress
- **Multiple ingress controllers**
- **Ingress classes**
- **Advanced routing rules**

### Multi-cluster Management
- **Cluster federation**
- **Multi-cloud deployments**
- **Disaster recovery**
- **Cross-cluster networking**

---

## Learning Path Recommendations

### For Beginners (Weeks 1-4)
1. **Week 1**: Chapters 1-2 (Introduction and Architecture)
2. **Week 2**: Chapter 3 (Pods and basic concepts)
3. **Week 3**: Chapter 4 (Deployments and workloads)
4. **Week 4**: Chapter 5 (Basic networking and services)

### For Intermediate Users (Weeks 5-8)
1. **Week 5**: Chapter 6 (Storage management)
2. **Week 6**: Chapter 7 (Configuration management)
3. **Week 7**: Chapter 8 (Security essentials)
4. **Week 8**: Chapter 9 (Management tools)

### For Advanced Users (Weeks 9-12)
1. **Week 9**: Chapter 10 (Monitoring and observability)
2. **Week 10-12**: Chapter 11 (Advanced topics and hands-on projects)

---

## Hands-on Labs and Exercises
*Practice exercises for each chapter*

### Lab Setup Options
1. **Minikube**: Local single-node cluster
2. **Kind**: Kubernetes in Docker
3. **Cloud providers**: EKS, GKE, AKS free tiers
4. **Online playgrounds**: Katacoda, Play with Kubernetes

### Practice Projects
1. **Deploy a web application** with database
2. **Set up monitoring** with Prometheus and Grafana
3. **Implement CI/CD pipeline** with GitOps
4. **Create custom operator** for application management
5. **Multi-environment setup** (dev/staging/prod)

---

## Additional Resources
- **Official Documentation**: kubernetes.io
- **Books**: "Kubernetes in Action", "Kubernetes: Up and Running"
- **Online Courses**: Kubernetes certification tracks (CKA, CKAD, CKS)
- **Community**: Kubernetes Slack, Stack Overflow, Reddit r/kubernetes