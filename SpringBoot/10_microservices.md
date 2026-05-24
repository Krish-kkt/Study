# 10 — Microservice-Relevant Spring Boot Topics

---

## 10.1 Service-to-Service Communication Overview

In a microservices architecture, services communicate with each other:

```
Order Service  →  Payment Service   (synchronous HTTP)
Order Service  →  Notification Kafka Topic  (async message)
API Gateway    →  All Services       (route + auth)
```

---

## 10.2 `RestTemplate` (Legacy but Widely Present)

`RestTemplate` is the synchronous blocking HTTP client. Each call occupies a thread until the response arrives.

```java
@Configuration
public class RestTemplateConfig {
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(10))
            .build();
    }
}

@Service
public class PaymentClient {
    private final RestTemplate restTemplate;
    private static final String PAYMENT_URL = "http://payment-service/api/payments";

    public PaymentClient(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    public PaymentResponse charge(ChargeRequest request) {
        return restTemplate.postForObject(PAYMENT_URL, request, PaymentResponse.class);
    }

    public PaymentResponse findById(Long id) {
        return restTemplate.getForObject(PAYMENT_URL + "/{id}", PaymentResponse.class, id);
    }

    public void refund(Long paymentId) {
        restTemplate.delete(PAYMENT_URL + "/{id}", paymentId);
    }

    // Full control with exchange()
    public ResponseEntity<PaymentResponse> chargeWithHeaders(ChargeRequest request) {
        HttpHeaders headers = new HttpHeaders();
        headers.set("X-Idempotency-Key", UUID.randomUUID().toString());
        HttpEntity<ChargeRequest> entity = new HttpEntity<>(request, headers);
        return restTemplate.exchange(PAYMENT_URL, HttpMethod.POST, entity, PaymentResponse.class);
    }
}
```

**Status:** `RestTemplate` is in maintenance mode since Spring 5. It still works but use `WebClient` for new code.

---

## 10.3 `WebClient` — Modern Non-Blocking HTTP Client

`WebClient` is non-blocking and reactive. You can use it in a regular (non-reactive) Spring MVC app too — just call `.block()` to get the result synchronously.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

```java
@Configuration
public class WebClientConfig {
    @Bean
    public WebClient paymentWebClient() {
        return WebClient.builder()
            .baseUrl("http://payment-service")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .codecs(c -> c.defaultCodecs().maxInMemorySize(1024 * 1024))  // 1MB buffer
            .build();
    }
}

@Service
public class PaymentClient {
    private final WebClient webClient;

    public PaymentClient(WebClient paymentWebClient) {
        this.webClient = paymentWebClient;
    }

    // Synchronous use in a non-reactive app
    public PaymentResponse charge(ChargeRequest request) {
        return webClient.post()
            .uri("/api/payments")
            .bodyValue(request)
            .retrieve()
            .onStatus(HttpStatus::is4xxClientError, response ->
                response.bodyToMono(String.class)
                    .map(body -> new PaymentException("Client error: " + body)))
            .onStatus(HttpStatus::is5xxServerError, response ->
                Mono.error(new PaymentServiceUnavailableException()))
            .bodyToMono(PaymentResponse.class)
            .block(Duration.ofSeconds(10));  // block the calling thread
    }

    // Non-blocking / async in a reactive app
    public Mono<PaymentResponse> chargeAsync(ChargeRequest request) {
        return webClient.post()
            .uri("/api/payments")
            .bodyValue(request)
            .retrieve()
            .bodyToMono(PaymentResponse.class);
    }
}
```

---

## 10.4 OpenFeign — Declarative HTTP Client

Feign generates an HTTP client from an interface — no implementation code needed:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableFeignClients  // required
public class Application { ... }

// Just an interface — Spring generates the implementation
@FeignClient(name = "payment-service", url = "${payment.service.url}")
public interface PaymentClient {

    @PostMapping("/api/payments")
    PaymentResponse charge(@RequestBody ChargeRequest request);

    @GetMapping("/api/payments/{id}")
    PaymentResponse findById(@PathVariable Long id);

    @DeleteMapping("/api/payments/{id}")
    void refund(@PathVariable Long id);
}

// Usage — inject and call like a regular service
@Service
public class OrderService {
    private final PaymentClient paymentClient;  // no implementation written

    public OrderResponse placeOrder(CreateOrderRequest request) {
        PaymentResponse payment = paymentClient.charge(new ChargeRequest(...));
        // ...
    }
}
```

### Feign Error Handling

```java
@Component
public class PaymentFeignErrorDecoder implements ErrorDecoder {
    @Override
    public Exception decode(String methodKey, Response response) {
        return switch (response.status()) {
            case 404 -> new PaymentNotFoundException("Payment not found");
            case 422 -> new InsufficientFundsException("Insufficient funds");
            default -> new FeignException.FeignClientException(
                response.status(), "Payment service error", null, null, null);
        };
    }
}
```

---

## 10.5 Resilience4j — Circuit Breaker and Fault Tolerance

In a microservice system, remote calls fail. Without fault tolerance, one slow downstream service can cascade and take down your entire service.

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>  <!-- required for annotations -->
</dependency>
```

### Circuit Breaker

```
CLOSED (normal — calls pass through)
  ↓ failure threshold exceeded (e.g. 50% of last 10 calls failed)
OPEN (circuit broken — calls rejected immediately without even trying)
  ↓ wait window expires (e.g. 60 seconds)
HALF_OPEN (probe — allow a few calls to test recovery)
  ↓ probes succeed → CLOSED
  ↓ probes fail → back to OPEN
```

```java
@Service
public class OrderService {

    @CircuitBreaker(name = "paymentService", fallbackMethod = "chargeWithFallback")
    public PaymentResponse charge(ChargeRequest request) {
        return paymentClient.charge(request);
    }

    // Fallback must have same parameters + exception parameter
    public PaymentResponse chargeWithFallback(ChargeRequest request, Exception ex) {
        log.warn("Payment service circuit open, using fallback. Error: {}", ex.getMessage());
        // Return a degraded response or throw a business exception
        return new PaymentResponse(null, PaymentStatus.PENDING, "Retry later");
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        sliding-window-size: 10          # evaluate last 10 calls
        failure-rate-threshold: 50       # open if 50%+ fail
        wait-duration-in-open-state: 60s # stay open for 60s
        permitted-number-of-calls-in-half-open-state: 3
        register-health-indicator: true  # visible in /actuator/health
```

### Retry

```java
@Retry(name = "paymentService", fallbackMethod = "chargeWithFallback")
public PaymentResponse charge(ChargeRequest request) {
    return paymentClient.charge(request);
}
```

```yaml
resilience4j:
  retry:
    instances:
      paymentService:
        max-attempts: 3
        wait-duration: 500ms
        exponential-backoff-multiplier: 2  # 500ms, 1000ms, 2000ms
        retry-exceptions:
          - java.net.ConnectException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.example.PaymentDeclinedException  # don't retry business errors
```

### Rate Limiter

```java
@RateLimiter(name = "externalApi", fallbackMethod = "rateLimitFallback")
public Data callExternalApi(String param) {
    return externalApiClient.fetch(param);
}
```

```yaml
resilience4j:
  ratelimiter:
    instances:
      externalApi:
        limit-for-period: 100      # max 100 calls
        limit-refresh-period: 1s   # per second
        timeout-duration: 100ms    # wait this long for a permit before fallback
```

### Bulkhead — Limit Concurrency

Prevents one downstream service from using all your threads:

```java
@Bulkhead(name = "paymentService")
public PaymentResponse charge(ChargeRequest request) { ... }
```

```yaml
resilience4j:
  bulkhead:
    instances:
      paymentService:
        max-concurrent-calls: 5    # max 5 concurrent calls to payment service
        max-wait-duration: 100ms   # wait this long for a slot before fallback
```

---

## 10.6 Distributed Tracing

In a microservices system, a single user request triggers calls across 5 services. Without tracing, you can't correlate logs across services.

### Spring Boot 3 + Micrometer Tracing

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-zipkin</artifactId>
</dependency>
```

```yaml
management:
  tracing:
    sampling:
      probability: 1.0  # 100% sampling in dev (use 0.1 in prod)
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

Spring automatically:
- Generates a `traceId` for incoming requests
- Propagates `traceId` via HTTP headers when using `WebClient`/`RestTemplate`
- Adds `traceId` and `spanId` to MDC, so they appear in every log line

```
Log output:
[order-service] [traceId=abc123, spanId=def456] Placing order for user 42
[payment-service] [traceId=abc123, spanId=789ghi] Charging payment for order 99
[notification-service] [traceId=abc123, spanId=jkl012] Sending confirmation to alice@test.com
```

All three logs share `traceId=abc123` — search in Zipkin/Grafana with this ID to see the full request flow.

---

## 10.7 API Gateway Pattern

```
Client
  ↓
API Gateway (Spring Cloud Gateway / Kong / AWS API Gateway)
  ↓
Route: /orders/** → Order Service
Route: /users/**  → User Service
Route: /payments/** → Payment Service
```

Spring Cloud Gateway example:

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service   # lb = load balanced via service discovery
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
            - name: CircuitBreaker
              args:
                name: orderService
                fallbackUri: forward:/fallback
```

---

## Tricky Interview Questions

**Q: What is the difference between `RestTemplate` and `WebClient`?**

| | `RestTemplate` | `WebClient` |
|---|---|---|
| Model | Synchronous, blocking | Non-blocking, reactive |
| Thread usage | 1 thread occupied per call | Thread is freed while waiting for I/O |
| Status | Maintenance mode (Spring 5+) | Actively developed |
| Use in | Spring MVC apps (legacy) | New apps, reactive apps |
| Result | Returns value directly | Returns `Mono<T>` / `Flux<T>` |

In a Spring MVC app, you can still use `WebClient` with `.block()` — you get better error handling and a more modern API without needing to go fully reactive.

---

**Q: What's the difference between Circuit Breaker and Retry? Can you use both?**

- **Retry**: tries the call again on failure. Good for transient errors (network blip). Bad if the service is overloaded — you make it worse.
- **Circuit Breaker**: after repeated failures, stops calling the service entirely for a window. Gives the downstream service time to recover. Good for sustained failures.

Use both together: retry first (for transient errors), circuit breaker wraps retries (opens if retries keep failing):

```java
@CircuitBreaker(name = "payment", fallbackMethod = "fallback")
@Retry(name = "payment")
public PaymentResponse charge(ChargeRequest req) { ... }
```

Order matters: Resilience4j applies in the order declared, inner → outer: retry runs inside circuit breaker.

---

**Q: Service A calls Service B. Service B is slow (10s response). How does this impact Service A?**

Without timeout or bulkhead, all of Service A's request threads get occupied waiting for Service B's response. Service A becomes unresponsive (thread pool exhaustion) even though Service A's own code is fine.

Fix:
1. Set connection/read timeout on the HTTP client
2. Add a bulkhead to limit concurrent calls to Service B
3. Add a circuit breaker to stop calling when B is consistently slow
4. Add a fallback that returns a degraded response when B is unavailable

---

**Q: What is idempotency and why does it matter for retry logic?**

An operation is **idempotent** if calling it multiple times produces the same result as calling it once. `GET /users/1` is idempotent. `POST /orders` (create order) is NOT — retrying creates duplicate orders.

For retried write operations, use an **idempotency key**:
```java
headers.set("X-Idempotency-Key", orderId.toString());
// Payment service: if this key was already processed, return the cached result
// Don't charge again
```

Always include idempotency keys in payment, email, and order creation APIs before adding retry logic.
