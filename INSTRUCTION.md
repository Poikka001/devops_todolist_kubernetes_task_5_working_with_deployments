# Kubernetes Deployment & Autoscaling Instruction

This document provides step-by-step instructions on how to deploy the Django ToDo application into a Kubernetes cluster using the configured Deployment and Horizontal Pod Autoscaler (HPA), along with architectural justifications for the parameters chosen.

---

## 🚀 Deployment Instructions

### 1. Prerequisites
Ensure you have `kubectl` configured and connected to your running Kubernetes cluster (e.g., Minikube, Kind, or a cloud provider).

### 2. Prepare the Environment
Create the unified namespace for all application resources:
```bash
kubectl apply -f namespace.yml
```

### 3. Deploy the Application Components
Apply the Deployment, Services, Pods, and Horizontal Pod Autoscaler manifests to the cluster:
```bash
kubectl apply -f deployment.yml
kubectl apply -f clusterIp.yml
kubectl apply -f hpa.yml
```

### 4. Verify the Deployment
Check the status of all resources within the `mateapp` namespace to ensure everything is running smoothly:
```bash
kubectl get all -n mateapp
```

---

## 🛠 Architectural Decisions & Configurations Explained

### 1. Resource Requests and Limits (`deployment.yml`)
The container resources are configured with matching requests and limits:
* **Memory:** `128Mi` (Request) / `128Mi` (Limit)
* **CPU:** `60m` (Request) / `60m` (Limit)

**Justification:**
By setting `requests` exactly equal to `limits`, Kubernetes assigns the **Guaranteed QoS (Quality of Service)** class to the pods. 
* **Memory:** Django applications consume a predictable baseline amount of memory during startup and idle states. Setting a strict limit of 128Mi prevents potential memory leaks from destabilizing the cluster node, while guaranteeing the container always has its required memory allocated.
* **CPU:** 60m (0.06 cores) is perfectly scaled for an idle state or lightweight API processing. Keeping requests and limits synchronized ensures predictable performance and prevents CPU throttling during moderate traffic spikes before the HPA triggers.

### 2. RollingUpdate Strategy Configuration
```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 2
```

**Justification:**
With an initial state of `replicas: 2`, these settings ensure **Zero-Downtime Deployment**:
* `maxUnavailable: 1`: Specifies that during an update, at most 1 pod can be taken offline. This guarantees that at least 1 healthy pod is always running and actively serving user traffic.
* `maxSurge: 2`: Allows the deployment to temporarily spin up up to 2 additional new pods (reaching 4 total pods during the update process). This speeds up the rollout phase significantly while ensuring that new instances are fully ready to take over traffic before old ones are terminated.

### 3. Horizontal Pod Autoscaler (HPA) Configuration
```yaml
minReplicas: 2
maxReplicas: 5
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
        type: Utilization
        averageUtilization: 70
```

**Justification:**
* **Replica Limits (`2` to `5`):** The minimum scale matches our high-availability baseline (2 pods for redundancy). The maximum scale limits resource consumption to a total of 5 pods to avoid cluster resource exhaustion.
* **70% Target Threshold:** The target threshold is set to 70% for both CPU and Memory. This creates a safe operational buffer. If a sudden spike in traffic occurs, the existing pods can safely process current requests while Kubernetes provisions and initializes new replicas. 
* **Multi-metric scaling:** Autoscaling triggers if *either* CPU or Memory exceeds 70%, protecting the application from both compute-heavy tasks and unexpected data caching loads.

---

## 🌐 How to Access the App After Deployment

Since all resources are now correctly unified under the `mateapp` namespace, you can easily access and test the application using standard methods.

### Method 1: Via NodePort Service (Best for local clusters like Minikube)
If you are using Minikube, you can automatically fetch the URL for the NodePort service:
```bash
minikube service todoapp -n mateapp
```
Alternatively, locate your cluster node IP and access the app directly via the designated node port `30080`:
