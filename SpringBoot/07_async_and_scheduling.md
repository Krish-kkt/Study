# 07 — Async & Scheduling

---

## 7.1 Why Async?

A synchronous method occupies a thread for its entire duration. If you have 10 Tomcat threads and each request triggers a long email-sending operation (500ms), your server can only serve 10 simultaneous requests before new ones queue up.

```
Synchronous flow (1 request, 1 thread blocked for 800ms):
  Thread-1: [Controller] → [Service] → [EmailSender 500ms] → [Return response]
             ← 800ms total held ─────────────────────────────────────────────→

Async flow (response returns in 10ms, email sent in background):
  Thread-1: [Controller] → [Service] → submit to pool → [Return response immediately]
  Thread-2 (pool):                      [EmailSender 500ms] ← separate thread
```

---

## 7.2 `@Async` — Run Methods in Background Threads

### Setup

```java
@SpringBootApplication
@EnableAsync  // required — activates async processing
public class Application { ... }
```

### Basic Usage

```java
@Service
public class NotificationService {

    @Async  // Spring runs this in a separate thread from the caller
    public void sendWelcomeEmail(String email, String name) {
        // This runs in a thread pool — caller doesn't wait
        emailClient.send(email, "Welcome " + name);
    }
}

// Caller:
@Service
public class UserService {
    private final NotificationService notificationService;

    public UserResponse createUser(CreateUserRequest request) {
        User user = userRepository.save(userMapper.toEntity(request));
        notificationService.sendWelcomeEmail(user.getEmail(), user.getName());
        // returns immediately — email is sent in background
        return userMapper.toResponse(user);
    }
}
```

### Return a `CompletableFuture` to Get the Result Later

```java
@Service
public class ReportService {

    @Async
    public CompletableFuture<Report> generateReport(Long userId) {
        // runs in background thread
        Report report = heavyReportGenerator.generate(userId);
        return CompletableFuture.completedFuture(report);
    }
}

// Caller — fire multiple async tasks in parallel:
@Service
public class DashboardService {
    private final ReportService reportService;

    public DashboardData getDashboard(Long userId) throws Exception {
        CompletableFuture<Report> salesFuture = reportService.generateReport(userId);
        CompletableFuture<Report> inventoryFuture = reportService.generateInventoryReport(userId);

        // Wait for both to complete
        CompletableFuture.allOf(salesFuture, inventoryFuture).join();

        return new DashboardData(salesFuture.get(), inventoryFuture.get());
    }
}
```

---

## 7.3 Configuring the Thread Pool

**Never use the default thread pool in production.** The default `SimpleAsyncTaskExecutor` creates a new thread for every async call — no pooling, no limits — which can exhaust system resources.

```java
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public Executor getAsyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);       // always-alive threads
        executor.setMaxPoolSize(10);       // max threads under load
        executor.setQueueCapacity(100);    // queue before rejecting
        executor.setThreadNamePrefix("async-");  // visible in thread dumps
        executor.setWaitForTasksToCompleteOnShutdown(true);
        executor.setAwaitTerminationSeconds(30);  // graceful shutdown
        executor.initialize();
        return executor;
    }

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("Async method '{}' threw exception: {}", method.getName(), ex.getMessage(), ex);
        };
    }
}
```

### Multiple Executors for Different Task Types

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("emailExecutor")
    public Executor emailExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(3);
        exec.setMaxPoolSize(5);
        exec.setThreadNamePrefix("email-");
        exec.initialize();
        return exec;
    }

    @Bean("reportExecutor")
    public Executor reportExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(2);
        exec.setMaxPoolSize(4);
        exec.setThreadNamePrefix("report-");
        exec.initialize();
        return exec;
    }
}

// Use a specific executor by name:
@Async("emailExecutor")
public void sendEmail(String to, String body) { ... }

@Async("reportExecutor")
public CompletableFuture<Report> generateReport(Long id) { ... }
```

---

## 7.4 `@Async` Pitfalls

### Pitfall 1: Self-Invocation (Same as AOP)

```java
@Service
public class OrderService {

    public void placeOrder(Order order) {
        processPayment(order);  // @Async is IGNORED — bypasses proxy
    }

    @Async
    public void processPayment(Order order) { ... }
}
```

**Fix:** Extract `processPayment` to a separate bean.

### Pitfall 2: Return Type Must Be `void` or `Future`

```java
@Async
public String buildReport() {
    // WRONG — @Async on non-void/non-Future return value:
    // Spring fires the method async but the caller gets null, not the return value
    return heavyComputation();
}

// CORRECT:
@Async
public CompletableFuture<String> buildReport() {
    return CompletableFuture.completedFuture(heavyComputation());
}
```

### Pitfall 3: Exception Handling for void Methods

Exceptions from `void` `@Async` methods are silently swallowed unless you configure `AsyncUncaughtExceptionHandler`:

```java
// Without handler — exception disappears:
@Async
public void sendEmail(String to) {
    throw new RuntimeException("SMTP failed");  // no one sees this
}

// With handler configured in AsyncConfig:
@Override
public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
    return (ex, method, params) -> {
        log.error("Async failure in {}: {}", method.getName(), ex.getMessage());
        alertService.sendAlert(ex);
    };
}
```

### Pitfall 4: Transaction Context Not Propagated

`@Transactional` and `@Async` don't mix — a new thread gets a new transaction context (or no transaction):

```java
@Transactional
public void createOrder(Order order) {
    orderRepository.save(order);
    processAsync(order);  // runs in new thread — different transaction
}

@Async
public void processAsync(Order order) {
    // orderRepository.findById(order.getId()) might return nothing
    // if the outer transaction hasn't committed yet!
}
```

**Fix:** Ensure the async method runs after the outer transaction commits, or pass already-committed IDs.

---

## 7.5 `@Scheduled` — Recurring Background Jobs

### Setup

```java
@SpringBootApplication
@EnableScheduling  // required
public class Application { ... }
```

### Types of Scheduling

```java
@Component
public class ScheduledJobs {

    // 1. Fixed Rate — fires every 5 seconds from the START of the last execution
    // If the job takes 3s and rate is 5s: fires at 0s, 5s, 10s, 15s...
    @Scheduled(fixedRate = 5000)
    public void syncWithExternalApi() {
        externalApiClient.sync();
    }

    // 2. Fixed Delay — fires 5 seconds after the END of the last execution
    // If the job takes 3s and delay is 5s: fires at 0s, then 8s, then 16s...
    @Scheduled(fixedDelay = 5000)
    public void processPendingEmails() {
        emailQueue.processAll();
    }

    // 3. Cron — precise scheduling (second minute hour dayOfMonth month dayOfWeek)
    @Scheduled(cron = "0 0 2 * * ?")          // Every day at 2:00 AM
    public void generateDailyReport() { ... }

    @Scheduled(cron = "0 */15 9-17 * * MON-FRI")  // Every 15min, 9am-5pm, weekdays
    public void pollTransactions() { ... }

    // 4. Initial Delay — waits before first execution
    @Scheduled(fixedRate = 60000, initialDelay = 30000)
    public void pollWithStartDelay() { ... }

    // 5. Property-driven scheduling (configurable without code change)
    @Scheduled(cron = "${jobs.sync.cron:0 0 * * * ?}")
    public void configurableJob() { ... }
}
```

### Cron Expression Format

```
┌──────── second (0-59)
│  ┌─────── minute (0-59)
│  │  ┌──── hour (0-23)
│  │  │  ┌─ day of month (1-31)
│  │  │  │  ┌ month (1-12)
│  │  │  │  │  ┌ day of week (0-7, both 0 and 7 = Sunday)
│  │  │  │  │  │
0  0  2  *  *  ?     ← every day at 2:00 AM
0  0  9  *  *  MON   ← every Monday at 9:00 AM
0  */5 * *  *  ?     ← every 5 minutes
0  0  0  1  *  ?     ← 1st of every month at midnight
```

---

## 7.6 `@Scheduled` Pitfalls

### Pitfall 1: All Jobs Run in a Single Thread by Default

Spring uses a single-threaded scheduler by default. A long-running job blocks the next scheduled job:

```java
@Scheduled(fixedRate = 1000)     // every 1 second
public void fastJob() { }        // takes 200ms

@Scheduled(fixedRate = 1000)     // every 1 second
public void slowJob() {          // takes 5 seconds
    Thread.sleep(5000);
}
// Result: fastJob is delayed because both share the same thread
```

**Fix:** Configure a multi-threaded scheduler:

```java
@Configuration
@EnableScheduling
public class SchedulingConfig implements SchedulingConfigurer {

    @Override
    public void configureTasks(ScheduledTaskRegistrar registrar) {
        ThreadPoolTaskScheduler scheduler = new ThreadPoolTaskScheduler();
        scheduler.setPoolSize(5);
        scheduler.setThreadNamePrefix("scheduled-");
        scheduler.initialize();
        registrar.setTaskScheduler(scheduler);
    }
}
```

### Pitfall 2: Multiple Instances Run the Same Job

In a clustered environment (3 pods), `@Scheduled` runs on all 3 pods simultaneously. This causes duplicate processing, double emails, etc.

**Fix:** Use ShedLock:
```xml
<dependency>
    <groupId>net.javacrumbs.shedlock</groupId>
    <artifactId>shedlock-spring</artifactId>
</dependency>
```

```java
@Scheduled(cron = "0 0 2 * * ?")
@SchedulerLock(name = "generateDailyReport", lockAtLeastFor = "PT1M", lockAtMostFor = "PT10M")
public void generateDailyReport() {
    // Only ONE pod runs this at a time — others see the lock and skip
}
```

---

## 7.7 `@Async` vs `@Scheduled` vs Events

| | `@Async` | `@Scheduled` | `@EventListener` |
|---|---|---|---|
| Trigger | Method call | Timer/cron | Application event |
| Caller | Your code | Spring scheduler | Any Spring code via `ApplicationEventPublisher` |
| Background | Yes (in thread pool) | Yes (scheduler thread) | Optionally (add `@Async`) |
| Use for | Offload work from request thread | Periodic/recurring jobs | Decoupled reactions to domain events |

---

## Tricky Interview Questions

**Q: You have `@Async` on a method. The caller method is also `@Transactional`. Does the async method run inside the same transaction?**

No. `@Async` runs in a different thread, and Spring transactions are bound to the current thread via `ThreadLocal`. The async method runs without any transaction context (or starts its own if annotated with `@Transactional`).

**Implication:** If the caller's transaction hasn't committed yet when the async method fires, the async method might read the uncommitted (invisible) data.

---

**Q: What is the difference between `fixedRate` and `fixedDelay`?**

- `fixedRate = 5000`: fires every 5 seconds from the **start** of the last run. If the job takes 6 seconds, the next run starts immediately after completion (they don't overlap by default — the scheduler waits for the current run to finish before counting the rate).
- `fixedDelay = 5000`: waits 5 seconds after the **end** of the last run. If the job takes 6 seconds, total period is 11 seconds.

Use `fixedRate` for regular polling where the interval is measured from task start. Use `fixedDelay` when you want a fixed quiet period between runs.

---

**Q: How do you stop a scheduled task programmatically?**

```java
@Component
public class ControlledJob {

    private ScheduledFuture<?> scheduledFuture;

    @Autowired
    private TaskScheduler taskScheduler;

    public void startJob() {
        scheduledFuture = taskScheduler.scheduleAtFixedRate(
            this::doWork, Duration.ofSeconds(5));
    }

    public void stopJob() {
        if (scheduledFuture != null) {
            scheduledFuture.cancel(false);  // false = don't interrupt if running
        }
    }

    private void doWork() { ... }
}
```

---

**Q: You need to send 10,000 emails asynchronously. How do you ensure the app doesn't create 10,000 threads?**

Use a bounded thread pool with a queue:

```java
ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
executor.setCorePoolSize(5);
executor.setMaxPoolSize(10);
executor.setQueueCapacity(500);  // up to 500 tasks queued before rejection
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
// CallerRunsPolicy: if queue full, run in caller's thread (applies backpressure)
```

For truly large volumes, use a message queue (Kafka, RabbitMQ) instead of in-memory async — it persists tasks, allows retries, and scales across multiple instances.
