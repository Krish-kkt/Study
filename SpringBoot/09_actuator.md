# 09 — Spring Boot Actuator

---

## 9.1 What Actuator Is For

In production, you need to answer questions like:
- Is the service healthy? Is the DB connection alive?
- How much memory is the JVM using?
- What environment properties are active?
- Which HTTP endpoints are being called most?
- Is the app connected to Redis / Kafka?

Actuator exposes these as HTTP endpoints (or JMX) without writing any code.

### Dependency

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

---

## 9.2 Key Endpoints

| Endpoint | URL | What it shows |
|---|---|---|
| Health | `/actuator/health` | App health status (UP/DOWN), DB, disk, Redis |
| Info | `/actuator/info` | App metadata, git commit, version |
| Metrics | `/actuator/metrics` | JVM memory, CPU, HTTP request counts |
| Env | `/actuator/env` | All resolved properties and their sources |
| Beans | `/actuator/beans` | All beans registered in the context |
| Mappings | `/actuator/mappings` | All `@RequestMapping` routes |
| Thread Dump | `/actuator/threaddump` | Current thread states (debug deadlocks) |
| Heap Dump | `/actuator/heapdump` | Download JVM heap dump (debug OOM) |
| Loggers | `/actuator/loggers` | View/change log levels at runtime |
| HTTP Trace | `/actuator/httptrace` | Last N HTTP requests (Spring Boot 2) |
| Cache | `/actuator/caches` | Cache names and statistics |
| Conditions | `/actuator/conditions` | Auto-config positive/negative matches |
| Shutdown | `/actuator/shutdown` | Graceful shutdown (POST, disabled by default) |

---

## 9.3 Configuring Exposed Endpoints

By default, only `health` and `info` are exposed over HTTP:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "health,info,metrics,loggers"   # expose only these
        # include: "*"                           # expose all (dev only — not prod)
        exclude: "heapdump,shutdown"             # never expose these
  endpoint:
    health:
      show-details: always           # show component details (DB, disk, etc.)
      show-components: always
    shutdown:
      enabled: true                  # enable shutdown endpoint (disabled by default)
  server:
    port: 8081                       # separate management port (keep 8080 for app)
```

**Best practice:** expose actuator on a separate management port (`8081`) that is only accessible internally — never exposed to the internet.

---

## 9.4 Health Endpoint Deep Dive

The `/actuator/health` endpoint aggregates health from multiple **health indicators**:

```json
// Response with show-details: always
{
  "status": "UP",
  "components": {
    "db": {
      "status": "UP",
      "details": {
        "database": "PostgreSQL",
        "validationQuery": "isValid()"
      }
    },
    "diskSpace": {
      "status": "UP",
      "details": {
        "total": 499963174912,
        "free": 291367510016,
        "threshold": 10485760
      }
    },
    "redis": {
      "status": "UP",
      "details": {
        "version": "7.0.11"
      }
    }
  }
}
```

Spring Boot auto-configures health indicators for: DataSource, Redis, RabbitMQ, Kafka, Elasticsearch, and more — just by having them on the classpath.

---

## 9.5 Custom Health Indicator

```java
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final PaymentGatewayClient gatewayClient;

    public PaymentGatewayHealthIndicator(PaymentGatewayClient gatewayClient) {
        this.gatewayClient = gatewayClient;
    }

    @Override
    public Health health() {
        try {
            GatewayStatus status = gatewayClient.ping();

            if (status == GatewayStatus.OK) {
                return Health.up()
                    .withDetail("gateway", "Stripe")
                    .withDetail("latencyMs", status.getLatency())
                    .build();
            } else {
                return Health.down()
                    .withDetail("gateway", "Stripe")
                    .withDetail("reason", "Non-OK status: " + status)
                    .build();
            }
        } catch (Exception ex) {
            return Health.down()
                .withDetail("gateway", "Stripe")
                .withDetail("error", ex.getMessage())
                .build();
        }
    }
}
```

The component name in the health response is derived from the class name (minus `HealthIndicator`): `paymentGateway`.

### Liveness vs Readiness Probes (Kubernetes)

Spring Boot 2.3+ has built-in support for Kubernetes probes:

```yaml
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

| Probe | Endpoint | Meaning |
|---|---|---|
| **Liveness** | `/actuator/health/liveness` | Is the app running? If DOWN → restart the pod |
| **Readiness** | `/actuator/health/readiness` | Is the app ready to serve traffic? If DOWN → remove from load balancer |

```java
// Programmatically control readiness (e.g., during warm-up):
@Autowired
private ApplicationContext context;

public void warmUp() {
    // do heavy initialization...
    AvailabilityChangeEvent.publish(context, ReadinessState.ACCEPTING_TRAFFIC);
}
```

---

## 9.6 Metrics Endpoint

```
GET /actuator/metrics
→ Lists all metric names

GET /actuator/metrics/jvm.memory.used
→ Details for a specific metric

GET /actuator/metrics/http.server.requests?tag=uri:/api/users&tag=status:200
→ Filtered metrics
```

Spring Boot 3 uses **Micrometer** as the metrics facade. It publishes to backends like:
- Prometheus (most common in cloud)
- Datadog, New Relic
- Graphite, InfluxDB

### Prometheus Integration

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
management:
  endpoints:
    web:
      exposure:
        include: "prometheus"
```

Prometheus scrapes `/actuator/prometheus` every 15s and your service auto-publishes:
- JVM: GC pause time, heap/non-heap memory, thread count
- HTTP: request count, duration, status codes per endpoint
- HikariCP: pool size, connection acquisition time

### Custom Metrics

```java
@Service
public class OrderService {

    private final Counter orderCounter;
    private final Timer orderProcessingTimer;

    public OrderService(MeterRegistry registry) {
        // Counter — monotonically increasing value
        this.orderCounter = Counter.builder("orders.created")
            .tag("service", "order-service")
            .description("Number of orders created")
            .register(registry);

        // Timer — measures duration + count
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Time to process an order")
            .register(registry);
    }

    public OrderResponse createOrder(CreateOrderRequest request) {
        return orderProcessingTimer.record(() -> {
            Order order = processOrder(request);
            orderCounter.increment();
            return orderMapper.toResponse(order);
        });
    }
}
```

---

## 9.7 Loggers Endpoint — Change Log Level at Runtime

```bash
# View current log levels
GET /actuator/loggers/com.example.service

# Change log level at runtime (no restart needed)
POST /actuator/loggers/com.example.service
Content-Type: application/json
{"configuredLevel": "DEBUG"}

# Restore to default
POST /actuator/loggers/com.example.service
{"configuredLevel": null}
```

This is invaluable in production — you can turn on DEBUG logging for a specific package to diagnose an issue, then turn it off without restarting.

---

## 9.8 Info Endpoint

```yaml
# application.yml
info:
  app:
    name: Order Service
    version: 2.1.0
    description: Handles order lifecycle
  team:
    name: Backend
    contact: backend@example.com
```

For git commit info, add the git plugin:

```xml
<plugin>
    <groupId>io.github.git-commit-id</groupId>
    <artifactId>git-commit-id-maven-plugin</artifactId>
</plugin>
```

```yaml
management:
  info:
    git:
      mode: full   # include branch, commit hash, commit time
```

Response:
```json
{
  "app": { "name": "Order Service", "version": "2.1.0" },
  "git": {
    "branch": "main",
    "commit": { "id": "a1b2c3d", "time": "2024-01-15T10:00:00Z" }
  }
}
```

---

## Tricky Interview Questions

**Q: Actuator exposes sensitive data (environment properties, DB credentials). How do you secure it in production?**

Three approaches:

1. **Separate management port** — expose actuator only on an internal port (8081), never on the public-facing port (8080)
2. **Spring Security** — secure actuator endpoints with roles:
   ```java
   http.requestMatcher(EndpointRequest.toAnyEndpoint())
       .authorizeRequests()
       .requestMatchers(EndpointRequest.to("health", "info")).permitAll()
       .anyRequest().hasRole("ACTUATOR_ADMIN");
   ```
3. **Selective exposure** — only expose endpoints that are needed; exclude sensitive ones

---

**Q: The `health` endpoint shows `DOWN`. Does that mean the app stops serving requests?**

Not automatically. `DOWN` just means the health check failed — your app continues running. However, in Kubernetes, if the readiness probe hits `/actuator/health/readiness` and gets `DOWN`, Kubernetes removes the pod from the load balancer. Liveness probe failure triggers a pod restart.

---

**Q: How does Actuator know the DB is healthy without you writing any code?**

Spring Boot's auto-configuration registers a `DataSourceHealthIndicator` when a `DataSource` bean is present (via `DataSourceAutoConfiguration`). It periodically executes a validation query (`SELECT 1`) against the datasource. If the query succeeds, `status = UP`. If it throws an exception, `status = DOWN`.

You get this for free — no code needed — just by having `spring-boot-starter-data-jpa` and a datasource configured.

---

**Q: How do you add a custom metric that tracks the number of currently active users?**

Use a `Gauge` (not Counter) — it tracks an instantaneous value that can go up and down:

```java
@Bean
public MeterBinder activeUserGauge(SessionRegistry sessionRegistry) {
    return registry -> Gauge.builder("users.active", sessionRegistry, sr ->
        (double) sr.getAllPrincipals().stream()
            .filter(p -> !sessionRegistry.getAllSessions(p, false).isEmpty())
            .count())
        .description("Currently active users")
        .register(registry);
}
```

`Counter` for values that only increase (requests, errors). `Gauge` for current state (queue size, active connections, active users).
