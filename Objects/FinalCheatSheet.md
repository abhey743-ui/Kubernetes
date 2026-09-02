# 10. Final Cheat Sheet

## One-line summary table

| Object | One-line meaning |
|---|---|
| **ConfigMap** | Non-sensitive configuration (URLs, service names, feature flags) |
| **Secret** | Sensitive data/credentials (passwords, tokens, keys) — base64-encoded, not automatically encrypted |
| **ServiceAccount** | An identity for software (Pods, apps, dashboards) — the "WHO" |
| **Token** | A credential that proves a ServiceAccount's identity — the "PROOF" |
| **ClusterRole** | A definition of permissions — the "WHAT IS ALLOWED" |
| **ClusterRoleBinding** | Connects an identity to a permission set — the "WIRE" between WHO and WHAT |
| **Helm** | A packaging/templating/deployment tool — generates and installs Kubernetes YAML (ConfigMaps, Secrets, Deployments, etc.) |

## The full mental map

```
                         +-----------+
                         |   Helm    |   (a tool — generates the YAML below)
                         +-----+-----+
                               |
          -----------------------------------------------
          |               |               |              |
    +-----v-----+   +-----v-----+   +-----v------+  +-----v------+
    | ConfigMap  |   |  Secret    |   | ServiceAcct|  |  Deployment |
    | non-sens.  |   | sensitive  |   | identity   |  |  Service    |
    | config     |   | data       |   | (WHO)      |  |  etc.       |
    +------------+   +-----+------+   +-----+------+  +-------------+
                            |                 |
                    (special type)      (used by RBAC below)
                            |                 |
                      +-----v------+          |
                      | SA Token    |          |
                      | Secret      |          |
                      | (PROOF)     |          |
                      +-----+------+          |
                            |                 |
                            +--------+--------+
                                     |
                        sent as: Authorization: Bearer <token>
                                     |
                                     v
                          Kubernetes API Server
                                     |
                     1. AUTHENTICATION (verify token -> WHO)
                                     |
                                     v
                     2. AUTHORIZATION (check RBAC -> WHAT)
                                     |
                     looks up: ClusterRoleBinding
                          subjects -> ServiceAccount
                          roleRef  -> ClusterRole (e.g. cluster-admin)
                                     |
                                     v
                          Request ALLOWED or DENIED
```

## The identity + permission flow (memorize this)

```
ServiceAccount  ->  Token  ->  API Server  ->  identity recognized
                                                       |
                                                       v
                                              RBAC checked
                                                       |
                                                       v
                                        request allowed / denied
```

## The naming trap, in one line

> `metadata.name` only labels an object. Only `subjects[].name`, `roleRef.name`, and the Secret's `kubernetes.io/service-account.name` annotation actually create relationships.

## The ConfigMap/Secret split, in one line

> If it's a password, token, or key — it's a Secret. If it's a URL, service name, or flag — it's a ConfigMap. Never mix the two.

## The Helm split, in one line

> Helm is the tool that builds and installs your Kubernetes YAML (including ConfigMaps and Secrets); it is never itself the running application (Kafka, RabbitMQ, your microservices, etc.).

## Least privilege, in one line

> Never grant `cluster-admin` (or any broad role) unless the identity genuinely needs full cluster power — scope permissions down to exactly what's required.
