## Step 1: Create Kind Cluster

kind-config.yml
```
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

Create cluster:
```
kind create cluster --name argocd-cluster --config kind-config.yml
```

Verification:
```
kind get clusters
kubectl cluster-info
kubectl get nodes
```

## Step 2: Install ArgoCD using Helm (recommended for customization/production)

### 1. Add Argo Helm repo

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### 2. Create namespace

```bash
kubectl create namespace argocd
```

### 3. Install ArgoCD

```bash
helm install argocd argo/argo-cd -n argocd
```

### 4. Verify installation

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
```

### 5. Access the ArgoCD UI

Port-forward the service:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443 --address=0.0.0.0 &
```

Now open → **[https://localhost:8080](https://localhost:8080)**

### 6. Get initial admin password

```bash
kubectl get secret argocd-initial-admin-secret -n argocd \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Login with:

* Username: `admin`
* Password: (above output)

## Step 3: Install ArgoCD CLI (Ubuntu/Linux)

ArgoCD server runs inside Kubernetes, but to interact with it from the terminal you need the **ArgoCD CLI (`argocd`)**.  
This is separate from the server installation.

### 1. Install ArgoCD CLI

```bash
curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd
rm argocd-linux-amd64
```

### 2. Verify installation

```bash
# Verify installation
argocd version --client
```

### 3. Login to ArgoCD CLI

```bash
argocd login localhost:8080 --username admin --password <initial_password> --insecure
```

> Note: The --insecure flag is required when using port-forward with self-signed TLS certs.
For production, you’d configure proper TLS certs (then --insecure is not needed).

### 4. Get user info

```bash
argocd account get-user-info
```

## Step 4: Adding Cluster to ArgoCD server
1. Check your config contexts:

```bash
kubectl config get-contexts
```

Identify your cluster context (e.g., `kind-argocd-cluster`).

2. Add the cluster to ArgoCD:

```bash
argocd cluster add kind-argocd-cluster --name argocd-cluster --insecure --in-cluster -y --upsert
```

3. Verify using:

```bash
argocd cluster list
```

## Step 5: Create Application in ArgoCD UI

1. In ArgoCD UI, Go to **Applications** and click **New App**.
2. Fill the fields:

   * **App Name**: `nginx-app`
   * **Project**: `default`
   * **Repository URL**: `<select_the_connected_repo>` 
   * **Revision**: `main`
   * **Path**: `ui_approach/nginx`
   * **Cluster**: `<select_added_cluster_url>`
   * **Namespace**: `default`
  
3. Leave Sync Policy as **Manual** for now.
4. Click **Create**.


## Step 6: Sync the Application

* The app will show as **OutOfSync**.
* Click **Sync → Synchronize**.
* ArgoCD will apply `nginx_deployment.yml` and `nginx_svc.yml` into the `default` namespace.
* After syncing, the status should change to **Synced** and **Healthy**.


## Step 7: Verify the Deployment

From CLI:

```bash
kubectl get pods -n default
kubectl get svc -n default
```

You should see:

* NGINX pods running (`nginx-deployment-xxxx`).
* `nginx-service` of type ClusterIP exposing port 80.

## Step 8: Access Nginx via browser

1. Port-forward the NGINX service:

```bash
kubectl port-forward svc/nginx-service 8081:80 --address=0.0.0.0 &
```

2. Open the inbound rule for port 8081 on your EC2 instance

3. Access the Nginx app at:

```bash
http://localhost:8081
```


## Step 9: Testing

### Make a Change in Git

1. Open `nginx_deployment.yml` in your local repo.
2. Change the replicas from 3 to 5

```yaml
spec:
  replicas: 5  # Change this from 3 to 5
``` 

3. Commit and push the change to GitHub:

```bash
git add nginx_deployment.yml
git commit -m "Increase replicas to 5"
git push origin main
```

### Observe ArgoCD UI

1. Go to ArgoCD UI, select `nginx-app`.
2. You should see the app is **OutOfSync** again.
3. Click **Sync → Synchronize** to apply the changes.
4. After syncing, the status should change to **Synced** and **Healthy**.
5. Verify the change:

```bash
kubectl get pods -n default
```
You should see 5 NGINX pods running now.

In your ArgoCD, in `nginx-app` you can see that pods are created from 3 to 5.

## Destroy 

For complete destroy, you can directly delete the cluster by using below command:

```bash
kind delete cluster --name argocd-cluster
```
