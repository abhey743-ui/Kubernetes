# 9. Kubernetes Dashboard Example — Putting It All Together

This note ties together everything from notes 4, 5, 6, and 8 into one concrete story: **how you get admin access to the Kubernetes Dashboard using a ServiceAccount, a token Secret, and a ClusterRoleBinding.**

## The three pieces, side by side

```yaml
# PIECE 1: WHO — the identity
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kubernetes-dashboard

---
# PIECE 2: PROOF — the credential
apiVersion: v1
kind: Secret
metadata:
  name: admin-user
  namespace: kubernetes-dashboard
  annotations:
    kubernetes.io/service-account.name: "admin-user"
type: kubernetes.io/service-account-token

---
# PIECE 3: PERMISSIONS — what it can do
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

## How they work together

```
+-------------------+     +----------------------+     +--------------------------+
|  ServiceAccount    |     |   Secret (token)      |     |   ClusterRoleBinding      |
|  admin-user        |<----|   annotation points    |     |   subjects -> admin-user  |
|  (an identity,      |     |   to admin-user        |     |   roleRef -> cluster-admin|
|   nothing more)     |     |   (proof of identity)  |     |   (grants full power)    |
+-------------------+     +----------------------+     +--------------------------+
        ^                          |                              |
        |                          v                              |
        |                 Paste this token                        |
        |                 into the Dashboard                      |
        |                 login screen                            |
        |                          |                               |
        +----- API Server recognizes identity from token ----------+
                                   |
                                   v
                    RBAC checked -> cluster-admin -> ALLOWED
                                   |
                                   v
                Dashboard shows/does anything in the whole cluster
```

## Step-by-step walkthrough

1. You create the **ServiceAccount** `admin-user`. Right now, it's just a name — it can't log in anywhere and can't do anything.
2. You create the **Secret** with `type: kubernetes.io/service-account-token`, annotated to point at `admin-user`. Kubernetes automatically fills this Secret with a real signed token.
3. You retrieve that token (e.g. `kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath='{.data.token}' | base64 -d`) and paste it into the **Kubernetes Dashboard's login screen**.
4. Every action you now take in the Dashboard sends this token along with the request to the API Server.
5. The API Server authenticates the token → confirms "this is `admin-user`."
6. The API Server checks RBAC → finds the **ClusterRoleBinding** connecting `admin-user` to `cluster-admin` → authorizes the action.
7. The Dashboard can now show you every namespace, every workload, every Secret, and let you create/edit/delete anything — because that's exactly what `cluster-admin` grants.

## Important: the Secret itself does NOT give admin permissions

This is the key misconception this whole topic tends to create. Look again at the Secret's YAML — there is **nothing about permissions in it at all**. It only contains:

- A pointer to which ServiceAccount it belongs to (the annotation).
- The generated token itself.

```
Secret (token)  =  ONLY proves identity
                    contains ZERO permission information
```

If you deleted the ClusterRoleBinding but kept the Secret and ServiceAccount, the token would still work perfectly fine for **authentication** — the API Server would still recognize it as `admin-user` — but every single action would be **denied**, because there'd be no RBAC rule granting it anything. You'd be a "confirmed nobody with no permissions."

## The ClusterRoleBinding is what actually grants cluster-admin

All the real power comes from exactly one object: the **ClusterRoleBinding**. It's the only piece in this whole setup that connects an identity to actual permissions. Remove it, and `admin-user` goes from "can do literally anything" to "authenticated, but can't do anything at all" — instantly, with no other change needed.

## Quick recap

- ServiceAccount = who the Dashboard logs in as.
- Secret (token) = the credential you paste into the login screen; proves identity only.
- ClusterRoleBinding = the only piece that actually grants `cluster-admin` power.
- Removing the ClusterRoleBinding alone is enough to strip all permissions, without touching identity or the token at all.
