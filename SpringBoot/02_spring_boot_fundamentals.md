# 02 — Spring Boot Fundamentals

---

## 2.1 What Spring Boot Adds Over Plain Spring

Plain Spring requires you to manually configure everything:

```java
// Plain Spring — you write all of this
@Configuration
@EnableWebMvc
public class AppConfig {
    @Bean
    public DataSource dataSource() { ... }      // configure manually
    @Bean
    public EntityManagerFactory emf() { ... }  // configure manually
    @Bean
    public TransactionManager tm() { ... }      // configure manually
    // ... 100 more lines
}
```

Spring Boot's promise: **if a library is on the classpath, configure it automatically with sensible defaults.**

Add `spring-boot-starter-data-jpa` and Spring Boot auto-configures:
- A `DataSource` (HikariCP)
- `EntityManagerFactory` (Hibernate)
- `JpaTransactionManager`
- Spring Data repositories

You only write configuration when you want to override the defaults.

---

## 2.2 `@SpringBootApplication`

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

`@SpringBootApplication` is a shortcut for three annotations:

```java
@Configuration          // This class can declare @Bean methods
@EnableAutoConfiguration // Trigger auto-configuration based on classpath
@ComponentScan          // Scan this package + subpackages for @Component classes
public @interface SpringBootApplication { ... }
```

### Why Main Class Location Matters

`@ComponentScan` scans from the package of the annotated class downward. If your main class is in `com.example`, all classes in `com.example.*` are found. If a service is in `com.util`, it won't be found.

```
com.example
├── Application.java         ✅ root — scans everything below
├── controller/
│   └── UserController.java  ✅ found
├── service/
│   └── UserService.java     ✅ found
└── repository/
    └── UserRepository.java  ✅ found

com.util
└── HelperService.java       ❌ NOT found — outside scan path
```

Fix: either move classes under `com.example` or add `@ComponentScan(basePackages = {"com.example", "com.util"})`.

---

## 2.3 Auto-Configuration Deep Dive

### How It Works

When Spring Boot starts, it reads a list of auto-configuration classes from:
- Boot 2.x: `META-INF/spring.factories` under key `org.springframework.boot.autoconfigure.EnableAutoConfiguration`
- Boot 3.x: `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

Each entry is a class like `DataSourceAutoConfiguration`, `JacksonAutoConfiguration`, etc.

### Conditional Loading

Auto-config classes use `@Conditional` annotations to decide whether to apply:

```java
@Configuration
@ConditionalOnClass(DataSource.class)           // only if DataSource is on classpath
@ConditionalOnMissingBean(DataSource.class)     // only if user hasn't defined their own
@ConditionalOnProperty(name = "spring.datasource.url")  // only if property is set
public class DataSourceAutoConfiguration {
    @Bean
    public DataSource dataSource(DataSourceProperties props) {
        return DataSourceBuilder.create()
            .url(props.getUrl())
            .username(props.getUsername())
            .build();
    }
}
```

**Key insight:** `@ConditionalOnMissingBean` means: "if the user already defined a bean of this type, back off." This is how you override auto-configuration — just define your own `@Bean`.

### Debugging Auto-Configuration

Run with `--debug` flag or set `logging.level.org.springframework.boot.autoconfigure=DEBUG`:

```
=========================
AUTO-CONFIGURATION REPORT
=========================

Positive matches (applied):
--------------------------
   DataSourceAutoConfiguration matched:
      - @ConditionalOnClass found required class 'javax.sql.DataSource' (OnClassCondition)

Negative matches (not applied):
-------------------------------
   MongoAutoConfiguration:
      - @ConditionalOnClass did not find required class 'com.mongodb.MongoClient'
```

### Disabling Auto-Configuration

```java
// Disable specific auto-configs
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    SecurityAutoConfiguration.class
})
public class Application { ... }
```

Or in `application.yml`:
```yaml
spring:
  autoconfigure:
    exclude:
      - org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

---

## 2.4 Starter Dependencies

Starters are POM files that pull in all dependencies you need for a feature:

| Starter | What it includes |
|---|---|
| `spring-boot-starter-web` | Spring MVC, Jackson, embedded Tomcat, validation |
| `spring-boot-starter-data-jpa` | Spring Data, Hibernate, HikariCP, JDBC |
| `spring-boot-starter-security` | Spring Security core |
| `spring-boot-starter-test` | JUnit 5, Mockito, AssertJ, MockMvc |
| `spring-boot-starter-cache` | Spring Cache abstraction |
| `spring-boot-starter-actuator` | Actuator endpoints |

Each starter triggers the corresponding auto-configuration.

---

## 2.5 Externalized Configuration

### Priority Order (Highest Wins)

1. Command-line args: `--server.port=9090`
2. `SPRING_APPLICATION_JSON` env variable (inline JSON)
3. OS environment variables: `SERVER_PORT=9090`
4. `application-{profile}.properties` inside JAR
5. `application.properties` inside JAR
6. `application-{profile}.properties` on classpath
7. `application.properties` on classpath

### `@Value` for Simple Values

```java
@Component
public class EmailService {
    @Value("${smtp.host}")
    private String smtpHost;

    @Value("${smtp.port:25}")  // 25 is the default if property not set
    private int smtpPort;

    @Value("${app.allowed-origins}")
    private List<String> allowedOrigins; // Spring splits comma-separated values
}
```

#### Full Example — Common `@Value` Patterns

```yaml
# application.yml
smtp:
  host: smtp.company.com
  port: 587

app:
  allowed-origins: https://app.company.com,https://admin.company.com
  max-upload-size: 10485760   # 10 MB in bytes
  feature-flags:
    new-checkout: true
  admin-emails: ops@company.com,dev@company.com
  greeting: "Hello, %s! Welcome to %s."
```

```java
@Component
public class EmailService {

    // Plain string — fails startup if smtp.host is missing in config
    @Value("${smtp.host}")
    private String smtpHost;

    // Primitive with default — uses 25 if smtp.port is not set
    @Value("${smtp.port:25}")
    private int smtpPort;

    // Comma-separated string → List automatically split by Spring
    @Value("${app.allowed-origins}")
    private List<String> allowedOrigins;

    // Long / numeric types work directly
    @Value("${app.max-upload-size:5242880}")  // default 5 MB
    private long maxUploadSizeBytes;

    // Boolean flag
    @Value("${app.feature-flags.new-checkout:false}")
    private boolean newCheckoutEnabled;

    // SpEL (Spring Expression Language) — evaluate an expression at inject time
    // Converts bytes → MB so the rest of the code works in MB
    @Value("#{${app.max-upload-size} / 1048576}")
    private long maxUploadSizeMb;

    // SpEL — split a comma string into a Set instead of a List
    @Value("#{'${app.admin-emails}'.split(',')}")
    private Set<String> adminEmails;

    // SpEL — use a formatted string (like String.format)
    @Value("#{T(java.lang.String).format('${app.greeting}', 'Alice', 'OrderService')}")
    private String welcomeMessage;

    public void sendEmail(String to, String subject) {
        if (!newCheckoutEnabled) {
            System.out.println("New checkout is off, skipping promo email");
            return;
        }
        System.out.printf("Connecting to %s:%d%n", smtpHost, smtpPort);
        System.out.printf("Max attachment: %d MB%n", maxUploadSizeMb);
        System.out.println("Allowed origins: " + allowedOrigins);
    }
}
```

#### Using `@Value` via Constructor (Testable Pattern)

Field injection with `@Value` works but makes testing painful — you can't set the field without starting Spring. The cleaner approach:

```java
@Component
public class EmailService {

    private final String smtpHost;
    private final int smtpPort;
    private final List<String> allowedOrigins;

    // Spring resolves @Value expressions on constructor parameters too
    public EmailService(
            @Value("${smtp.host}") String smtpHost,
            @Value("${smtp.port:25}") int smtpPort,
            @Value("${app.allowed-origins}") List<String> allowedOrigins) {
        this.smtpHost = smtpHost;
        this.smtpPort = smtpPort;
        this.allowedOrigins = allowedOrigins;
    }

    // Now you can unit test without Spring:
    // new EmailService("localhost", 25, List.of("https://test.com"))
}
```

#### `@Value` vs `@ConfigurationProperties` — When to Use Which

| Situation | Use |
|---|---|
| One or two simple values in a class | `@Value` |
| A group of related config (3+ fields) | `@ConfigurationProperties` |
| You need validation (`@NotBlank`, `@Min`) | `@ConfigurationProperties` |
| You want IDE autocomplete in `application.yml` | `@ConfigurationProperties` |
| You need a computed value (SpEL expression) | `@Value` with `#{}` |
| Map or nested object from config | `@ConfigurationProperties` |

> **Real-world rule of thumb:** if you find yourself writing more than two `@Value` fields in the same class, extract them into a `@ConfigurationProperties` class. It's easier to validate, document, and test.

### `@ConfigurationProperties` for Complex Config (Preferred)

Binds an entire prefix to a POJO — type-safe, IDE-autocomplete, and validatable:

```yaml
# application.yml
app:
  payment:
    provider: stripe
    timeout-seconds: 30
    retry-count: 3
    api-keys:
      stripe: sk_live_xxx
      paypal: PAYPAL_xxx
```

```java
@ConfigurationProperties(prefix = "app.payment")
@Validated
public class PaymentProperties {
    @NotBlank
    private String provider;

    @Min(1) @Max(120)
    private int timeoutSeconds;

    private int retryCount = 1; // default value

    private Map<String, String> apiKeys = new HashMap<>();

    // getters and setters (or use Lombok @Data)
}
```

```java
// Register it — two ways:
@Configuration
@EnableConfigurationProperties(PaymentProperties.class)
public class AppConfig { }

// Or simpler — add @Component directly to the properties class
@Component
@ConfigurationProperties(prefix = "app.payment")
public class PaymentProperties { ... }
```

Then inject it anywhere:
```java
@Service
public class PaymentService {
    private final PaymentProperties props;

    public PaymentService(PaymentProperties props) {
        this.props = props;
    }

    public void processPayment() {
        String key = props.getApiKeys().get(props.getProvider());
        // ...
    }
}
```

#### Full Example — Nested Properties

Real configs are rarely flat. Here's how nested YAML maps to nested Java classes:

```yaml
# application.yml
app:
  database:
    primary:
      url: jdbc:postgresql://prod-db:5432/orders
      pool-size: 20
    replica:
      url: jdbc:postgresql://replica-db:5432/orders
      pool-size: 10
  notifications:
    email:
      host: smtp.company.com
      port: 587
    sms:
      provider: twilio
      from-number: "+15551234567"
```

```java
// Each nested object is its own inner class (or standalone class)
@Component
@ConfigurationProperties(prefix = "app")
@Validated
public class AppProperties {

    private Database database = new Database();
    private Notifications notifications = new Notifications();

    // Spring needs getters AND setters to bind nested objects
    public Database getDatabase() { return database; }
    public void setDatabase(Database database) { this.database = database; }

    public Notifications getNotifications() { return notifications; }
    public void setNotifications(Notifications notifications) { this.notifications = notifications; }

    public static class Database {
        private DataSourceConfig primary;
        private DataSourceConfig replica;

        // getters + setters for both fields
        public DataSourceConfig getPrimary() { return primary; }
        public void setPrimary(DataSourceConfig primary) { this.primary = primary; }
        public DataSourceConfig getReplica() { return replica; }
        public void setReplica(DataSourceConfig replica) { this.replica = replica; }
    }

    public static class DataSourceConfig {
        @NotBlank private String url;
        @Min(1) private int poolSize;

        public String getUrl() { return url; }
        public void setUrl(String url) { this.url = url; }
        public int getPoolSize() { return poolSize; }
        public void setPoolSize(int poolSize) { this.poolSize = poolSize; }
    }

    public static class Notifications {
        private EmailConfig email;
        private SmsConfig sms;

        public EmailConfig getEmail() { return email; }
        public void setEmail(EmailConfig email) { this.email = email; }
        public SmsConfig getSms() { return sms; }
        public void setSms(SmsConfig sms) { this.sms = sms; }
    }

    public static class EmailConfig {
        private String host;
        private int port;

        public String getHost() { return host; }
        public void setHost(String host) { this.host = host; }
        public int getPort() { return port; }
        public void setPort(int port) { this.port = port; }
    }

    public static class SmsConfig {
        private String provider;
        private String fromNumber;

        public String getProvider() { return provider; }
        public void setProvider(String provider) { this.provider = provider; }
        public String getFromNumber() { return fromNumber; }
        public void setFromNumber(String fromNumber) { this.fromNumber = fromNumber; }
    }
}
```

> **Tip:** In real projects, replace all those getters/setters with Lombok's `@Data` or `@Getter @Setter` on each inner class. Spring Boot works with Lombok seamlessly.

#### Using the Properties Class Elsewhere

Once registered as a bean, inject `AppProperties` (or any `@ConfigurationProperties` class) the same way you inject any Spring bean — via constructor injection:

```java
// Scenario 1: Service that routes notifications
@Service
public class NotificationService {

    private final AppProperties appProps;

    public NotificationService(AppProperties appProps) {
        this.appProps = appProps;
    }

    public void sendEmail(String to, String body) {
        String host = appProps.getNotifications().getEmail().getHost();
        int port = appProps.getNotifications().getEmail().getPort();
        // use host + port to open an SMTP connection
        System.out.println("Sending via " + host + ":" + port);
    }

    public void sendSms(String to, String message) {
        String provider = appProps.getNotifications().getSms().getProvider();
        String from = appProps.getNotifications().getSms().getFromNumber();
        System.out.println("SMS via " + provider + " from " + from);
    }
}
```

```java
// Scenario 2: Repository that picks the right datasource
@Repository
public class OrderRepository {

    private final AppProperties appProps;

    public OrderRepository(AppProperties appProps) {
        this.appProps = appProps;
    }

    public Order findById(long id, boolean readOnly) {
        // Use replica for reads, primary for writes — URLs come from config
        String url = readOnly
            ? appProps.getDatabase().getReplica().getUrl()
            : appProps.getDatabase().getPrimary().getUrl();

        System.out.println("Connecting to: " + url);
        // ... actual DB logic
        return new Order();
    }
}
```

```java
// Scenario 3: A @Configuration class that creates beans from the properties
@Configuration
public class DatabaseConfig {

    private final AppProperties appProps;

    public DatabaseConfig(AppProperties appProps) {
        this.appProps = appProps;
    }

    @Bean
    @Primary
    public DataSource primaryDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(appProps.getDatabase().getPrimary().getUrl());
        config.setMaximumPoolSize(appProps.getDatabase().getPrimary().getPoolSize());
        return new HikariDataSource(config);
    }

    @Bean
    public DataSource replicaDataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(appProps.getDatabase().getReplica().getUrl());
        config.setMaximumPoolSize(appProps.getDatabase().getReplica().getPoolSize());
        return new HikariDataSource(config);
    }
}
```

**Common gotcha:** Never use `@Autowired` field injection for `@ConfigurationProperties` beans. Always use constructor injection — it makes the dependency explicit and the class testable without Spring:

```java
// Bad — field injection hides the dependency
@Service
public class BadService {
    @Autowired
    private AppProperties props; // hidden dependency, untestable in isolation
}

// Good — constructor injection
@Service
public class GoodService {
    private final AppProperties props;

    public GoodService(AppProperties props) {
        this.props = props; // explicit, easy to unit test with new GoodService(mockProps)
    }
}
```

---

## 2.6 Profiles

Profiles let you have different configurations for different environments without changing code.

### Defining Profile-Specific Config

```yaml
# application.yml — shared config
spring:
  application:
    name: order-service

# application-dev.yml — overrides for dev
spring:
  datasource:
    url: jdbc:h2:mem:devdb
logging:
  level:
    root: DEBUG

# application-prod.yml — overrides for prod
spring:
  datasource:
    url: jdbc:postgresql://prod-host:5432/orders
logging:
  level:
    root: WARN
```

### Profile-Specific Beans

```java
@Service
@Profile("dev")
public class MockEmailService implements EmailService {
    public void send(String to, String body) {
        System.out.println("DEV: Would send email to " + to);
    }
}

@Service
@Profile("prod")
public class SmtpEmailService implements EmailService {
    public void send(String to, String body) {
        // real SMTP sending
    }
}
```

### Activating Profiles

```bash
# Environment variable
SPRING_PROFILES_ACTIVE=prod java -jar app.jar

# Command-line
java -jar app.jar --spring.profiles.active=prod

# In application.yml (useful for setting default)
spring:
  profiles:
    default: dev
```

### Multiple Active Profiles

```bash
# Combine profiles
java -jar app.jar --spring.profiles.active=prod,metrics,featureX
```

---

## 2.7 Embedded Server

Spring Boot embeds Tomcat by default inside the JAR. You run `java -jar app.jar` instead of deploying a WAR to an external server.

### Switching Embedded Server

```xml
<!-- In pom.xml — switch from Tomcat to Undertow -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-undertow</artifactId>
</dependency>
```

### Common Server Config

```yaml
server:
  port: 8080
  servlet:
    context-path: /api       # all endpoints are at /api/...
  compression:
    enabled: true
    mime-types: application/json,text/html
  tomcat:
    max-threads: 200
    connection-timeout: 5000
```

---

## Tricky Interview Questions

**Q: What is the difference between `application.properties` and `application.yml`? Which is better?**

They're equivalent — Spring supports both. YAML is preferred for complex nested config (less repetition). You can use both simultaneously — properties file takes higher precedence over YAML. In a real project, pick one and stick to it.

---

**Q: You defined a `DataSource` `@Bean` in your config class. How does Spring Boot know not to apply `DataSourceAutoConfiguration`?**

Because `DataSourceAutoConfiguration` is annotated with `@ConditionalOnMissingBean(DataSource.class)`. When Spring Boot sees your `DataSource` bean already registered, the condition evaluates to false, and `DataSourceAutoConfiguration` backs off entirely.

---

**Q: Your `@Value("${smtp.host}")` is throwing `IllegalArgumentException` at startup with "Could not resolve placeholder." What do you check?**

1. Is the property defined in `application.yml` / `application.properties`?
2. Is the right profile active? The property might be in `application-prod.yml` but you're running dev.
3. Is the class annotated with `@Component` (or scanned)? `@Value` only works on Spring-managed beans.
4. Typo in the property name?
5. If the value is optional, use `@Value("${smtp.host:localhost}")` with a default.

---

**Q: What's the difference between `@Value` and `@ConfigurationProperties`?**

| | `@Value` | `@ConfigurationProperties` |
|---|---|---|
| Binding | Single property | Entire prefix group |
| Type safety | Limited (SpEL string) | Full type binding |
| Validation | No | Yes (`@Validated`) |
| IDE support | Weak | Full autocomplete |
| Relaxed binding | No | Yes (`SMTP_HOST` binds to `smtp-host`) |

`@ConfigurationProperties` supports relaxed binding: `APP_PAYMENT_TIMEOUT` in env vars binds to `app.payment.timeout` in YAML — they're all the same.

---

**Q: Can you have two `application.yml` files — one inside the JAR and one outside — and which wins?**

Yes. The file outside the JAR (in the current directory or a `config/` subfolder) takes higher precedence than the one inside. This is how you inject production secrets without rebuilding the JAR:

```
Priority (file system locations, highest to lowest):
1. ./config/application.yml       (config subdir of current dir)
2. ./application.yml              (current directory)
3. classpath:/config/application.yml
4. classpath:application.yml      (inside JAR)
```

---

**Q: What is `SpringApplication` and what customization can you do before `run()`?**

`SpringApplication` is the bootstrap class. You can customize it before `run()`:

```java
public static void main(String[] args) {
    SpringApplication app = new SpringApplication(Application.class);
    app.setBannerMode(Banner.Mode.OFF);          // disable startup banner
    app.setWebApplicationType(WebApplicationType.NONE); // no web server (for CLI apps)
    app.addListeners(new MyApplicationListener()); // listen to startup events
    app.run(args);
}
```

Or use `SpringApplicationBuilder` for a fluent API:
```java
new SpringApplicationBuilder(Application.class)
    .profiles("prod")
    .run(args);
```
