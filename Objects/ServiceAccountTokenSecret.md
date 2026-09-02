# 5. ServiceAccount Token Secret

## The exact YAML

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

## Line by line

| Line | Meaning |
|---|---|
| `apiVersion: v1` | Secrets are part of the core Kubernetes API. |
| `kind: Secret` | This is a Secret object (holds sensitive data). |
| `metadata.name: admin-user` | The name of this specific Secret object. It happens to match the ServiceAccount's name here, but that's just a naming choice, not a rule (see note 7). |
| `metadata.namespace: kubernetes-dashboard` | This Secret lives in the same namespace as the ServiceAccount it's tied to. |
| `annotations: kubernetes.io/service-account.name: "admin-user"` | This is the line that actually **links** this Secret to the ServiceAccount named `admin-user`. Kubernetes reads this annotation to know which ServiceAccount to generate a token for. |
| `type: kubernetes.io/service-account-token` | This special type tells Kubernetes: "don't just store whatever data I give you — **generate a signed authentication token** for the linked ServiceAccount, and put it in this Secret." |

## Why is the token generated?

The ServiceAccount from note 4 is just a name — it has no way to prove its identity to anyone. A **token** is what gives it that proof. When you create a Secret with `type: kubernetes.io/service-account-token` and the right annotation, the Kubernetes control plane automatically:

1. Notices the special type + annotation.
2. Generates a cryptographically signed token for the named ServiceAccount.
3. Fills that token into the Secret's data automatically (you'll see fields like `token`, `ca.crt`, and `namespace` appear inside the Secret once Kubernetes finishes processing it).

You didn't write the token yourself — Kubernetes creates it for you, because you asked (via this YAML) for a token belonging to `admin-user`.

## What does the token represent?

The token is a **long string (a JWT — JSON Web Token)** that represents: *"I am proof that the bearer of this string is acting as the ServiceAccount `admin-user` in namespace `kubernetes-dashboard`."*

```
Token = a signed piece of proof
        "Whoever holds this string IS admin-user"
```

Anyone who has this token can present it to the Kubernetes API Server and be treated as if they were the `admin-user` ServiceAccount — which is exactly why tokens must be protected carefully.

## How is the token associated with the ServiceAccount?

Through the **annotation**, not through the Secret's name:

```yaml
annotations:
  kubernetes.io/service-account.name: "admin-user"
```

This annotation is the actual link. Kubernetes reads it and says: "generate a token for the ServiceAccount literally named `admin-user`, in this same namespace." If the Secret had been named something totally different (e.g. `dashboard-login-secret`), the association would still work perfectly fine, **as long as this annotation points to the right ServiceAccount name.**

## Why is the token stored in a Secret?

Because a token is sensitive — it's a piece of proof of identity, functionally similar to a password. Storing it as a **Secret** (rather than, say, a ConfigMap) means:

- It gets the same base64 storage treatment and more careful default handling as other sensitive Kubernetes data.
- Access to it can be restricted using RBAC, just like any other Secret.

## How is the token used to authenticate with the Kubernetes API Server?

```
 1. Client reads the token out of the Secret
         |
         v
 2. Client sends a request to the Kubernetes API Server
    with header: "Authorization: Bearer <token>"
         |
         v
 3. API Server verifies the token's signature
         |
         v
 4. API Server recognizes: "this request is coming from
    ServiceAccount admin-user in namespace kubernetes-dashboard"
         |
         v
 5. API Server then checks RBAC rules (note 6) to decide
    what admin-user is ALLOWED to do
```

This is exactly how tools like the Kubernetes Dashboard let you "log in" with a token — you paste this token into the dashboard's login screen, and every action the dashboard performs afterward is really the API Server treating it as `admin-user`.

## ⚠️ This is the older / legacy approach

Manually creating a `type: kubernetes.io/service-account-token` Secret like this used to be Kubernetes' default way of giving ServiceAccounts tokens (pre-1.24, Kubernetes auto-created one of these for every ServiceAccount). This has some downsides:

- The token is **long-lived** (it doesn't expire on its own) unless you manually delete/rotate the Secret.
- A long-lived token that leaks is a bigger risk, since it keeps working indefinitely.

## Modern Kubernetes prefers short-lived tokens

Since Kubernetes 1.24+, the recommended approach is:

- **TokenRequest API** / **projected service account tokens** — tokens that are **short-lived** (e.g. valid for 1 hour), automatically refreshed, and scoped to a specific audience (e.g. "only valid for talking to this specific service").

```
Old way:  Secret (type: service-account-token)  -> long-lived token, manual
New way:  TokenRequest API / projected volume    -> short-lived, auto-rotated
```

In a Pod spec, the modern approach looks like this instead of a manual Secret:

```yaml
volumes:
  - name: kube-api-access
    projected:
      sources:
        - serviceAccountToken:
            expirationSeconds: 3600
            path: token
```

You'll still see the manual Secret approach used deliberately in specific cases — like generating a long-lived admin token to paste into the Kubernetes Dashboard login screen for a demo or learning setup (exactly what the YAML at the top of this note is for) — but for real production workloads, the short-lived, auto-rotated token is the safer, modern default.

## Quick recap

- The token is generated automatically once you create a Secret of type `kubernetes.io/service-account-token` with the right annotation.
- The token proves identity; the annotation (not the name) links the Secret to its ServiceAccount.
- The API Server checks the token's signature to recognize identity, then checks RBAC to decide permissions.
- This manual style is legacy; modern Kubernetes prefers short-lived, auto-rotated tokens via the TokenRequest API.
