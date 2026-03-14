# How to Setup KIND with ArgoCD in Windows 11

## Prerequisites
1. **Install Docker Desktop**: Ensure Docker Desktop is installed and running. Verify that Docker commands work by running:
   ```powershell
   docker --version
   ```

2. **Install kubectl**: Use the following PowerShell command to install `kubectl`:
   ```powershell
   winget install -e --id Kubernetes.kubectl
   ```

3. **Install Kind**: Use the following PowerShell command to install `Kind`:
   ```powershell
   winget install Kubernetes.kind
   ```

## Step 1 — Create Cluster YAML Files
Ensure you have the YAML files for each cluster configuration:
- `argocd-cluster.yaml`
- `cluster-a.yaml`
- `cluster-b.yaml`

## Step 2 — Create the Clusters
Run the following commands to create the clusters:
```bash
kind create cluster --config argocd-cluster.yaml
kind create cluster --config cluster-a.yaml
kind create cluster --config cluster-b.yaml
```

## Step 3 — Verify Clusters and Nodes
Check the created clusters and their nodes:
```bash
kind get clusters

kubectl get nodes --context kind-argocd
kubectl get nodes --context kind-cluster-a
kubectl get nodes --context kind-cluster-b
```

## Step 4 — Install ArgoCD
Switch to the `kind-argocd` context:
```bash
kubectl config use-context kind-argocd
```

Create a namespace for ArgoCD:
```bash
kubectl create namespace argocd
```

Install ArgoCD in the `argocd` namespace:
```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for all ArgoCD pods to be ready:
```bash
kubectl wait --for=condition=Ready pod --all -n argocd --timeout=180s
```

## Step 5 — Retrieve ArgoCD Admin Password
Run the following command in a PowerShell window to get the admin password:
```powershell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | % { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }
```

### Explanation:
- `kubectl -n argocd get secret argocd-initial-admin-secret`: Retrieves the secret containing the initial admin password for ArgoCD in the `argocd` namespace.
- `-o jsonpath="{.data.password}"`: Extracts the password field from the secret in Base64 format.
- `% { [System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($_)) }`: Decodes the Base64-encoded password into plain text using PowerShell.

## Step 6 — Port-Forward ArgoCD UI
Run this command in another PowerShell window to access the ArgoCD UI:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
> **Note:** The session will terminate if the window is closed.

## Step 7 — Get Internal Docker IPs
Retrieve the internal IPs of the control planes:
```bash
docker inspect cluster-a-control-plane --format "{{.NetworkSettings.Networks.kind.IPAddress}}"
docker inspect cluster-b-control-plane --format "{{.NetworkSettings.Networks.kind.IPAddress}}"
```

## Step 8 — Get Host-Mapped Ports
Find the host-mapped ports for the clusters:
```bash
docker port cluster-a-control-plane 6443
docker port cluster-b-control-plane 6443
```

## Step 9 — Copy Kubeconfig into ArgoCD Pod
Switch to the `kind-argocd` context:
```bash
kubectl config use-context kind-argocd
```

Copy the kubeconfig file into the ArgoCD pod:
```powershell
$POD = kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o jsonpath="{.items[0].metadata.name}"
Get-Content C:\Users\<username>\.kube\config | kubectl exec -i $POD -n argocd -- sh -c "cat > /tmp/kubeconfig"
```

### Explanation:
- `$POD = kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o jsonpath="{.items[0].metadata.name}"`: Retrieves the name of the ArgoCD server pod by filtering pods in the `argocd` namespace with the label `app.kubernetes.io/name=argocd-server`.
- `Get-Content C:\Users\<username>\.kube\config`: Reads the local kubeconfig file.
- `kubectl exec -i $POD -n argocd -- sh -c "cat > /tmp/kubeconfig"`: Executes a command inside the ArgoCD pod to write the kubeconfig file to `/tmp/kubeconfig`.

## Step 10 — Exec into ArgoCD Pod
Access the ArgoCD pod:
```bash
kubectl exec -it $POD -n argocd -- bash
```

## Step 11 — Patch IPs and Register Clusters
Inside the pod, patch the localhost URLs to internal Docker IPs:
```bash
sed -i 's|https://127.0.0.1:55716|https://172.19.0.2:6443|g' /tmp/kubeconfig
sed -i 's|https://127.0.0.1:61071|https://172.19.0.7:6443|g' /tmp/kubeconfig
```

Verify the changes:
```bash
grep server /tmp/kubeconfig
```

Login to ArgoCD:
```bash
argocd login localhost:8080 --username admin --password YOUR_PASSWORD --insecure
```

Register the clusters:
```bash
argocd cluster add kind-cluster-a --name cluster-a --kubeconfig /tmp/kubeconfig
argocd cluster add kind-cluster-b --name cluster-b --kubeconfig /tmp/kubeconfig
```

Exit the pod:
```bash
exit
```

## Step 12 — Restore Local Kubeconfig
Restore the original kubeconfig settings:
```bash
kubectl config set-cluster kind-cluster-a --server=https://127.0.0.1:55716
kubectl config set-cluster kind-cluster-b --server=https://127.0.0.1:61071
```

## Step 13 — Verify in ArgoCD UI
Access the ArgoCD UI at [https://localhost:8080](https://localhost:8080):
1. Navigate to **Settings → Clusters**.
2. Verify that `cluster-a`, `cluster-b`, and `in-cluster` are listed. ✅

## Test the ArgoCD Deployment

### Step 1 — Clone the Demo Repository
This repository contains the necessary YAML files for deploying NGINX:
[argo-cd-demo](https://github.com/Ron-Algorithmx/argo-cd-demo)

### Step 2 — Create Applications in ArgoCD
Login to the ArgoCD UI and create two applications. Use the "Edit as YAML" option to define the applications.

#### Application for Cluster A
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-cluster-a
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Ron-Algorithmx/argo-cd-demo
    targetRevision: HEAD
    path: apps/nginx
  destination:
    server: https://172.19.0.2:6443
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

#### Application for Cluster B
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-cluster-b
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Ron-Algorithmx/argo-cd-demo
    targetRevision: HEAD
    path: apps/nginx
  destination:
    server: https://172.19.0.7:6443
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Step 3 — Watch ArgoCD Sync the Applications
Go to [https://localhost:8080](https://localhost:8080). You should see two new applications appear:
- **nginx-cluster-a** → syncing → healthy ✅
- **nginx-cluster-b** → syncing → healthy ✅

### Step 4 — Verify Pods are Running on Each Cluster
Run the following commands to check the status of the pods:
```powershell
kubectl get pods --context kind-cluster-a
kubectl get pods --context kind-cluster-b
```

You should see the NGINX pods running in the `default` namespace on both clusters.