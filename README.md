# Spring Boot Microservices Issues & Solutions

A comprehensive guide to understanding, diagnosing, and resolving common production issues in Spring Boot microservices architecture.

## Table of Contents

1. [Overview](#overview)
2. [Common Production Issues](#common-production-issues)
3. [Performance & Optimization](#performance--optimization)
4. [Resilience & Fault Tolerance](#resilience--fault-tolerance)
5. [Monitoring & Observability](#monitoring--observability)
6. [Configuration Management](#configuration-management)
7. [Database Issues](#database-issues)
8. [Deployment & Scaling](#deployment--scaling)
9. [Best Practices](#best-practices)
10. [Troubleshooting Guide](#troubleshooting-guide)

---

## Overview

This repository documents real-world challenges encountered in Spring Boot microservices deployments and provides practical solutions, code examples, and best practices to address them. Whether you're dealing with memory leaks, connection pooling issues, or distributed tracing problems, this guide offers insights and remedies.

---

## Common Production Issues

### 1. Memory Leaks & Heap Exhaustion

**Symptoms:**
- Gradual increase in memory consumption
- OutOfMemoryError exceptions
- Garbage collection pauses becoming longer
- Application becoming unresponsive

**Root Causes:**
- Unbounded caches without eviction policies
- Improper connection pooling configuration
- Circular object references
- Static collections accumulating objects

**Solutions:**

```properties
# Heap Memory Configuration
-Xms2g
-Xmx4g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:+ParallelRefProcEnabled
```

```java
@Configuration
public class CacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        CacheBuilder<Object, Object> cacheBuilder = CacheBuilder.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(10, TimeUnit.MINUTES)
            .recordStats();
        
        return new CaffeineCacheManager();
    }
}
```

**Prevention:**
- Use bounded caches with TTL
- Monitor heap usage with JMX metrics
- Perform regular load testing
- Analyze heap dumps using Eclipse MAT or JProfiler

---

### 2. Connection Pool Exhaustion

**Symptoms:**
- "Unable to get a connection, pool error" exceptions
- Requests timing out waiting for database connections
- Application hangs intermittently

**Root Causes:**
- Connections not being closed properly
- Long-running queries holding connections
- Insufficient pool size for concurrent load
- Connection leaks in try-catch blocks

**Solutions:**

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
      auto-commit: false
      leak-detection-threshold: 60000
      connection-test-query: "SELECT 1"
```

```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    public DataSource dataSource(DataSourceProperties properties) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(properties.getUrl());
        config.setUsername(properties.getUsername());
        config.setPassword(properties.getPassword());
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setLeakDetectionThreshold(60000);
        config.setAutoCommit(false);
        
        return new HikariDataSource(config);
    }
}
```

**Prevention:**
- Use try-with-resources for database operations
- Set appropriate timeout values
- Monitor connection pool metrics
- Configure leak detection

---

### 3. Database Query Performance

**Symptoms:**
- Slow API response times
- High CPU and I/O on database server
- Timeout exceptions
- Database connection pool exhaustion

**Root Causes:**
- Missing indexes
- N+1 query problems
- Inefficient JOIN operations
- Cartesian products

**Solutions:**

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    
    @Query("SELECT u FROM User u LEFT JOIN FETCH u.roles WHERE u.id = :id")
    Optional<User> findByIdWithRoles(@Param("id") Long id);
    
    @Query(value = "SELECT u FROM User u WHERE u.email = :email", 
           nativeQuery = false)
    @EntityGraph(attributePaths = "roles")
    Optional<User> findByEmail(@Param("email") String email);
}
```

```java
@Service
public class UserService {
    
    @Transactional(readOnly = true)
    public List<User> getAllUsersWithRoles() {
        // Avoid N+1 queries with FETCH JOIN
        return userRepository.findAll(); // Use EntityGraph or QueryDSL
    }
}
```

**Prevention:**
- Use FETCH JOIN for Hibernate queries
- Enable SQL logging to detect N+1 issues
- Create appropriate database indexes
- Use pagination for large result sets

---

### 4. Distributed Transaction Issues

**Symptoms:**
- Data inconsistency across services
- Orphaned records
- Rollback failures
- Eventually consistent data conflicts

**Root Causes:**
- Lack of distributed transaction coordination
- No idempotency implementation
- Missing compensating transactions
- Network failures mid-transaction

**Solutions:**

```java
@Service
public class OrderService {
    
    private final OrderRepository orderRepository;
    private final PaymentClient paymentClient;
    private final InventoryClient inventoryClient;
    
    @Transactional
    public Order createOrder(OrderRequest request) {
        // Step 1: Create order
        Order order = orderRepository.save(new Order(request));
        
        try {
            // Step 2: Process payment
            paymentClient.processPayment(order.getId(), request.getAmount());
            
            // Step 3: Reserve inventory
            inventoryClient.reserveItems(order.getItems());
            
            return order;
        } catch (Exception e) {
            // Compensating transaction
            orderRepository.delete(order);
            throw e;
        }
    }
}
```

**Idempotent Payment Processing:**

```java
@Service
public class PaymentService {
    
    @Transactional
    public PaymentResult processPayment(String idempotencyKey, PaymentRequest request) {
        // Check if payment already processed
        PaymentResult existing = paymentRepository.findByIdempotencyKey(idempotencyKey);
        if (existing != null) {
            return existing;
        }
        
        // Process payment
        PaymentResult result = executePayment(request);
        result.setIdempotencyKey(idempotencyKey);
        
        return paymentRepository.save(result);
    }
}
```

**Prevention:**
- Implement Saga pattern for distributed transactions
- Use idempotency keys for critical operations
- Implement compensating transactions
- Design for eventual consistency

---

### 5. Service-to-Service Communication Failures

**Symptoms:**
- Cascading failures across services
- TimeoutException
- CircuitBreaker trips
- Slow response times

**Root Causes:**
- No retry logic
- No circuit breaker pattern
- Insufficient timeout configuration
- Network instability

**Solutions:**

```java
@Configuration
public class RestClientConfig {
    
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(10))
            .build();
    }
}
```

**Implementing Circuit Breaker with Resilience4j:**

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 5s
        failureRateThreshold: 50
        slowCallRateThreshold: 50
        slowCallDurationThreshold: 2s
  
  retry:
    instances:
      paymentService:
        maxAttempts: 3
        waitDuration: 1000
        retryExceptions:
          - java.net.ConnectException
          - java.io.IOException
  
  timelimiter:
    instances:
      paymentService:
        cancelRunningFuture: false
        timeoutDuration: 10s
```

```java
@Service
public class PaymentClient {
    
    private final RestTemplate restTemplate;
    
    @CircuitBreaker(name = "paymentService")
    @Retry(name = "paymentService")
    @TimeLimiter(name = "paymentService")
    public CompletableFuture<PaymentResponse> processPayment(PaymentRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            ResponseEntity<PaymentResponse> response = restTemplate.postForEntity(
                "http://payment-service/api/payments",
                request,
                PaymentResponse.class
            );
            return response.getBody();
        });
    }
}
```

**Prevention:**
- Implement Circuit Breaker pattern
- Configure appropriate timeouts
- Implement exponential backoff retry logic
- Use bulkhead isolation
- Monitor service health

---

## Performance & Optimization

### 1. Thread Pool Exhaustion

**Configuration:**

```yaml
spring:
  task:
    execution:
      pool:
        core-size: 10
        max-size: 20
        queue-capacity: 100
    scheduling:
      pool:
        size: 5
      thread-name-prefix: "scheduled-"
```

### 2. Inefficient Serialization

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class User {
    private Long id;
    private String name;
    private String email;
    
    @JsonIgnore
    private transient List<Role> roles;
}
```

### 3. String Concatenation & GC Pressure

```java
// ❌ Avoid
String sql = "SELECT * FROM users WHERE id = " + userId;

// ✅ Use StringBuilder
StringBuilder builder = new StringBuilder("SELECT * FROM users WHERE id = ");
builder.append(userId);

// ✅ Or use String.format
String sql = String.format("SELECT * FROM users WHERE id = %d", userId);
```

---

## Resilience & Fault Tolerance

### 1. Bulkhead Pattern

```java
@Configuration
public class ResilienceConfig {
    
    @Bean
    public Bulkhead bulkhead() {
        BulkheadConfig config = BulkheadConfig.custom()
            .maxConcurrentCalls(10)
            .maxWaitDuration(Duration.ofSeconds(5))
            .build();
        return Bulkhead.of("payment", config);
    }
}
```

### 2. Rate Limiting

```java
@Service
public class RateLimitedService {
    
    private final RateLimiter rateLimiter;
    
    @Bulkhead(name = "paymentService", type = Type.THREADPOOL)
    @RateLimiter(name = "paymentService")
    public void processPayment(PaymentRequest request) {
        // Process payment
    }
}
```

---

## Monitoring & Observability

### 1. Metrics Collection with Micrometer

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    export:
      prometheus:
        enabled: true
  endpoint:
    health:
      show-details: always
```

```java
@Configuration
public class MetricsConfig {
    
    @Bean
    public MeterBinder customMetrics() {
        return (registry) -> {
            Counter.builder("orders.created")
                .description("Total orders created")
                .register(registry);
        };
    }
}
```

### 2. Structured Logging

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

### 3. Distributed Tracing with Sleuth & Zipkin

```yaml
spring:
  sleuth:
    sampler:
      probability: 1.0
    zipkin:
      base-url: http://zipkin:9411/
      enabled: true
```

---

## Configuration Management

### 1. Externalized Configuration

```yaml
# application.yml
spring:
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:dev}
  application:
    name: user-service
  config:
    import: "configserver:"

management:
  endpoints:
    web:
      exposure:
        include: "*"
```

### 2. Secrets Management

```java
@Configuration
public class SecretsConfig {
    
    @Value("${db.password}")
    private String dbPassword;
    
    @Bean
    public DataSource dataSource() {
        // Use dbPassword from secure vault
        return null;
    }
}
```

---

## Database Issues

### 1. Connection Timeout Configuration

```properties
spring.datasource.hikari.connection-timeout=30000
spring.datasource.hikari.idle-timeout=600000
spring.datasource.hikari.max-lifetime=1800000
```

### 2. Transaction Timeout

```java
@Service
public class UserService {
    
    @Transactional(timeout = 30)
    public User updateUser(Long id, UserRequest request) {
        // Method must complete within 30 seconds
        return userRepository.save(user);
    }
}
```

### 3. Deadlock Prevention

```java
@Transactional
public void transferMoney(Account from, Account to, BigDecimal amount) {
    // Lock accounts in consistent order to prevent deadlocks
    Account account1 = from.getId() < to.getId() ? from : to;
    Account account2 = from.getId() < to.getId() ? to : from;
    
    entityManager.lock(account1, LockModeType.PESSIMISTIC_WRITE);
    entityManager.lock(account2, LockModeType.PESSIMISTIC_WRITE);
    
    // Perform transfer
}
```

---

## Deployment & Scaling

### 1. Health Checks Configuration

```yaml
management:
  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
  endpoint:
    health:
      probes:
        enabled: true
      show-details: always
```

```java
@Component
public class CustomHealthIndicator extends AbstractHealthIndicator {
    
    @Override
    protected void doHealthCheck(Health.Builder builder) {
        try {
            // Check critical service dependency
            boolean isHealthy = checkServiceHealth();
            if (isHealthy) {
                builder.up().withDetail("service", "available");
            } else {
                builder.down().withDetail("service", "unavailable");
            }
        } catch (Exception e) {
            builder.down().withException(e);
        }
    }
}
```

### 2. Graceful Shutdown

```yaml
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

server:
  shutdown: graceful
```

```java
@Component
public class GracefulShutdownHandler {
    
    @PreDestroy
    public void shutdown() {
        // Perform cleanup operations
        // Complete in-flight requests
        // Close connections gracefully
    }
}
```

### 3. Docker Optimization

```dockerfile
FROM eclipse-temurin:17-jre-alpine

RUN addgroup -g 1001 -S app && adduser -u 1001 -S app -G app

WORKDIR /app

COPY --chown=app:app build/libs/app.jar app.jar

USER app

ENTRYPOINT ["java", "-XX:+UseG1GC", "-XX:MaxGCPauseMillis=200", "-jar", "app.jar"]
```

---

## Best Practices

### 1. Error Handling & Resilience

```java
@ControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException e) {
        ErrorResponse error = new ErrorResponse(
            "RESOURCE_NOT_FOUND",
            e.getMessage(),
            LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGenericException(Exception e) {
        ErrorResponse error = new ErrorResponse(
            "INTERNAL_ERROR",
            "An unexpected error occurred",
            LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

### 2. Logging Best Practices

```java
@Service
public class OrderService {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderService.class);
    
    public Order createOrder(OrderRequest request) {
        logger.info("Creating order for customer: {}", request.getCustomerId());
        
        try {
            Order order = performOrderCreation(request);
            logger.info("Order created successfully: {}", order.getId());
            return order;
        } catch (Exception e) {
            logger.error("Failed to create order for customer: {}", 
                request.getCustomerId(), e);
            throw e;
        }
    }
}
```

### 3. API Rate Limiting

```java
@Configuration
public class RateLimitingConfig {
    
    @Bean
    public FilterRegistrationBean<RateLimitingFilter> rateLimitingFilter() {
        FilterRegistrationBean<RateLimitingFilter> registrationBean = 
            new FilterRegistrationBean<>();
        registrationBean.setFilter(new RateLimitingFilter());
        registrationBean.addUrlPatterns("/api/*");
        return registrationBean;
    }
}
```

---

## Troubleshooting Guide

### Issue: High CPU Usage

**Steps:**
1. Enable profiling: `-XX:+UnlockCommercialFeatures -XX:+FlightRecorder`
2. Analyze with Java Mission Control (JMC)
3. Look for:
   - Busy wait loops
   - Inefficient algorithms
   - Excessive logging

### Issue: OutOfMemoryError

**Steps:**
1. Generate heap dump: `-XX:+HeapDumpOnOutOfMemoryError`
2. Analyze with Eclipse MAT
3. Look for:
   - Large object retention
   - Circular references
   - Unbounded collections

### Issue: Slow API Responses

**Steps:**
1. Enable SQL logging: `logging.level.org.hibernate.SQL=DEBUG`
2. Check database query performance
3. Monitor thread pool utilization
4. Check for network latency
5. Review cache hit rates

### Issue: Database Connection Errors

**Steps:**
1. Check connection pool status
2. Verify database server availability
3. Review network connectivity
4. Check for connection leaks
5. Increase pool size if needed

---

## Contributing

We welcome contributions! Please feel free to submit pull requests or open issues for bugs and feature requests.

## License

This project is licensed under the MIT License - see the LICENSE file for details.

## Resources

- [Spring Boot Official Documentation](https://spring.io/projects/spring-boot)
- [Resilience4j Documentation](https://resilience4j.readme.io/)
- [Micrometer Documentation](https://micrometer.io/)
- [Spring Cloud Sleuth](https://spring.io/projects/spring-cloud-sleuth)
- [HikariCP Documentation](https://github.com/brettwooldridge/HikariCP)

---

**Last Updated:** 2025-12-25

For questions or discussions, please open an issue in the repository.
