# Kubernetes Service Discovery — Complete Implementation (Server + Microservices + Gateway)

> **Version note:** Spring Cloud Kubernetes explicitly requires the Discovery Server image tag and the client dependency version to be aligned with each other and with your Spring Boot/Spring Cloud BOM version. The tag used below (`3.1.6`) matches Spring Cloud `2023.0.x` (Spring Boot 3.2.x). Check `https://mvnrepository.com/artifact/org.springframework.cloud/spring-cloud-kubernetes-discovery` for the exact version matching your project's Spring Cloud BOM before deploying.

---

## Part 0 — The Discovery Server (Kubernetes Manifest)

This is the piece that replaces Eureka Server as an actual running component — it's a Spring app maintained by the Spring Cloud Kubernetes project, shipped as a ready-to-use image on Docker Hub. It watches the Kubernetes API for Services/Endpoints and exposes that info over HTTP, the same role Eureka's dashboard/registry played.

```yaml
apiVersion: v1
kind: List
items:
  - apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: discoveryserver

  - apiVersion: rbac.authorization.k8s.io/v1
    kind: Role
    metadata:
      name: discoveryserver-role
      namespace: default
    rules:
      - apiGroups: ["", "extensions", "apps"]
        resources: ["pods", "services", "endpoints"]
        verbs: ["get", "list", "watch"]

  - apiVersion: rbac.authorization.k8s.io/v1
    kind: RoleBinding
    metadata:
      name: discoveryserver-binding
    roleRef:
      kind: Role
      apiGroup: rbac.authorization.k8s.io
      name: discoveryserver-role
    subjects:
      - kind: ServiceAccount
        name: discoveryserver

  - apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: discoveryserver-deployment
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: discoveryserver
      template:
        metadata:
          labels:
            app: discoveryserver
        spec:
          serviceAccountName: discoveryserver
          containers:
            - name: discoveryserver
              image: springcloud/spring-cloud-kubernetes-discoveryserver:3.1.6
              ports:
                - containerPort: 8761
              env:
                - name: SPRING_CLOUD_KUBERNETES_HTTP_DISCOVERY_CATALOG_WATCHER_ENABLED
                  value: "TRUE"

  - apiVersion: v1
    kind: Service
    metadata:
      name: discoveryserver
    spec:
      selector:
        app: discoveryserver
      ports:
        - name: http
          port: 8761
          targetPort: 8761
      type: ClusterIP
```

Apply it directly with `kubectl apply -f discoveryserver.yaml`, or wrap it as a chart the same way you did for every other service — either way, it results in a `discoveryserver` Service reachable at `http://discoveryserver:8761` from inside the cluster.

---

## Part 1 — In Each Microservice (`accounts`, `cards`, `loans`, etc.)

### 1. Dependency (`pom.xml`)
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-discoveryclient</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```
`spring-cloud-starter-kubernetes-discoveryclient` is the HTTP client that talks to the `discoveryserver` you just deployed — this is the piece that plays the same role the Eureka client dependency used to play.

### 2. `application.yml` Config
```yaml
spring:
  application:
    name: cards
  cloud:
    kubernetes:
      discovery:
        discoveryServerUrl: http://discoveryserver:8761
    loadbalancer:
      ribbon:
        enabled: false
```
`discoveryServerUrl` points at the Discovery Server's Service DNS name from Part 0. `spring.application.name` matters here — it should match the Kubernetes Service name of the microservice itself (e.g. `cards`), since that's the identity other services will look it up by.

### 3. Feign Client Config
Because the discovery client is active, you can go back to the Eureka-style logical name — no manual URL needed:
```java
@FeignClient(name = "cards")
public interface CardsFeignClient {

    @GetMapping(value = "/api/fetch", consumes = "application/json")
    ResponseEntity<CardsDto> fetchCardDetails(@RequestParam String mobileNumber);
}
```
```java
@FeignClient(name = "loans")
public interface LoansFeignClient {

    @GetMapping(value = "/api/fetch", consumes = "application/json")
    ResponseEntity<LoansDto> fetchLoanDetails(@RequestParam String mobileNumber);
}
```
`name` here must match the **exact Kubernetes Service name**, lowercase — e.g. `cards`, not `CARDS`. Spring Cloud LoadBalancer resolves it through the discovery client from Part 1.1/1.2, the same way Ribbon used to resolve it through Eureka.

### 4. Main Application Class
```java
@SpringBootApplication
@EnableFeignClients
@EnableDiscoveryClient
public class AccountsApplication {
    public static void main(String[] args) {
        SpringApplication.run(AccountsApplication.class, args);
    }
}
```
`@EnableDiscoveryClient` is back — it's generic across discovery implementations, so the same annotation you used for Eureka now activates the Kubernetes discovery client instead.

---

## Part 2 — In the Gateway (`gatewayserver`)

### 1. Dependency (`pom.xml`)
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-discoveryclient</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-loadbalancer</artifactId>
</dependency>
```

### 2. `application.yml` Config — exactly where `lb://` goes
```yaml
spring:
  application:
    name: gatewayserver
  cloud:
    kubernetes:
      discovery:
        discoveryServerUrl: http://discoveryserver:8761
    gateway:
      routes:
        - id: accounts
          uri: lb://accounts
          predicates:
            - Path=/eazybank/accounts/**
          filters:
            - RewritePath=/eazybank/accounts/(?<segment>.*), /$\{segment}

        - id: cards
          uri: lb://cards
          predicates:
            - Path=/eazybank/cards/**
          filters:
            - RewritePath=/eazybank/cards/(?<segment>.*), /$\{segment}

        - id: loans
          uri: lb://loans
          predicates:
            - Path=/eazybank/loans/**
          filters:
            - RewritePath=/eazybank/loans/(?<segment>.*), /$\{segment}
```
This is the exact spot you asked about: `uri: lb://accounts` / `lb://cards` / `lb://loans` — same `lb://` scheme you used with Eureka, but the name after it must now match the **Kubernetes Service name exactly, lowercase** (`accounts`, `cards`, `loans`), since resolution goes through the Kubernetes discovery client against real Service objects instead of Eureka's registry.

### 3. Main Application Class
```java
@SpringBootApplication
@EnableDiscoveryClient
public class GatewayserverApplication {
    public static void main(String[] args) {
        SpringApplication.run(GatewayserverApplication.class, args);
    }
}
```
