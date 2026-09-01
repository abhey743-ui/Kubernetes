# Kubernetes Dashboard Setup Using Helm — Complete Guide (Windows 11 / Docker Desktop)

This is your **complete, working setup guide** for installing and accessing the Kubernetes Dashboard locally — combining the Helm install process with the ServiceAccount/RBAC/token steps you actually ran, fixed for the repo URL issue.

---

## 1. What You're Setting Up

- **Helm** → installs the Dashboard app into your cluster.
- **Kubernetes Dashboard** → a web UI to visually manage your cluster.
- **ServiceAccount** → an identity the Dashboard login uses.
- **ClusterRoleBinding** → grants that identity admin permissions.
- **Secret/Token** → the actual credential you paste into the Dashboard login screen.
- **Port-forward** → the tunnel from your browser to the Dashboard running inside the cluster.

---

## 2. Prerequisites Check

```powershell
kubectl version --client
kubectl get nodes
```
Confirms `kubectl` works and your Docker Desktop cluster node shows `Ready`.

Install Helm if not already installed:
```powershell
choco install kubernetes-helm
```
Verify:
```powershell
helm version
```

---

## 3. Go to Your Working Folder

```powershell
cd ([Environment]::GetFolderPath("Desktop"))
cd K8s
```
(This auto-resolves your Desktop path, which works even though yours is redirected into OneDrive.)

---

## 4. Install Kubernetes Dashboard via Helm

Add the Helm repo:
```powershell
helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard/
```

⚠️ **If you get this error:**
```
Error: looks like "https://kubernetes.github.io/dashboard/" is not a valid chart repository or cannot be reached: failed to fetch https://kubernetes.github.io/dashboard/index.yaml : 404 Not Found
```
It's because the official Dashboard repo was recently retired/archived. Use the community-maintained URL instead:
```powershell
helm repo remove kubernetes-dashboard
helm repo add kubernetes-dashboard https://kubernetes-retired.github.io/dashboard
```

Update and install:
```powershell
helm repo update
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard
```

If you see:
```
Congratulations! You have just installed Kubernetes Dashboard in your cluster.
```
...the install worked, and Helm itself will print the port-forward command to use (Step 8 below).

Verify the pods are running:
```powershell
kubectl get pods -n kubernetes-dashboard
```

---

## 5. Create the ServiceAccount (Login Identity)

Create the file:
```powershell
New-Item dashboard-adminuser.yaml -ItemType File
```

Paste this into `dashboard-adminuser.yaml`:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
```

Apply it:
```powershell
kubectl apply -f dashboard-adminuser.yaml
```
Expected output: `serviceaccount/admin-user created` (or `unchanged` if it already exists — that's fine, it just means nothing needed to change).

---

## 6. Create the ClusterRoleBinding (Grant Admin Access)

Create the file:
```powershell
New-Item dashboard-rolebinding.yaml -ItemType File
```

Paste this into `dashboard-rolebinding.yaml`:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: admin-user
    namespace: kubernetes-dashboard
```

Apply it:
```powershell
kubectl apply -f dashboard-rolebinding.yaml
```
Expected output: `clusterrolebinding.rbac.authorization.k8s.io/admin-user created` (or `unchanged`).

> ⚠️ This grants **full cluster-admin** access — fine for local learning, too broad for production (see Section 10).

---

## 7. Get a Login Token — Two Methods

### Method A: Quick Token (Recommended, No Extra File Needed)
```powershell
kubectl -n kubernetes-dashboard create token admin-user
```
This prints a token directly in your terminal. Copy it — you'll paste it into the Dashboard login screen. Note: this token **expires** (default ~1 hour), so re-run this command whenever it expires.

### Method B: Long-Lived Secret-Based Token
Create the file:
```powershell
New-Item secret.yaml -ItemType File
```

Paste this into `secret.yaml`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: "admin-user"
type: kubernetes.io/service-account-token
```

Apply it:
```powershell
kubectl apply -f secret.yaml
```

Retrieve the token (Windows PowerShell version, since `base64 -d` isn't native to PowerShell):
```powershell
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String((kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath="{.data.token}")))
```

👉 **Which should you use?** Method A is simpler and safer (short-lived). Use Method B only if you want a token that doesn't expire while you're actively learning.

---

## 8. Start the Dashboard (Port-Forward)

```powershell
kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443
```

- Leave this terminal window **open and running** — closing it or pressing `Ctrl+C` disconnects the Dashboard.
- This creates a temporary tunnel: `Browser → localhost:8443 → kubectl port-forward → Kubernetes Service → Dashboard Pod`.

Open your browser to:
```
https://localhost:8443
```
(Your browser will warn about a self-signed certificate — this is expected locally, click through/proceed.)

---

## 9. Log In

1. Choose **"Token"** as the login method on the Dashboard screen.
2. Paste the token you got from Section 7 (Method A or B).
3. You're in — you should now see your cluster's Pods, Deployments, Services, and Nodes visually.

---

## 10. Full Command Cheat Sheet (Quick Reference)

| Purpose | Command |
|---|---|
| Check kubectl | `kubectl version --client` |
| Check nodes | `kubectl get nodes` |
| Add working Dashboard repo | `helm repo add kubernetes-dashboard https://kubernetes-retired.github.io/dashboard` |
| Update repos | `helm repo update` |
| Install Dashboard | `helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard` |
| Check Dashboard pods | `kubectl get pods -n kubernetes-dashboard` |
| Check Dashboard service | `kubectl -n kubernetes-dashboard get svc` |
| Apply any YAML file | `kubectl apply -f filename.yaml` |
| Get quick token | `kubectl -n kubernetes-dashboard create token admin-user` |
| Port-forward Dashboard | `kubectl -n kubernetes-dashboard port-forward svc/kubernetes-dashboard-kong-proxy 8443:443` |
| Open Dashboard | `https://localhost:8443` |
| Uninstall Dashboard | `helm uninstall kubernetes-dashboard -n kubernetes-dashboard` |

---

## 11. Production Note (Why This Setup Is Learning-Only)

This exact setup (`cluster-admin` + open port-forward) is fine for local practice, but in a real company:
- You'd never bind an identity to `cluster-admin` casually — you'd create a scoped, least-privilege Role instead.
- You'd never rely on `kubectl port-forward` for real access — you'd use an Ingress + HTTPS behind a VPN/internal network.
- Many production teams skip the raw Dashboard entirely and use GitOps tools like ArgoCD instead, for better audit trails.

---

*Your complete, working Kubernetes Dashboard setup reference — combines the Helm install, ServiceAccount/RBAC, and token steps into one place.*
