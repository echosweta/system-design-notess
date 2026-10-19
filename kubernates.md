Kubernetes — Compact Notes
1. History of deployment
1. Traditional deployment: Applications ran on physical servers. Scaling and maintenance were difficult.
2. Virtual machines: Multiple isolated operating systems could run on the same physical machine.
3. Cloud computing: Providers such as AWS made compute resources available on demand.
4. Containerization: Docker packaged applications and dependencies into portable containers.
5. Container orchestration: Managing many containers manually became difficult, leading to tools like Kubernetes.
2. What is Kubernetes?
- Kubernetes (K8s): Open-source container orchestration platform.
- Automates deployment, scaling, scheduling, and management of containers.
- Supports self-healing, high availability, and efficient resource utilization.
- Works across supported cloud and on-premises environments.
3. Why do we need Kubernetes?
Suppose an application needs 10 containers.

Manually managing their deployment, failures, scaling, and networking becomes difficult.

Kubernetes helps by:
- Maintaining the required number of replicas.
- Replacing failed Pods.
- Distributing workloads across nodes.
- Managing application updates and scaling.
4. Kubernetes architecture
A Kubernetes cluster consists of a Control Plane and Worker Nodes.
Control Plane
Component	Responsibility
API Server	Entry point for Kubernetes requests
etcd	Stores cluster state
Scheduler	Selects a worker node for an unscheduled Pod
Controller Manager	Reconciles desired state with actual state
Cloud Controller Manager (CCM)	Integrates with cloud-provider APIs
Worker Node
Component	Responsibility
kubelet	Ensures assigned Pods run on the node
kube-proxy	Traditionally manages Service networking rules
Container runtime	Runs containers, e.g. containerd or CRI-O
Pods	Run application containers
5. How Kubernetes works
Example: You request five replicas of an application.
1. Submit the configuration through the API Server.
2. Kubernetes stores the desired state in etcd.
3. The controller works to create five Pods.
4. The scheduler assigns unscheduled Pods to suitable nodes.
5. The kubelet works with the container runtime to start containers.
6. Kubernetes monitors the workload and reconciles failures.
Desired state vs. current state
- Desired state: Five replicas should run.
- Current state: Only three replicas are running.
- Kubernetes: Works to restore the missing two replicas.
If you change the desired count to one, Kubernetes scales down the excess replicas.

Core concept: Kubernetes continuously tries to make the current state match the desired state.
6. Container Runtime Interface (CRI)
- CRI allows Kubernetes' kubelet to communicate with compatible container runtimes.
- Examples: containerd and CRI-O.
- Docker is not required as the runtime in modern Kubernetes clusters.
7. Cloud Controller Manager (CCM)
Different cloud providers have different APIs and infrastructure services.

For example, AWS, DigitalOcean, and GCP provide their own load-balancing integrations.

The Cloud Controller Manager connects Kubernetes with supported cloud-provider functionality, allowing Kubernetes resources to interact with cloud infrastructure.
8. Final interview revision
- Docker: Builds images and runs containers.
- Kubernetes: Orchestrates containerized applications.
- API Server: Accepts and validates requests.
- etcd: Stores cluster state.
- Scheduler: Selects nodes for Pods.
- Controller Manager: Reconciles desired and actual states.
- kubelet: Manages Pods on a worker node.
- CRI: Interface between kubelet and container runtime.
- CCM: Integrates with cloud-provider APIs.
- Pod: Smallest deployable unit in Kubernetes.
One-line summary: Kubernetes manages containerized applications by maintaining the desired state across a cluster of machines.
