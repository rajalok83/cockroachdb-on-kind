# CockroachDB on Kubernetes (Kind) - Complete Setup Guide

This guide provides step-by-step instructions to set up a multi-region CockroachDB cluster on a local Kubernetes cluster using Kind.

---

## 1. Install Kind

### Option A: Using Binary Download on Linux
```bash
# For Linux/macOS
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

```
#### On Windows
```powershell
# Download Kind binary
$KindVersion = "v0.20.0"
Invoke-WebRequest -Uri "https://kind.sigs.k8s.io/dl/$KindVersion/kind-windows-amd64" -OutFile "kind.exe"

# Add to PATH or move to a directory in PATH
Move-Item -Path kind.exe -Destination "C:\Program Files\kind\kind.exe" -Force

# Add C:\Program Files\kind to system PATH environment variable
[Environment]::SetEnvironmentVariable("PATH", $env:PATH + ";C:\Program Files\kind", "User")
```

### Verify Installation
```bash
kind version
# Output should show: kind v0.20.0 (or your installed version)
```

---

## 2. Install Additional Prerequisites

### Install kubectl
```bash
# For Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# Verify installation
kubectl version --client
```

#### Windows Installation
```powershell
# Using Direct Binary Download
$KubectlVersion = $(Invoke-WebRequest -Uri 'https://dl.k8s.io/release/stable.txt').Content.Trim()
Invoke-WebRequest -Uri "https://dl.k8s.io/release/$KubectlVersion/bin/windows/amd64/kubectl.exe" -OutFile "kubectl.exe"

# Move to PATH or add directory to PATH
Move-Item -Path kubectl.exe -Destination "C:\Program Files\kubectl\kubectl.exe" -Force
[Environment]::SetEnvironmentVariable("PATH", $env:PATH + ";C:\Program Files\kubectl", "User")

# Verify installation (open new PowerShell session after PATH update)
kubectl version --client
```

### Install Docker
```bash
# For Linux (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install -y docker.io

# Start Docker daemon
sudo systemctl start docker
sudo systemctl enable docker

# Add current user to docker group (optional, requires logout/login)
sudo usermod -aG docker $USER
```

#### Windows Installation

**Docker Desktop (Recommended)**
```powershell
# Download Docker Desktop from:
# https://www.docker.com/products/docker-desktop

# Or using Chocolatey
choco install docker-desktop

# Or using winget
winget install Docker.DockerDesktop

# After installation:
# 1. Start Docker Desktop application
# 2. Right-click system tray icon and ensure "Start Docker Desktop" is complete
# 3. Open new PowerShell terminal
# 4. Verify installation:
docker --version
```


**System Requirements for Windows:**
- Windows 10 or Windows 11 Pro/Enterprise/Education
- Hyper-V or WSL 2 enabled
- Virtualization enabled in BIOS

**Enable WSL 2 (Windows Subsystem for Linux 2):**
```powershell
# Run as Administrator
dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart

# Download and install Linux kernel update:
# https://wslstorestorage.blob.core.windows.net/wslblob/wsl_update_x64.msi

# Set WSL 2 as default
wsl --set-default-version 2

# Restart computer
Restart-Computer
```

---

## 3. Create Kind Cluster

### Step 1: Delete Existing Cluster (if any)
```bash
# Delete previous cluster named "ckdb" if it exists
kind delete cluster --name ckdb

# Verify deletion
kind get clusters
```

### Step 2: Create New Kind Cluster
```bash
# Create cluster using kind.yaml configuration
kind create cluster --name ckdb --config kind.yaml

# Note: The kind.yaml defines:
# - Cluster name: c0 (will be created as "ckdb")
# - 1 control-plane node
# - 6 worker nodes (3 in us-east-1, 3 in us-west-1)
# - Port mappings: 31257 (gRPC) and 8080 (HTTP Console)
```

### Step 3: Verify Cluster Creation
```bash
# List all Kind clusters
kind get clusters

# Check cluster nodes
kubectl get nodes

# Output should show 1 control-plane and 6 worker nodes
```

### Step 4: Check Node Labels
```bash
# Verify node labels for regions
kubectl get nodes --show-labels | grep region

# Output example:
# ckdb-worker              Ready   worker   10s   region=us-east-1
# ckdb-worker2             Ready   worker   10s   region=us-east-1
# ckdb-worker3             Ready   worker   10s   region=us-east-1
# ckdb-worker4             Ready   worker   10s   region=us-west-1
# ckdb-worker5             Ready   worker   10s   region=us-west-1
# ckdb-worker6             Ready   worker   10s   region=us-west-1
```

### Windows-Specific Notes

**Docker Desktop with WSL 2:**
```powershell
# If using WSL 2 backend, nodes will be accessible from Windows via localhost
# If using Hyper-V backend, use the VM's IP address instead

# Get Kind cluster info
kind get kubeconfig --name ckdb > $env:USERPROFILE\.kube\config-ckdb

# Set KUBECONFIG
$env:KUBECONFIG = "$env:USERPROFILE\.kube\config-ckdb"
```

**Firewall Considerations:**
```powershell
# Windows Firewall may need configuration for port forwarding
# Ports needed: 26257 (gRPC), 8080 (Console), 31257 (NodePort), 31080 (NodePort Console)

# Check current port-forwarding rules
netsh interface portproxy show all

# If needed, add port forwarding rules
netsh interface portproxy add v4tov4 listenport=26257 listenaddress=localhost connectport=26257 connectaddress=localhost
netsh interface portproxy add v4tov4 listenport=8080 listenaddress=localhost connectport=8080 connectaddress=localhost
```

---

## 4. Create CockroachDB Cluster

### Step 1: Apply CockroachDB Manifest
```bash
# Apply the ckdb.yaml manifest to create CockroachDB cluster
kubectl apply -f ckdb.yaml

# This creates:
# - ckdb namespace
# - cockroachdb headless service
# - cockroachdb-public NodePort service
# - 2 StatefulSets (us-east-1 and us-west-1) with 3 replicas each
# - Auto-init job
```

### Step 2: Verify Manifest Application
```bash
# Check if namespace was created
kubectl get namespaces | grep ckdb

# Check services in ckdb namespace
kubectl get svc -n ckdb

# Output example:
# NAME                    TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
# cockroachdb             ClusterIP   None         <none>        26257/TCP,8080/TCP
# cockroachdb-public      NodePort    10.96.x.x    <none>        26257:31257/TCP,8080:31080/TCP
```

---

## 5. Monitor Cluster Initialization Progress

### Step 1: Watch StatefulSet Rollout
```bash
# Watch the StatefulSet deployment progress for us-east-1
kubectl rollout status statefulset/cockroachdb-us-east-1 -n ckdb --timeout=5m

# Watch the StatefulSet deployment progress for us-west-1
kubectl rollout status statefulset/cockroachdb-us-west-1 -n ckdb --timeout=5m
```

### Step 2: Monitor Pod Status
```bash
# Watch pods in real-time
kubectl get pods -n ckdb -w

# Expected output (wait until all pods are Running):
# NAME                              READY   STATUS    RESTARTS   AGE
# cockroachdb-us-east-1-0           1/1     Running   0          2m
# cockroachdb-us-east-1-1           1/1     Running   0          1m
# cockroachdb-us-east-1-2           1/1     Running   0          30s
# cockroachdb-us-west-1-0           1/1     Running   0          2m
# cockroachdb-us-west-1-1           1/1     Running   0          1m
# cockroachdb-us-west-1-2           1/1     Running   0          30s
# cockroachdb-init-xxxxx            0/1     Completed 0          1m
```

### Step 3: Check YAML Logs
```bash
# One-time check of pod status
kubectl get pods -n ckdb -o wide

# Check pod descriptions for details
kubectl describe pod cockroachdb-us-east-1-0 -n ckdb

# View logs from the init job
kubectl logs -n ckdb -l job-name=cockroachdb-init
```

### Step 4: Wait for Ready State
```bash
# Wait for all CockroachDB pods to be ready
kubectl wait --for=condition=ready pod -l app=cockroachdb -n ckdb --timeout=5m

# Alternative: Watch until Running
kubectl get pods -n ckdb --field-selector=app=cockroachdb --watch
```

---

## 6. Validate Services

### Step 1: Check Service Endpoints
```bash
# View service details
kubectl get svc -n ckdb -o wide

# Check service endpoints (should show pod IPs)
kubectl get endpoints -n ckdb

# Example output:
# NAME                  ENDPOINTS                                    AGE
# cockroachdb           10.244.x.x:26257,10.244.x.x:26257,...       2m
# cockroachdb-public    10.244.x.x:26257,10.244.x.x:26257,...       2m
```

### Step 2: Verify Service DNS Resolution
```bash
# Test DNS resolution within cluster
kubectl run -it --rm debug --image=nicolaka/netshoe/netshoot --restart=Never -- nslookup cockroachdb.ckdb

# Alternative: Test from a pod
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
# Inside the pod shell:
# nslookup cockroachdb.ckdb
# nslookup cockroachdb-public.ckdb
```

### Step 3: Test gRPC Port Connectivity
```bash
# Port-forward to test local connectivity
kubectl port-forward -n ckdb svc/cockroachdb-public 26257:26257 &

# In another terminal, verify connection
nc -zv localhost 26257
# Output: Connection to localhost 26257 port [tcp/*] succeeded!

# Stop port-forward
fg
# (Press Ctrl+C)
```

### Step 4: Verify NodePort Accessibility
```bash
# Get node IP
kubectl get nodes -o wide | grep "INTERNAL-IP"

# Test direct nodeport connection (replace NODE-IP with actual IP)
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
nc -zv $NODE_IP 31257
```

---

## 7. Connect to CockroachDB

#### Step 1: Set Up Port-Forward
```bash
# Forward local port 26257 to CockroachDB gRPC port
kubectl port-forward -n ckdb svc/cockroachdb-public 26257:26257 &

# Forward local port 8080 to DB Console
kubectl port-forward -n ckdb svc/cockroachdb-public 8080:8080 &

# Save the background job PIDs for later cleanup
# You can check with: jobs -l
```
### Option A: Using psql (PostgreSQL Compatible)

```bash
# Install psql (if not already installed)
sudo apt-get install postgresql-client

# Connect to CockroachDB
psql -h localhost -p 26257 -U root postgres --insecure

# Inside psql:
# postgres=> CREATE DATABASE testdb;
# postgres=> \c testdb
# postgres=> CREATE TABLE users (id INT PRIMARY KEY, name STRING);
# postgres=> INSERT INTO users VALUES (1, 'Alice'), (2, 'Bob');
# postgres=> SELECT * FROM users;
# postgres=> \q (to exit)
```

### Option B: Using kubectl exec (Direct Pod Access)

```bash
# Execute SQL commands directly in a pod
kubectl exec -it -n ckdb cockroachdb-us-east-1-0 -- cockroach sql --insecure

# Inside the pod's SQL shell:
# root@cockroachdb-us-east-1-0/defaultdb> CREATE DATABASE testdb;
# root@cockroachdb-us-east-1-0/defaultdb> \q (to exit)
```

---

## 8. Access CockroachDB Console

### Step 1: Set Up Port-Forward for Console
```bash
# Forward local port 8080 to DB Console (if not already done)
kubectl port-forward -n ckdb svc/cockroachdb-public 8080:8080 &
```

### Step 2: Open Browser
```bash
# Using curl to verify console is accessible
curl -I http://localhost:8080

# Expected output:
# HTTP/1.1 200 OK
# Content-Type: text/html; charset=utf-8
# ...
```

#### Windows Verification
```powershell
# Using Invoke-WebRequest to verify console is accessible
Invoke-WebRequest -Uri http://localhost:8080 -UseBasicParsing

# Or using curl (if installed)
curl -I http://localhost:8080
```

### Step 3: View Console UI
```
Open in your browser:
http://localhost:8080

Features visible in the console:
- Cluster overview
- Node status
- Database metrics
- SQL user management
- Security settings
```
---

## 9. Cleanup and Troubleshooting

### Stop Port-Forwards
```bash
# List all background jobs
jobs -l

# Kill specific port-forward job (replace %1 with job number)
kill %1
kill %2

# Or kill all background jobs
killall kubectl
```

#### Windows PowerShell
```powershell
# List all processes running kubectl
Get-Process kubectl

# Kill specific process by ID
Stop-Process -Id <ProcessId> -Force

# Or kill all kubectl processes
Stop-Process -Name kubectl -Force

# Verify cleanup
Get-Process kubectl -ErrorAction SilentlyContinue
```

### Delete Cluster
```bash
# Delete the entire Kind cluster
kind delete cluster --name ckdb

# Verify deletion
kind get clusters
```

### Troubleshooting Common Issues

#### Issue: Pods stuck in Pending state
```bash
# Check events for resources
kubectl describe pod cockroachdb-us-east-1-0 -n ckdb | tail -20

# Check PersistentVolumeClaim status
kubectl get pvc -n ckdb

# Check node capacity
kubectl describe nodes | grep -A 5 "Allocated resources"

# Check local storage provisioner 
kubectl logs -n local-path-storage -l app=local-path-provisioner

# You might probably see too many files open, set below to resolve on host Linux OS

sudo sysctl fs.inotify.max_user_watches=524288
sudo sysctl fs.inotify.max_user_instances=512


```

#### Issue: Init job failed
```bash
# Check init job logs
kubectl logs -n ckdb -l job-name=cockroachdb-init

# Check if pods can communicate
kubectl exec -it -n ckdb cockroachdb-us-east-1-0 -- ping cockroachdb-us-west-1-0.cockroachdb

# Restart the job
kubectl delete job cockroachdb-init -n ckdb
kubectl apply -f ckdb.yaml
```

#### Issue: Cannot connect to database
```bash
# Verify service is running
kubectl get svc -n ckdb

# Check port-forward is active
netstat -tlnp | grep 26257

# Verify pod network connectivity
kubectl exec -it -n ckdb cockroachdb-us-east-1-0 -- netstat -tlnp | grep 26257
```

#### Issue: Console not accessible at localhost:8080
```bash
# Verify port-forward for console is active
netstat -tlnp | grep 8080

# Check if another service is using port 8080
sudo lsof -i :8080

# Use a different local port
kubectl port-forward -n ckdb svc/cockroachdb-public 9000:8080
# Now access: http://localhost:9000
```

### Windows-Specific Troubleshooting

#### Issue: Docker Desktop not starting / WSL 2 issue
```powershell
# Check WSL 2 installation
wsl --list --verbose

# Update WSL 2 kernel
wsl --update

# Restart Docker Desktop from Settings > General > Restart Docker Desktop
```

#### Issue: Port forwarding not accessible
```powershell
# Check if kubectl process is running
Get-Process kubectl

# Verify port is listening
netstat -ano | findstr :26257
netstat -ano | findstr :8080

# If port is in use, find the process
netstat -ano | findstr :26257 | findstr -o "^[^ ]* [^ ]* [^ ]* [^ ]* [^ ]* [^ ]* [^ ]* [^ ]* [0-9]*$"

# Kill the process
taskkill /PID <ProcessId> /F
```

#### Issue: Kubeconfig permissions denied
```powershell
# Ensure kubeconfig has proper permissions
$KubeConfig = "$env:USERPROFILE\.kube\config"

# Change ownership to current user
icacls $KubeConfig /grant:r "$env:USERNAME`:F"

# Set proper permissions (readable by user only)
icacls $KubeConfig /inheritance:r /grant:r "$env:USERNAME`:F"
```

#### Issue: Networks/DNS issues in Windows
```powershell
# Restart Docker Desktop
Restart-Service Docker

# Clear DNS cache
ipconfig /flushdns

# Restart Kubernetes services
kubectl rollout restart statefulset/cockroachdb-us-east-1 -n ckdb
kubectl rollout restart statefulset/cockroachdb-us-west-1 -n ckdb
```

#### Issue: Kind cluster not visible from Windows
```powershell
# If using Hyper-V backend, get the VM IP
$KindVM = Get-VM -Name "kind-control-plane" -ErrorAction SilentlyContinue
if ($KindVM) {
    $KindVM | Select-Object Name, State
    Get-VMNetworkAdapter -VM $KindVM | Select-Object IPAddresses
}

# For WSL 2 backend, use localhost
# For Hyper-V backend, use the VM's IP from the output above
```

---

## 10. Quick Reference Commands

### Common Operations
```bash
# View all CockroachDB resources
kubectl get all -n ckdb

# View cluster status
kubectl cluster-info

# Get Kind cluster info
kind get kubeconfig --name ckdb

# Export kubeconfig for Kind cluster
kind export kubeconfig --name ckdb

# Check cluster events
kubectl get events -n ckdb --sort-by='.lastTimestamp'

# Monitor resource usage
kubectl top nodes
kubectl top pods -n ckdb

# View detailed pod information
kubectl describe nodes
kubectl describe pods -n ckdb
```

### Database Operations
```bash
# Create database
cockroach sql --insecure --host localhost:26257 -e "CREATE DATABASE mydb;"

# Backup database
cockroach dump defaultdb --insecure --host localhost:26257 > backup.sql

# Show all databases
cockroach sql --insecure --host localhost:26257 -e "SHOW DATABASES;"

# Show cluster settings
cockroach sql --insecure --host localhost:26257 -e "SHOW CLUSTER SETTINGS;"
```

---

## Summary

1. **Install Kind** - Set up the local Kubernetes cluster tool
2. **Create Cluster** - Deploy Kind cluster with 7 nodes (1 control-plane, 6 workers)
3. **Deploy CockroachDB** - Apply manifest to create multi-region cluster
4. **Monitor Progress** - Watch pods reach ready state
5. **Validate Services** - Verify cluster communication
6. **Connect** - Use CLI, psql, or kubectl exec to access database
7. **Access Console** - View UI at http://localhost:8080
8. **Cleanup** - Delete cluster when finished

---

## Additional Resources

- [Kind Documentation](https://kind.sigs.k8s.io/)
- [CockroachDB on Kubernetes](https://www.cockroachlabs.com/docs/stable/kubernetes-overview.html)
- [Kubectl Commands](https://kubernetes.io/docs/reference/kubectl/)
- [CockroachDB SQL Reference](https://www.cockroachlabs.com/docs/stable/sql-statements.html)
