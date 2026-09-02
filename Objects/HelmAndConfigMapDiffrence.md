# 2. Helm vs ConfigMap

## The most important point first

**Helm and ConfigMap are NOT alternatives.** They are not two competing ways to do the same thing. They solve two completely different problems, and in real projects they are used **together**.

```
ConfigMap  = a THING (a piece of data sitting in Kubernetes)
Helm       = a TOOL that can CREATE that thing (and many other things)
```

Comparing them directly is a bit like comparing "a form you fill out" (ConfigMap) to "the printer that produces the form" (Helm) — they're not the same kind of object.

## Helm: a packaging / templating / deployment mechanism

**Helm** is a tool for Kubernetes that lets you:

- Package a whole application's Kubernetes YAML (Deployments, Services, ConfigMaps, Secrets, etc.) into one bundle called a **chart**.
- Use **templates** with placeholders instead of hardcoded values.
- Install, upgrade, and remove that whole bundle with one command (`helm install`, `helm upgrade`, `helm uninstall`).
- Reuse the exact same chart across DEV, TEST, and PROD, just swapping out a values file.

Helm doesn't run in your cluster as a "thing" your app talks to — it's a tool you (or your CI/CD pipeline) run to **generate and apply** real Kubernetes YAML.

## ConfigMap: an actual Kubernetes resource

A **ConfigMap**, as covered in note 1, is a **real object that exists inside the cluster** — Kubernetes itself knows about it, stores it, and serves it to Pods.

```
   Helm (a tool you run)
        |
        | generates & applies YAML
        v
+---------------------------+
| Real Kubernetes objects   |
|  - ConfigMap              |
|  - Secret                 |
|  - Deployment             |
|  - Service                |
+---------------------------+
```

So a ConfigMap can exist **with or without Helm** — you can `kubectl apply -f configmap.yaml` directly, no Helm required. Helm is just a convenient, repeatable way to generate and manage that YAML (and a lot of other YAML) at the same time.

## How Helm generates/creates a ConfigMap

Instead of writing one fixed `configmap.yaml`, in a Helm chart you write a **template**, like:

```yaml
# templates/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Values.appName }}-configmap
data:
  SPRING_PROFILES_ACTIVE: {{ .Values.springProfile }}
  EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: {{ .Values.eurekaUrl | quote }}
```

The `{{ .Values.xxx }}` parts are placeholders. Helm fills them in using a separate file.

## values.yaml, templates, and environment-specific values

- **`templates/`** — the folder containing your YAML templates (Deployment, Service, ConfigMap, Secret, etc.) with `{{ }}` placeholders instead of fixed values.
- **`values.yaml`** — the default values used to fill in those placeholders.
- **Environment-specific values files** — e.g. `values-dev.yaml`, `values-prod.yaml` — override just the values that differ per environment.

```
templates/configmap.yaml   (the shape/structure — same for all environments)
        +
values-prod.yaml           (springProfile: prod, eurekaUrl: http://eurekaserver:8070/eureka/)
        =
Final ConfigMap YAML applied to the PROD cluster
```

You'd run something like:

```
helm install eazybank ./eazybank-chart -f values-prod.yaml
```

and Helm renders the templates using `values-prod.yaml`, then sends the final YAML to Kubernetes — creating the real ConfigMap (and everything else in the chart) in one shot.

## Helm can also create Deployment, Service, Secret, and more

A single Helm chart usually bundles **everything one application needs**:

```
eazybank-chart/
  templates/
    deployment.yaml   -> creates the Deployment (runs your app's containers)
    service.yaml       -> creates the Service (stable network access to the Pods)
    configmap.yaml      -> creates the ConfigMap (non-sensitive config)
    secret.yaml          -> creates the Secret (sensitive config)
    serviceaccount.yaml   -> creates the ServiceAccount (identity)
  values.yaml
```

One `helm install` can create your Deployment, Service, ConfigMap, and Secret all together, all wired to reference each other correctly — instead of you manually applying five separate YAML files in the right order.

## The Kafka / RabbitMQ Helm chart example

A very common real-world example: you don't want to hand-write all the Kubernetes YAML needed to run Kafka or RabbitMQ (StatefulSets, Services, persistent storage, configuration, security) — that's a lot of complex, easy-to-get-wrong YAML.

Instead, you install a **pre-built Helm chart** that someone else (e.g. Bitnami) already wrote and tested:

```
helm install my-kafka bitnami/kafka
helm install my-rabbitmq bitnami/rabbitmq
```

This one command creates all the underlying Kubernetes resources needed to run a working Kafka or RabbitMQ cluster.

## Important: Helm is NOT Kafka/RabbitMQ itself

This is the same confusion as "Helm vs ConfigMap," just with a different resource:

- **Kafka** and **RabbitMQ** are the actual messaging systems (the software that runs).
- **Helm** is just the delivery mechanism — the tool that packages and installs the Kubernetes YAML needed to run Kafka/RabbitMQ.

```
Helm chart "bitnami/kafka"
        |
        | helm install
        v
Real Kafka Pods, Services, StatefulSets running in your cluster
```

Once installed, Helm's job is basically done — Kafka itself runs as normal Kubernetes Pods, using whatever Deployment/StatefulSet/ConfigMap/Secret objects Helm generated for it. If you `helm uninstall` it, Helm removes those objects — but while Kafka is running, it's Kubernetes (not Helm) actually keeping it alive.

## Quick recap

| Question | Answer |
|---|---|
| Is Helm a replacement for ConfigMap? | No — Helm is a tool, ConfigMap is a resource. |
| Can a ConfigMap exist without Helm? | Yes, you can `kubectl apply` it directly. |
| What does Helm actually produce? | Rendered Kubernetes YAML (Deployments, Services, ConfigMaps, Secrets, etc.), applied to the cluster. |
| Is Helm the same as Kafka/RabbitMQ? | No — Helm just installs the Kubernetes resources that run Kafka/RabbitMQ. |
