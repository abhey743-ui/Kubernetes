# 1. Kubernetes ConfigMap

## What is a ConfigMap?

A **ConfigMap** is a Kubernetes object that stores **non-sensitive configuration data** as simple key-value pairs.

Think of it like a settings file that lives *outside* your application, but Kubernetes can hand it to your application when it starts.

```
+----------------------+
|     ConfigMap        |
|  KEY1 = value1       |
|  KEY2 = value2       |
|  KEY3 = value3       |
+----------------------+
```

It is **not** a program, not a container, not a volume by itself — it's just a named bucket of key-value data sitting in the cluster, waiting to be used.

## Why do we use it?

Without a ConfigMap, you would normally "bake" configuration values directly into your Docker image (hardcoded in application.properties, application.yml, or environment variables set in the Dockerfile).

That causes real problems:

- If you change one config value, you must **rebuild the whole Docker image**.
- The **same image** can't be reused for DEV, TEST, and PROD, because the config is stuck inside it.
- Config and code get mixed together, which is messy and hard to manage.

A ConfigMap solves this by separating **configuration** from **code/image**.

## Why keep non-sensitive configuration outside the Docker image?

The core idea in modern app design (the "12-factor app" principle) is:

> **Build once, run anywhere.**

You build **one** Docker image, and then you configure it differently for each environment (DEV/TEST/PROD) by injecting different ConfigMaps — without touching the image at all.

```
   Same Docker Image
          |
   +------+------+------+
   |      |      |      |
  DEV    TEST   PROD   ...
   |      |      |      |
ConfigMap ConfigMap ConfigMap
 (dev)    (test)    (prod)
```

This makes deployments safer, faster, and far more predictable, since the exact image you tested is the exact image you ship.

## How does a Pod/Deployment consume a ConfigMap?

There are two common ways:

**1. As environment variables** (most common for Spring Boot-style apps):

```yaml
envFrom:
  - configMapRef:
      name: eazybank-configmap
```

This takes every key in the ConfigMap and turns it into an environment variable inside the container.

**2. As a mounted file (volume):**

```yaml
volumes:
  - name: config-volume
    configMap:
      name: eazybank-configmap
```

This creates actual files inside the container's filesystem, one file per key (or one file with all the data, depending on how you mount it). Apps that read config from a file (instead of env vars) use this method.

```
Deployment / Pod
  |
  |-- reads --> ConfigMap (env vars OR mounted files)
  |
  container starts with configuration already available
```

## A ConfigMap does NOT automatically belong to one microservice

This is an important and often misunderstood point:

A ConfigMap is just a **named object in a namespace**. Kubernetes does not know or care whether it "belongs" to one service or ten services. **Any Pod in the same namespace can reference any ConfigMap**, regardless of which team or microservice created it.

So "ownership" of a ConfigMap is a **convention you choose**, not something Kubernetes enforces.

## Common/shared config vs service-specific config

In a microservices system, you usually have two kinds of configuration:

| Type | Examples | Used by |
|---|---|---|
| **Shared/common config** | Eureka server URL, Config Server URL, Keycloak URL | Many/all microservices |
| **Service-specific config** | `ACCOUNTS_APPLICATION_NAME=accounts`, a service's own port or feature flag | Just one microservice |

A practical pattern is:

- One **shared ConfigMap** holding values every service needs (discovery URL, config server URL, security URLs).
- Optionally, **small service-specific ConfigMaps** (or plain environment variables in the Deployment) for values only one service cares about.

## How does a ConfigMap work across DEV / TEST / PROD?

The ConfigMap's **name can stay the same**, but its **content changes per environment**, usually by:

- Having separate YAML files per environment (`configmap-dev.yaml`, `configmap-prod.yaml`), applied to different namespaces or clusters, OR
- Using **Helm** to template one ConfigMap definition and inject different values per environment (see the next note, `02-helm-vs-configmap.md`).

```
dev namespace   -> ConfigMap "eazybank-configmap" (SPRING_PROFILES_ACTIVE=dev)
test namespace  -> ConfigMap "eazybank-configmap" (SPRING_PROFILES_ACTIVE=test)
prod namespace  -> ConfigMap "eazybank-configmap" (SPRING_PROFILES_ACTIVE=prod)
```

Same name, same keys, different values — that's the whole trick.

## Do we need one ConfigMap per microservice, and/or per profile (environment)?

Short answer: **it depends, but there's no hard Kubernetes rule.** Common real-world patterns:

1. **One shared ConfigMap per environment** — holds common values (Eureka URL, Config Server URL). Simple, used a lot in smaller systems (this matches your `eazybank-configmap` below).
2. **One ConfigMap per microservice per environment** — more granular, avoids one giant file, easier to control who can edit what, but means more objects to manage.
3. **A mix** — one shared ConfigMap for common values + one small ConfigMap (or plain env vars) per service for service-specific values.

## A practical professional approach

In real production systems, teams usually:

- Keep a **shared ConfigMap** for cross-cutting values (discovery, config server, security endpoints).
- Keep **service-specific settings** either in a small dedicated ConfigMap, or directly in that service's Helm `values.yaml`.
- **Never** put passwords, secrets, or tokens in a ConfigMap — those go in a Kubernetes **Secret** (see `03-kubernetes-secret.md`).
- Manage all of this through **Helm**, so different environments (dev/test/prod) reuse the same templates with different values files, instead of hand-editing YAML each time.

---

## Your example, explained line by line

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: eazybank-configmap
data:
  SPRING_PROFILES_ACTIVE: prod
  SPRING_CONFIG_IMPORT: "configserver:http://configserver:8071/"
  EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: "http://eurekaserver:8070/eureka/"
  CONFIGSERVER_APPLICATION_NAME: configserver
  EUREKA_APPLICATION_NAME: eurekaserver
  ACCOUNTS_APPLICATION_NAME: accounts
  LOANS_APPLICATION_NAME: loans
  CARDS_APPLICATION_NAME: cards
  GATEWAY_APPLICATION_NAME: gatewayserver
  KEYCLOAK_ADMIN: admin
  KEYCLOAK_ADMIN_PASSWORD: admin
  SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_JWK-SET-URI: "http://keycloak:7080/realms/eazybank/protocol/openid-connect/certs"
```

| Line | Meaning |
|---|---|
| `apiVersion: v1` | ConfigMaps are part of the core ("v1") Kubernetes API — no special API group needed. |
| `kind: ConfigMap` | Tells Kubernetes what type of object this is. |
| `metadata.name: eazybank-configmap` | The name other objects (Pods, Deployments) will use to reference this ConfigMap. This is a **shared** ConfigMap — its name doesn't say "accounts" or "loans", it's generic for the whole `eazybank` system. |
| `data:` | The actual key-value configuration. Every line under here is one setting. |
| `SPRING_PROFILES_ACTIVE: prod` | Tells every Spring Boot service which profile to run with. This is what makes this ConfigMap "the PROD version" — in DEV you'd change this single value to `dev`. |
| `SPRING_CONFIG_IMPORT` | Tells each service to pull its detailed configuration from a central **Spring Cloud Config Server**, reachable inside the cluster at `configserver:8071`. |
| `EUREKA_CLIENT_SERVICEURL_DEFAULTZONE` | The address of the **Eureka service discovery server**, so every microservice can register itself and find the others. |
| `CONFIGSERVER_APPLICATION_NAME`, `EUREKA_APPLICATION_NAME`, `ACCOUNTS_APPLICATION_NAME`, `LOANS_APPLICATION_NAME`, `CARDS_APPLICATION_NAME`, `GATEWAY_APPLICATION_NAME` | These give each microservice a **logical name** used for service discovery/registration (Eureka) and internal routing (Gateway). Notice this ConfigMap knows the names of *all* services — that's exactly what a "shared" ConfigMap looks like. |
| `KEYCLOAK_ADMIN` / `KEYCLOAK_ADMIN_PASSWORD` | Admin login for **Keycloak** (the identity/auth server). |
| `SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_JWK-SET-URI` | Tells services where to fetch the public keys used to **validate JWT tokens** issued by Keycloak. |

### ⚠️ Something worth flagging

`KEYCLOAK_ADMIN` and `KEYCLOAK_ADMIN_PASSWORD` are **credentials**, sitting inside a **ConfigMap**. ConfigMaps are not encrypted or access-restricted the way Secrets are — anyone who can read this ConfigMap can read the admin password in plain text. In a real production setup, these two values should be moved into a **Kubernetes Secret** instead (see `03-kubernetes-secret.md` for exactly why).

### How this ConfigMap is likely used

```
eazybank-configmap (shared, one per environment)
        |
        |---- read by ---- accounts Deployment
        |---- read by ---- loans Deployment
        |---- read by ---- cards Deployment
        |---- read by ---- gatewayserver Deployment
        |---- read by ---- eurekaserver Deployment
        |---- read by ---- configserver Deployment
```

Every microservice Deployment would reference this same ConfigMap (via `envFrom.configMapRef`), so they all agree on where Eureka, Config Server, and Keycloak live — one source of truth, one place to update.
