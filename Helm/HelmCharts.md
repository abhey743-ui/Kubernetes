# Your Helm Architecture — Step by Step (3 Layers)

You build this bottom-up, in 3 steps. Each step is one folder. Nothing in Step 1 or Step 2 needs to "know about" the layer above it — the layer above just plugs values in.

```
Step 1: eazybank-common   →  generic, reusable Kubernetes YAML "templates" (functions)
Step 2: accounts / cards / loans / ...  →  one folder per microservice, local settings
Step 3: dev-env   →  the umbrella chart you actually install; defines global values
                       AND is where the shared ConfigMap gets created (only once)
```

---

## STEP 1 — Generic Layer (`eazybank-common`)

**Purpose:** write the Kubernetes YAML shapes *once*, as reusable functions, so no microservice has to repeat Deployment/Service/ConfigMap boilerplate.

**Rule:** this chart never gets installed by itself. It only *exports* templates for others to call.

### File: `eazybank-common/templates/_helpers.tpl`

```yaml
{{- define "common.configmap" -}}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Values.global.configMapName }}
data:
  SPRING_PROFILES_ACTIVE: {{ .Values.global.activeProfile }}
  SPRING_CONFIG_IMPORT: {{ .Values.global.configServerURL }}
  EUREKA_CLIENT_SERVICEURL_DEFAULTZONE: {{ .Values.global.eurekaServerURL }}
  SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_JWK-SET-URI: {{ .Values.global.keyCloakURL }}
  JAVA_TOOL_OPTIONS: {{ .Values.global.openTelemetryJavaAgent }}
  OTEL_EXPORTER_OTLP_ENDPOINT: {{ .Values.global.otelExporterEndPoint }}
  OTEL_METRICS_EXPORTER: {{ .Values.global.otelMetricsExporter }}
  OTEL_LOGS_EXPORTER: {{ .Values.global.otelLogsExporter }}
  SPRING_CLOUD_STREAM_KAFKA_BINDER_BROKERS: {{ .Values.global.kafkaBrokerURL }}
{{- end -}}
```
**What this is:** a function named `common.configmap`. It produces one ConfigMap, filled entirely from `.Values.global.*`. It renders **nothing** until something calls it. Notice it does not hardcode which microservice it belongs to — it's generic on purpose.

```yaml
{{- define "common.deployment" -}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Values.deploymentName }}
  labels:
    app: {{ .Values.appLabel }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Values.appLabel }}
  template:
    metadata:
      labels:
        app: {{ .Values.appLabel }}
    spec:
      containers:
      - name: {{ .Values.appLabel }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        ports:
        - containerPort: {{ .Values.containerPort }}
          protocol: TCP
        env:
        {{- if .Values.appname_enabled }}
        - name: SPRING_APPLICATION_NAME
          value: {{ .Values.appName }}
        {{- end }}
        {{- if .Values.profile_enabled }}
        - name: SPRING_PROFILES_ACTIVE
          valueFrom:
            configMapKeyRef:
              name: {{ .Values.global.configMapName }}
              key: SPRING_PROFILES_ACTIVE
        {{- end }}
        {{- if .Values.config_enabled }}
        - name: SPRING_CONFIG_IMPORT
          valueFrom:
            configMapKeyRef:
              name: {{ .Values.global.configMapName }}
              key: SPRING_CONFIG_IMPORT
        {{- end }}
        {{- if .Values.eureka_enabled }}
        - name: EUREKA_CLIENT_SERVICEURL_DEFAULTZONE
          valueFrom:
            configMapKeyRef:
              name: {{ .Values.global.configMapName }}
              key: EUREKA_CLIENT_SERVICEURL_DEFAULTZONE
        {{- end }}
        {{- if .Values.resouceserver_enabled }}
        - name: SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_JWK-SET-URI
          valueFrom:
            configMapKeyRef:
              name: {{ .Values.global.configMapName }}
              key: SPRING_SECURITY_OAUTH2_RESOURCESERVER_JWT_JWK-SET-URI
        {{- end }}
        {{- if .Values.otel_enabled }}
        - name: JAVA_TOOL_OPTIONS
          valueFrom:
            configMapKeyRef:
              name: {{ .Values.global.configMapName }}
              key: JAVA_TOOL_OPTIONS
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          valueFrom:
            configMapKeyRef:
              name: {{ .Values.global.configMapName }}
              key: OTEL_EXPORTER_OTLP_ENDPOINT
        - name: OTEL_METRICS_EXPORTER
          valueFrom:
            configMapKeyRef:
              name: {{ .Values.global.configMapName }}
              key: OTEL_METRICS_EXPORTER
        - name: OTEL_LOGS_EXPORTER
          valueFrom:
            configMapKeyRef:
              name: {{ .Values.global.configMapName }}
              key: OTEL_LOGS_EXPORTER
        - name: OTEL_SERVICE_NAME
          value: {{ .Values.appName }}
        {{- end }}
        {{- if .Values.kafka_enabled }}
        - name: SPRING_CLOUD_STREAM_KAFKA_BINDER_BROKERS
          valueFrom:
            configMapKeyRef:
              name: {{ .Values.global.configMapName }}
              key: SPRING_CLOUD_STREAM_KAFKA_BINDER_BROKERS
        {{- end }}
{{- end -}}
```
**What this is:** a function named `common.deployment`. It builds one Deployment. Everything specific to a microservice (`deploymentName`, `appLabel`, `image`, `containerPort`) is read from `.Values` — not hardcoded — so the exact same function produces a correct Deployment for `accounts`, `cards`, or `loans`, just with different inputs. The `if .Values.xxx_enabled` blocks are **feature switches**: each microservice decides which env vars it needs by flipping true/false in its own values — the function itself doesn't care which service is calling it.

```yaml
{{- define "common.service" -}}
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.serviceName }}
spec:
  selector:
    app: {{ .Values.appLabel }}
  type: {{ .Values.service.type }}
  ports:
    - name: http
      protocol: TCP
      port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
{{- end -}}
```
**What this is:** a function named `common.service`. Same idea — generic Service shape, filled in from whatever `.Values` it's called with.

**End of Step 1.** You now have 3 reusable functions. Nothing has been created in Kubernetes yet — you've only written the *recipe*, not baked anything.

---

## STEP 2 — Microservice-Specific Layer (e.g. `accounts`)

**Purpose:** for each microservice, provide the specific inputs (image, port, name, which features it needs) and call the Step 1 functions with those inputs.

### File: `accounts/Chart.yaml`
```yaml
apiVersion: v2
name: accounts
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.0.0"
dependencies:
  - name: eazybank-common
    version: 0.1.0
    repository: file://../../eazybank-common
```
**What this does:** declares "this chart depends on the library from Step 1." This line is what makes `common.deployment`, `common.service`, and `common.configmap` *available* to be called from inside `accounts`. Without this dependency line, `{{ template "common.deployment" . }}` wouldn't exist.

### File: `accounts/values.yaml`
```yaml
deploymentName: accounts-deployment
serviceName: accounts
appLabel: accounts
appName: accounts
replicaCount: 1
image:
  repository: eazybytes/accounts
  tag: s14
containerPort: 8080
service:
  type: ClusterIP
  port: 8080
  targetPort: 8080
appname_enabled: true
profile_enabled: true
config_enabled: true
eureka_enabled: true
resouceserver_enabled: false
otel_enabled: true
kafka_enabled: true
```
**What this does:** every value the `common.deployment` and `common.service` functions from Step 1 need, specific to `accounts`. This is **local to this chart only** — `cards` and `loans` have their own separate copy of this file, with their own image/name/toggles. Notice: **no `global:` key here** — that's intentional, it arrives from Step 3.

### File: `accounts/templates/deployment.yaml`
```yaml
{{- template "common.deployment" . -}}
```
### File: `accounts/templates/service.yaml`
```yaml
{{- template "common.service" . -}}
```
**What these do:** each is one line. They call the Step 1 function, passing `.` (meaning: "use everything currently available — my `values.yaml` plus whatever gets merged in from Step 3"). This is the entire content of the accounts chart's own templates — all the real logic lives in Step 1, this layer just supplies data.

> **Important — no `configmap.yaml` file here.** In your original design, `accounts` also called `{{ template "common.configmap" . }}`, which meant `accounts`, `cards`, `loans`, etc. each created their *own* copy of an identically-named ConfigMap — 8 duplicate objects. In the corrected design, the ConfigMap is created **once**, in Step 3, not here. `accounts` only *references* it (via `configMapKeyRef` inside `common.deployment`, which you already have).

**Repeat this whole Step 2 folder once per microservice** (`cards`, `loans`, `configserver`, `eurekaserver`, `gatewayserver`, `message`) — same 3 files, different `values.yaml` contents.

---

## STEP 3 — Top Layer / Environment Layer (`dev-env`)

**Purpose:** (1) declare which microservices belong to this environment, (2) define the environment-wide `global` values every microservice needs, and (3) create the shared ConfigMap exactly once, by calling `common.configmap` **here** instead of inside every microservice.

### File: `dev-env/Chart.yaml`
```yaml
apiVersion: v2
name: dev-env
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.0.0"
dependencies:
  - name: eazybank-common
    version: 0.1.0
    repository: file://../../eazybank-common
  - name: configserver
    version: 0.1.0
    repository: file://../../eazybank-services/configserver
  - name: eurekaserver
    version: 0.1.0
    repository: file://../../eazybank-services/eurekaserver
  - name: accounts
    version: 0.1.0
    repository: file://../../eazybank-services/accounts
  - name: cards
    version: 0.1.0
    repository: file://../../eazybank-services/cards
  - name: loans
    version: 0.1.0
    repository: file://../../eazybank-services/loans
  - name: gatewayserver
    version: 0.1.0
    repository: file://../../eazybank-services/gatewayserver
  - name: message
    version: 0.1.0
    repository: file://../../eazybank-services/message
```
**What this does:** lists every Step 2 microservice chart as a dependency, plus the Step 1 library chart (needed here too, since Step 3 will call `common.configmap` directly). This is the "which services make up this environment" list.

### File: `dev-env/values.yaml`
```yaml
global:
  configMapName: eazybankdev-configmap
  activeProfile: default
  configServerURL: configserver:http://configserver:8071/
  eurekaServerURL: http://eurekaserver:8070/eureka/
  keyCloakURL: http://keycloak.default.svc.cluster.local:80/realms/master/protocol/openid-connect/certs
  openTelemetryJavaAgent: "-javaagent:/app/libs/opentelemetry-javaagent-2.22.0.jar"
  otelExporterEndPoint: http://tempo.default.svc.cluster.local:4318
  otelMetricsExporter: none
  otelLogsExporter: none
  kafkaBrokerURL: kafka-controller-0.kafka-controller-headless.default.svc.cluster.local:9092
```
**What this does:** everything under the `global:` key gets **automatically copied into `.Values.global` of every dependency listed above** — that's a built-in Helm rule tied to the literal word `global`, not something you wire up yourself. This is the one file where all environment-wide URLs live. Change this file, and every microservice picks up the new Eureka/Config Server/Keycloak/Kafka address without touching any of their individual `values.yaml` files.

### File: `dev-env/templates/configmap.yaml`  ← **new — this is the fix**
```yaml
{{- template "common.configmap" . -}}
```
**What this does:** this is the *only* place `common.configmap` gets called. It runs with `dev-env`'s own context, so `.Values.global.*` here is exactly the block above — nothing merged from anywhere, since this is the top layer. This produces exactly **one** ConfigMap object named `eazybankdev-configmap`, containing all 9 keys.

---

## What Happens When You Run `helm install eazybank ./dev-env`

1. Helm reads `dev-env/values.yaml` → sees the `global:` block.
2. Helm copies that `global:` block into the `.Values.global` of **every** dependency: `accounts`, `cards`, `loans`, `configserver`, `eurekaserver`, `gatewayserver`, `message`.
3. Helm renders `dev-env/templates/configmap.yaml` → **one** ConfigMap: `eazybankdev-configmap`, filled with the 9 global values.
4. Helm renders each microservice's `deployment.yaml` / `service.yaml` → one Deployment + one Service per microservice, each with local settings (image, port, toggles) plus `configMapKeyRef` pointers into `eazybankdev-configmap` for whichever env vars that service's toggles turned on.
5. All of these rendered manifests (1 ConfigMap + N Deployments + N Services) get bundled and applied to the cluster **as one Helm release**.

```
Step 1 (eazybank-common)          →  supplies the 3 reusable functions
        ↓ called by
Step 2 (accounts, cards, loans…)  →  supplies local values, calls common.deployment / common.service
        ↑ global values flow down from
Step 3 (dev-env)                  →  supplies global values + calls common.configmap ONCE
```

Read bottom-to-top when building it (write the generic layer first, then each microservice, then the environment) — but read top-to-bottom when tracing *where a value comes from* (start at `dev-env`'s `global:` block, follow it down into whichever microservice and template you're debugging).
