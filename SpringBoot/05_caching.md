# 05 — Caching

---

## 5.1 Why Caching Exists

Every time your code calls the database, there's latency (network round trip + query execution). For data that:
- Is read frequently but changes rarely (e.g., product catalog, user profile, config)
- Is expensive to compute (e.g., aggregated reports)
- Comes from a slow external API

...caching stores the result in fast memory so subsequent reads skip the slow operation entirely.

```
Without cache:
  Request → Service → DB (10ms) → Response

With cache:
  1st request → Service → Cache MISS → DB (10ms) → Store in Cache → Response
  2nd+ request → Service → Cache HIT → Return cached value (0.1ms) → Response
```

---

## 5.2 Spring Cache Abstraction

Spring provides a cache abstraction — you annotate methods, and Spring handles the caching logic. The underlying cache store (in-memory, Redis, Caffeine) is pluggable.

### Enable Caching

```java
@SpringBootApplication
@EnableCaching  // required — enables annotation-driven cache management
public class Application { ... }
```

---

## 5.3 Cache Annotations

### `@Cacheable` — Read-Through Cache

Cache the return value. On subsequent calls with the same key, return cached value without executing the method:

```java
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public Product findById(Long id) {
        // This method body is SKIPPED on a cache hit
        return productRepository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }

    // Default key = all parameters combined
    @Cacheable("products")
    public List<Product> findByCategory(String category) {
        return productRepository.findByCategory(category);
    }

    // Conditional caching
    @Cacheable(value = "products", key = "#id", condition = "#id > 0")
    public Product findByIdConditional(Long id) { ... }

    // Don't cache null results
    @Cacheable(value = "products", key = "#id", unless = "#result == null")
    public Product findByIdNullSafe(Long id) { ... }
}
```

### `@CachePut` — Write-Through Cache

Always run the method AND update the cache. Use after creating or updating data:

```java
@CachePut(value = "products", key = "#result.id")
public Product createProduct(CreateProductRequest request) {
    Product product = productMapper.toEntity(request);
    return productRepository.save(product);  // saves AND updates cache
}

@CachePut(value = "products", key = "#id")
public Product updateProduct(Long id, UpdateProductRequest request) {
    Product existing = productRepository.findById(id)
        .orElseThrow(() -> new ProductNotFoundException(id));
    productMapper.update(existing, request);
    return productRepository.save(existing);
}
```

### `@CacheEvict` — Remove from Cache

Remove stale data when something changes:

```java
@CacheEvict(value = "products", key = "#id")
public void deleteProduct(Long id) {
    productRepository.deleteById(id);
}

// Clear entire cache
@CacheEvict(value = "products", allEntries = true)
public void clearAll() {
    // just triggers eviction — the method body can be empty
}

// Evict before the method runs (default is after)
@CacheEvict(value = "products", key = "#id", beforeInvocation = true)
public void deleteProductEarly(Long id) {
    productRepository.deleteById(id);
    // even if deleteById throws, cache entry was already removed
}
```

### `@Caching` — Multiple Cache Operations

```java
@Caching(
    put = { @CachePut(value = "users", key = "#result.id") },
    evict = { @CacheEvict(value = "usersList", allEntries = true) }
)
public User createUser(CreateUserRequest request) {
    return userRepository.save(userMapper.toEntity(request));
}
```

---

## 5.4 Cache Key Generation

Spring's default key = method parameters combined:

```java
@Cacheable("users")
public User findByEmailAndStatus(String email, UserStatus status) {
    // default key = email + status (SimpleKey)
}
```

Custom key expressions use SpEL (Spring Expression Language):

```java
@Cacheable(value = "users", key = "#email.toLowerCase()")
public User findByEmail(String email) { ... }

@Cacheable(value = "orders", key = "'user:' + #userId + ':status:' + #status")
public List<Order> findOrdersByUserAndStatus(Long userId, OrderStatus status) { ... }

// key based on result (for @CachePut)
@CachePut(value = "users", key = "#result.id")
public User save(User user) { ... }

// Custom KeyGenerator bean
@Cacheable(value = "data", keyGenerator = "customKeyGenerator")
public Data fetchData(ComplexParam param) { ... }
```

---

## 5.5 Cache Providers

### Default: Simple (ConcurrentHashMap)

Auto-configured when no other cache library is on the classpath. Good for:
- Development/testing
- Single-node apps with small datasets
- No eviction needed

Limitations: no TTL, no eviction, not distributed, lost on restart.

### Caffeine — High-Performance In-Process Cache

```xml
<dependency>
    <groupId>com.github.ben-manes.caffeine</groupId>
    <artifactId>caffeine</artifactId>
</dependency>
```

```yaml
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=1000,expireAfterWrite=10m
```

Or programmatic configuration with eviction policies:

```java
@Bean
public CacheManager cacheManager() {
    CaffeineCacheManager manager = new CaffeineCacheManager();
    manager.setCaffeine(Caffeine.newBuilder()
        .maximumSize(1000)
        .expireAfterWrite(10, TimeUnit.MINUTES)
        .recordStats());  // enables hit/miss metrics
    return manager;
}
```

### Redis — Distributed Cache (Production Standard)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```

```yaml
spring:
  cache:
    type: redis
  redis:
    host: localhost
    port: 6379
    password: yourpassword
```

```java
@Bean
public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
    RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(10))
        .serializeValuesWith(
            RedisSerializationContext.SerializationPair.fromSerializer(
                new GenericJackson2JsonRedisSerializer()  // store as JSON, not binary
            )
        )
        .disableCachingNullValues();

    // Per-cache TTL
    Map<String, RedisCacheConfiguration> cacheConfigs = Map.of(
        "users",    config.entryTtl(Duration.ofMinutes(30)),
        "products", config.entryTtl(Duration.ofHours(1))
    );

    return RedisCacheManager.builder(connectionFactory)
        .cacheDefaults(config)
        .withInitialCacheConfigurations(cacheConfigs)
        .build();
}
```

---

## 5.6 Cache Pitfalls

### 1. Caching Mutable Objects

```java
@Cacheable("users")
public User getUser(Long id) {
    return userRepository.findById(id).orElseThrow();
}

// Caller does this:
User user = userService.getUser(1L);
user.setName("Hacked!");  // Mutates the cached object!

// Next caller:
User user2 = userService.getUser(1L);  // gets the mutated object from cache!
```

**Fix:** Return DTOs (immutable), or deep-copy before returning from cache.

### 2. Not Setting TTL in Production

Without a TTL, cached data stays forever. User changes their profile → old data served indefinitely.

**Fix:** Always set a TTL matching the acceptable staleness of the data.

### 3. Cache Stampede (Thundering Herd)

When a popular cache entry expires, hundreds of requests simultaneously see a cache miss and all hit the database at once.

**Fix:** Use probabilistic early expiration, or a cache-aside pattern with a lock.

### 4. Self-Invocation (Same as `@Transactional`)

```java
@Service
public class ProductService {
    @Cacheable("products")
    public Product findById(Long id) { ... }

    public Product findByIdWrapper(Long id) {
        return findById(id);  // BYPASSES proxy — caching does not work
    }
}
```

**Fix:** Call through the proxy — inject `ProductService` into itself, or extract to a separate bean.

### 5. Caching Without Serializable Objects (Redis)

```java
// Product must be Serializable for Redis cache
public class Product implements Serializable {  // required for default Redis serialization
    private Long id;
    // ...
}
```

With `GenericJackson2JsonRedisSerializer`, you need default constructors and proper Jackson config, not `Serializable`.

---

## Tricky Interview Questions

**Q: What is the difference between `@Cacheable` and `@CachePut`?**

- `@Cacheable`: on a cache hit, **skip the method** and return the cached value. Updates cache only on miss.
- `@CachePut`: **always execute the method**, then update the cache with the result. Never skips the method body.

Use `@Cacheable` for reads, `@CachePut` for writes/updates.

---

**Q: You annotate a method with `@Cacheable`. The method is called 5 times with the same argument. How many times does the DB query run?**

**Once.** First call misses cache → calls DB → stores result. Calls 2–5 hit the cache and return the stored result without touching the DB.

---

**Q: Two servers are running your app. Server 1 updates a product and evicts from its local cache. Server 2 still has stale data in its local cache. How do you fix this?**

Use a **distributed cache** like Redis. Both servers share the same cache — when Server 1 evicts, Server 2 also sees the eviction. In-process caches (Caffeine, ConcurrentHashMap) are per-JVM and not shared.

---

**Q: `@Cacheable` on a method that returns `null`. What happens?**

By default, Spring **does cache null values**. Subsequent calls return `null` from cache without hitting the DB. This is often undesirable (the data might exist later). Prevent it:

```java
@Cacheable(value = "users", unless = "#result == null")
public User findById(Long id) { ... }
```

Or globally in `RedisCacheConfiguration`:
```java
RedisCacheConfiguration.defaultCacheConfig().disableCachingNullValues()
```

---

**Q: How do you cache only some calls based on a condition?**

```java
// Only cache if the result has more than 0 items (don't cache empty lists)
@Cacheable(value = "search", unless = "#result.isEmpty()")
public List<Product> search(String query) { ... }

// Only cache for premium users
@Cacheable(value = "reports", condition = "#user.isPremium()")
public Report generateReport(User user) { ... }
```

- `condition` — evaluated BEFORE method execution (using method parameters)
- `unless` — evaluated AFTER method execution (can use `#result`)
