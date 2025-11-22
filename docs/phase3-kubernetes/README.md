# Phase 3: Kubernetes & AWS Deployment

> 🚀 **Advanced microservices architecture with Kubernetes orchestration, distributed tracing, Saga pattern, and cloud-native improvements**

## 📋 Overview

**Status**: 🔄 In Progress (25% Complete) | **Target**: AWS EKS Deployment

**Progress**: 
- Rich Domain Model refactoring completed for 3 services (user-service, content-service, engagement-service)
- Inter-service communication optimization completed (blocking write operations migrated to Kafka events)
- Hybrid idempotency implementation completed (Redis + Database table for all event consumers)

Phase 3 represents a significant evolution from Phase 2, focusing on:
- **Cloud-Native Architecture**: Kubernetes-native service discovery and orchestration
- **Domain-Driven Design**: Rich domain models with proper aggregate boundaries
- **Event-Driven Excellence**: Improved eventual consistency patterns
- **Distributed Systems**: Saga pattern, distributed tracing, service mesh
- **Production Hardening**: Circuit breakers, rate limiting, idempotency

## 🎯 Planned Features

### Infrastructure & Deployment
- ✅ Kubernetes orchestration (AWS EKS)
- ✅ Kubernetes-native service discovery (replacing Eureka)
- ✅ Auto-scaling and container orchestration
- ✅ AWS cloud services integration
- ✅ Service mesh (Istio/Linkerd)
- ✅ Distributed tracing (Jaeger/Zipkin)

### Architecture Improvements
- ✅ **Rich Domain Models** (fixing Anemic Domain Model) - **COMPLETED**
  - ✅ user-service: User entity with business logic methods
  - ✅ content-service: Novel, Chapter, Category entities with business logic methods
  - ✅ engagement-service: Comment, Review, Report, Vote entities with business logic methods
- ✅ **Inter-Service Communication Optimization** - **COMPLETED**
  - ✅ Migrated blocking write operations to Kafka events
  - ✅ Decision: Cache table pattern not needed (write operations optimized, read operations acceptable)
- ✅ **Hybrid Idempotency Implementation** - **COMPLETED**
  - ✅ Implemented hybrid approach: Redis (fast checks) + Database table (persistent backup)
  - ✅ Created `processed_events` table in all event-consuming services
  - ✅ All Kafka event consumers now use hybrid idempotency (gamification-service, content-service, user-service)
  - ✅ Ensures idempotency even when Redis is restarted
- [ ] Repository Pattern implementation
- [ ] Aggregate boundaries and Domain Events
- [ ] SAGA pattern for distributed transactions

### Resilience & Observability
- ✅ Circuit breakers (comprehensive coverage)
- ✅ Rate limiting
- ✅ Distributed tracing
- ✅ Enhanced monitoring and observability

### Security Improvements
- ✅ Gateway-level JWT authentication (centralized validation)
- ✅ Token validation at API Gateway (reduce microservice load)
- ✅ Consistent security policy across all services
- ✅ Inactive user token validation (fix security issue where inactive users can still use tokens)

## 🏗️ Architecture Improvements

### 1. Rich Domain Model (Fixing Anemic Domain Model)

**Problem**: Current entities are Anemic Domain Models - they only contain data without business logic.

**Solution**: Move business logic into domain entities and aggregates.

**Example**:
```java
// ❌ Phase 2: Anemic Domain Model
@Entity
public class Novel {
    private Long id;
    private String title;
    private NovelStatus status;
    // No business logic
}

// Service directly sets status
novel.setStatus(NovelStatus.PUBLISHED);
novelMapper.update(novel);

// ✅ Phase 3: Rich Domain Model
@Entity
public class Novel {
    private Long id;
    private String title;
    private NovelStatus status;
    
    // Business logic in domain
    public void publish() {
        if (this.status == NovelStatus.DRAFT) {
            this.status = NovelStatus.PUBLISHED;
            DomainEventPublisher.publish(new NovelPublishedEvent(this.id));
        } else {
            throw new IllegalStateException("Only draft novels can be published");
        }
    }
    
    public void archive() {
        this.status = NovelStatus.ARCHIVED;
        DomainEventPublisher.publish(new NovelArchivedEvent(this.id));
    }
}

// Service calls domain method
novel.publish(); // Domain handles state transition
```

**Benefits**:
- Encapsulation of business rules
- Self-documenting domain logic
- Easier to test and maintain
- Prevents invalid state transitions

---

### 2. Repository Pattern Implementation

**Problem**: Services directly call mappers, violating separation of concerns.

**Solution**: Introduce Repository pattern to abstract data access.

**Implementation**:
```java
// ✅ Phase 3: Repository Pattern

// Domain interface
public interface NovelRepository {
    Novel findById(Long id);
    Novel save(Novel novel);
    void delete(Long id);
    List<Novel> findByAuthorId(Long authorId);
    // Aggregate-level operations
    NovelWithChapters findNovelWithChapters(Long novelId);
}

// Infrastructure implementation
@Repository
public class MyBatisNovelRepository implements NovelRepository {
    private final NovelMapper novelMapper;
    private final ChapterMapper chapterMapper;
    private final CategoryMapper categoryMapper;
    
    @Override
    public NovelWithChapters findNovelWithChapters(Long novelId) {
        Novel novel = novelMapper.findById(novelId);
        List<Chapter> chapters = chapterMapper.findByNovelId(novelId);
        List<Category> categories = categoryMapper.findByNovelId(novelId);
        return new NovelWithChapters(novel, chapters, categories);
    }
}

// Service depends on repository, not mapper
@Service
public class NovelService {
    private final NovelRepository novelRepository; // ✅ Repository, not mapper
    
    public void publishNovel(Long novelId) {
        Novel novel = novelRepository.findById(novelId);
        novel.publish(); // Domain method
        novelRepository.save(novel);
    }
}
```

**Benefits**:
- Clean separation between domain and infrastructure
- Easier to test (mock repository)
- Aggregate-level operations encapsulated
- Can switch data access technology without changing service

---

### 3. Aggregate Boundaries & Domain Events

**Problem**: Services cross aggregate boundaries with direct calls, violating DDD principles.

**Solution**: Define clear aggregate boundaries and use Domain Events for inter-aggregate communication.

**Aggregate Boundaries**:
```
Content Service:
├── Novel Aggregate (root)
│   ├── Novel entity
│   ├── Chapter entities (child entities)
│   └── Category references
└── Domain Events: NovelCreated, NovelPublished, ChapterAdded

Engagement Service:
├── Comment Aggregate (root)
│   └── Comment entity
├── Review Aggregate (root)
│   └── Review entity
└── Domain Events: CommentCreated, ReviewSubmitted

Gamification Service:
├── UserProgress Aggregate (root)
│   ├── UserProgress entity
│   ├── Achievement entities
│   └── Transaction entities
└── Domain Events: LevelUp, AchievementUnlocked
```

**Domain Events vs Integration Events**:
```java
// ✅ Domain Event (internal to aggregate, same transaction)
@Entity
public class Novel {
    public void publish() {
        this.status = NovelStatus.PUBLISHED;
        // Domain event - same transaction
        DomainEventPublisher.publish(new NovelPublishedEvent(this.id));
    }
}

// Integration Event (cross-service, via Kafka)
@Service
public class NovelService {
    @Transactional
    public void publishNovel(Long novelId) {
        Novel novel = novelRepository.findById(novelId);
        novel.publish(); // Domain event (internal)
        
        // Integration event (external, after transaction)
        kafkaProducer.send(new NovelPublishedIntegrationEvent(novel.getId()));
    }
}
```

**Rules**:
- ✅ **Within Aggregate**: Direct method calls, domain events (same transaction)
- ✅ **Cross Aggregate (Same Service)**: Domain events (same transaction)
- ✅ **Cross Service (Eventual Consistency OK)**: Integration events (Kafka)
- ✅ **Cross Service (Strong Consistency Required)**: Consider merging aggregates or sync call

**Transaction Guarantee**:
```java
@Transactional
public void publishNovel(Long novelId) {
    Novel novel = novelRepository.findById(novelId);
    novel.publish(); // Domain event published
    
    // Domain event and aggregate modification in same transaction
    // If transaction fails, domain event is rolled back
}
```

---

### 4. Inter-Service Communication Optimization ✅ **COMPLETED**

**Problem**: Too many synchronous API calls between services, especially blocking write operations that affect response times.

**Initial Proposal**: Event-driven cache tables with eventual consistency (as shown in code example below).

**Actual Solution**: Migrated blocking write operations to Kafka events instead of implementing cache tables.

**Implementation**:
```java
// ✅ Phase 3: Cache Table Pattern

// Engagement Service - Cache Table
@Entity
@Table(name = "novel_cache")
public class NovelCache {
    @Id
    private Long novelId;
    private String title;
    private NovelStatus status;
    private Long authorId;
    private LocalDateTime lastUpdated;
}

// Event Listener
@Component
public class NovelCacheEventListener {
    
    @KafkaListener(topics = "novel.events")
    public void handleNovelEvent(NovelEvent event) {
        switch (event.getType()) {
            case NOVEL_CREATED:
            case NOVEL_UPDATED:
                novelCacheRepository.saveOrUpdate(
                    new NovelCache(
                        event.getNovelId(),
                        event.getTitle(),
                        event.getStatus(),
                        event.getAuthorId()
                    )
                );
                break;
            case NOVEL_DELETED:
                novelCacheRepository.delete(event.getNovelId());
                break;
        }
    }
}

// Bootstrap on startup (for existing data)
@PostConstruct
public void bootstrapNovelCache() {
    // Sync call to get all existing novels
    List<NovelDTO> novels = contentServiceClient.getAllNovels();
    novels.forEach(novel -> {
        novelCacheRepository.save(new NovelCache(
            novel.getId(),
            novel.getTitle(),
            novel.getStatus(),
            novel.getAuthorId()
        ));
    });
}

// Service uses cache instead of API call
@Service
public class CommentService {
    public void createComment(Long novelId, String content) {
        // ✅ Check cache instead of API call
        NovelCache novel = novelCacheRepository.findById(novelId)
            .orElseThrow(() -> new NovelNotFoundException(novelId));
        
        if (novel.getStatus() != NovelStatus.PUBLISHED) {
            throw new NovelNotPublishedException(novelId);
        }
        
        // Create comment...
    }
}
```

**Why Cache Table Pattern Was NOT Implemented**:

After analysis, we determined that **cache table pattern is not necessary** for this use case. Here's why:

1. **Write Operations Already Optimized**: The blocking write operations have been migrated to Kafka:
   - `updateNovelRatingAndCount` → `novel-rating-events` Kafka topic
   - `incrementVoteCount` → `novel-vote-count-events` Kafka topic
   - Response time improved from 600-700ms to <100ms (non-blocking)

2. **Read-Only Operations Are Acceptable**: Remaining sync calls are primarily GET operations:
   - `getNovelById()` - Validate novel exists (validation step)
   - `getNovelsBatch()` - Batch fetch for display
   - `getChapterById()` - Read-only data retrieval
   - These operations don't block user writes and have acceptable latency (<50ms typically)

3. **Complexity vs. Benefit Trade-off**: Cache table pattern adds complexity:
   - Requires additional database tables per service
   - Needs bootstrap mechanism for existing data
   - Requires event listeners for cache updates
   - Introduces eventual consistency challenges
   - **Benefit is minimal** since write operations are already optimized

4. **Current Architecture is Sufficient**:
   - **Write operations** (blocking) → Kafka events ✅
   - **Read operations** (non-blocking) → Sync API calls ✅ (acceptable)
   - **Idempotency** → Redis-based checks ✅

**Implemented Solution**: Kafka Event-Driven Architecture

Instead of cache tables, we optimized inter-service communication by:
- Migrating blocking write operations to Kafka events
- Implementing idempotency checks using Redis
- Keeping read-only operations as sync calls (acceptable performance)

**Example - Migrated to Kafka**:
```java
// ❌ Before: Blocking sync API call
contentServiceClient.updateNovelRatingAndCount(novelId, avgRating, reviewCount);
// Response time: 600-700ms (blocking)

// ✅ After: Non-blocking Kafka event
kafkaEventProducerService.publishNovelRatingUpdateEvent(novelId, avgRating, reviewCount);
// Response time: <100ms (non-blocking)
```

**When Cache Tables Would Be Needed**:
- If read-only GET operations become a bottleneck (>100ms regularly)
- If Content Service becomes unavailable frequently
- If we need offline capability for Engagement Service
- If read operations increase significantly (e.g., 1000+ requests/second)

**Current Status**: ✅ **Task Completed** - Inter-service communication optimized via Kafka events for write operations, sync calls remain only for read-only operations where acceptable.

---

### 5. Hybrid Idempotency for Event Consumption ✅ **COMPLETED**

**Problem**: Events might be processed multiple times, causing duplicate side effects. Additionally, using only Redis for idempotency checks risks data loss when Redis is restarted.

**Solution**: Implement hybrid idempotency checks using both Redis (for speed) and a persistent database table (for durability).

**Implementation**:
```java
// ✅ Phase 3: Hybrid Idempotent Event Processing

// Database Entity
@Entity
@Table(name = "processed_events")
public class ProcessedEvent {
    @Id
    private String idempotencyKey; // Unique event identifier
    private String eventType;
    private String serviceName;
    private LocalDateTime processedAt;
    private String eventData; // JSON
}

// Hybrid Idempotency Service
@Service
public class IdempotencyService {
    
    @Autowired
    private RedisUtil redisUtil;
    
    @Autowired
    private ProcessedEventMapper processedEventMapper;
    
    private static final int REDIS_TTL_DAYS = 7;
    
    public boolean isProcessed(String idempotencyKey, String serviceName) {
        // Tier 1: Check Redis first (fast, <1ms)
        String redisKey = "idempotency:" + serviceName + ":" + idempotencyKey;
        if (redisUtil.exists(redisKey)) {
            return true;
        }
        
        // Tier 2: Check Database (slower, but persistent)
        return processedEventMapper.existsByIdempotencyKey(idempotencyKey) != null;
    }
    
    public void markAsProcessed(String idempotencyKey, String eventType, 
                                String serviceName, String eventData) {
        // Write to both Redis and Database
        String redisKey = "idempotency:" + serviceName + ":" + idempotencyKey;
        
        // Redis: Fast access (7-day TTL)
        redisUtil.set(redisKey, "1", REDIS_TTL_DAYS, TimeUnit.DAYS);
        
        // Database: Persistent backup
        ProcessedEvent processedEvent = new ProcessedEvent();
        processedEvent.setIdempotencyKey(idempotencyKey);
        processedEvent.setEventType(eventType);
        processedEvent.setServiceName(serviceName);
        processedEvent.setProcessedAt(LocalDateTime.now());
        processedEvent.setEventData(eventData);
        
        processedEventMapper.insert(processedEvent);
    }
}

// Event Listener with Hybrid Idempotency
@Component
public class EngagementEventListener {
    
    @Autowired
    private IdempotencyService idempotencyService;
    
    @KafkaListener(topics = "novel-rating-events")
    public void handleNovelRatingUpdateEvent(@Payload String eventJson) {
        String idempotencyKey = extractIdempotencyKey(eventJson);
        
        // Hybrid idempotency check
        if (idempotencyService.isProcessed(idempotencyKey, "content-service")) {
            log.info("Event already processed, skipping: {}", idempotencyKey);
            return;
        }
        
        try {
            // Process event
            processNovelRatingUpdate(eventJson);
            
            // Mark as processed (both Redis and DB)
            idempotencyService.markAsProcessed(
                idempotencyKey, 
                "NovelRatingUpdateEvent", 
                "content-service",
                eventJson
            );
        } catch (Exception e) {
            // If processing fails, event is not marked as processed
            // Will be retried by Kafka
            throw e;
        }
    }
}
```

**Hybrid Approach Benefits**:
1. **Fast Access**: Redis provides sub-millisecond lookup times (<1ms)
2. **Durability**: Database table persists idempotency records even if Redis restarts
3. **Recovery**: On Redis restart, system can recover from database table
4. **Performance**: Most checks hit Redis (99%+ hit rate), only fallback to DB when needed
5. **Scalability**: Database table can be cleaned up periodically (old records)

**Database Schema**:
```sql
CREATE TABLE processed_events (
    idempotency_key VARCHAR(255) PRIMARY KEY,
    event_type VARCHAR(100) NOT NULL,
    service_name VARCHAR(50) NOT NULL,
    processed_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    event_data JSONB
);

-- Indexes for performance
CREATE INDEX idx_processed_events_processed_at ON processed_events(processed_at);
CREATE INDEX idx_processed_events_event_type_service ON processed_events(event_type, service_name);
```

**Implementation Status**:
- ✅ **gamification-service**: Hybrid idempotency for all listeners (UserEventListener, EngagementEventListener, InternalEventListener)
- ✅ **content-service**: Hybrid idempotency for EngagementEventListener (novel-rating-events, novel-vote-count-events)
- ✅ **user-service**: Hybrid idempotency for UserActivityListener

**Idempotency Strategies**:
1. **Hybrid Event ID Tracking**: Redis (fast) + Database (persistent)
2. **Idempotent Operations**: Design operations to be naturally idempotent
3. **Idempotency Keys**: Use idempotency keys in event DTOs

**Guarantees**:
- ✅ **At-least-once delivery**: Event processed at least once
- ✅ **Idempotent processing**: Multiple processing = same result
- ✅ **No duplicate side effects**: Even if event is retried
- ✅ **Redis restart resilience**: Idempotency checks survive Redis restarts via database backup
- ✅ **Fast performance**: Redis provides <1ms lookup for most checks

---

### 6. Kubernetes-Native Service Discovery

**Problem**: Eureka adds complexity and is not cloud-native.

**Solution**: Use Kubernetes Service Discovery (native DNS-based).

**Implementation**:
```yaml
# ✅ Phase 3: Kubernetes Service Discovery

# Service definition
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: yushan
spec:
  selector:
    app: user-service
  ports:
    - port: 8080
      targetPort: 8080
  type: ClusterIP

# Pod can discover via DNS
# user-service.yushan.svc.cluster.local:8080
```

**Application Configuration**:
```yaml
# application.yml
spring:
  cloud:
    kubernetes:
      discovery:
        enabled: true
    loadbalancer:
      enabled: true

# OpenFeign uses Kubernetes service discovery
@FeignClient(name = "user-service", url = "http://user-service:8080")
public interface UserServiceClient {
    // Automatically resolves to Kubernetes service
}
```

**Benefits**:
- ✅ Native Kubernetes integration
- ✅ No additional service registry needed
- ✅ Automatic health checks
- ✅ Built-in load balancing
- ✅ Auto-scaling support

---

### 7. Circuit Breaker & Rate Limiter

**Problem**: Missing circuit breakers and rate limiters in some services.

**Solution**: Comprehensive resilience patterns.

**Circuit Breaker**:
```java
// ✅ Phase 3: Circuit Breaker

@Service
public class ContentServiceClient {
    
    @CircuitBreaker(name = "content-service", fallbackMethod = "getNovelFallback")
    @RateLimiter(name = "content-service")
    @Retry(name = "content-service")
    public NovelDTO getNovel(Long novelId) {
        return restTemplate.getForObject(
            "http://content-service/novels/" + novelId,
            NovelDTO.class
        );
    }
    
    public NovelDTO getNovelFallback(Long novelId, Exception e) {
        // Fallback: return cached data or default
        return novelCacheRepository.findById(novelId)
            .map(this::toDTO)
            .orElseThrow(() -> new NovelNotFoundException(novelId));
    }
}
```

**Configuration**:
```yaml
resilience4j:
  circuitbreaker:
    instances:
      content-service:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 5
        permittedNumberOfCallsInHalfOpenState: 3
        automaticTransitionFromOpenToHalfOpenEnabled: true
        waitDurationInOpenState: 10s
        failureRateThreshold: 50
        eventConsumerBufferSize: 10
  ratelimiter:
    instances:
      content-service:
        limitForPeriod: 10
        limitRefreshPeriod: 1s
        timeoutDuration: 0
```

**Rate Limiter**:
```java
@RestController
@RequestMapping("/api/v1/comments")
public class CommentController {
    
    @RateLimiter(name = "comment-creation")
    @PostMapping
    public ResponseEntity<CommentDTO> createComment(@RequestBody CreateCommentRequest request) {
        // Rate limited endpoint
        return ResponseEntity.ok(commentService.createComment(request));
    }
}
```

---

### 8. Distributed Tracing

**Problem**: No distributed tracing, difficult to debug cross-service calls.

**Solution**: Implement distributed tracing with Jaeger or Zipkin.

**Implementation**:
```java
// ✅ Phase 3: Distributed Tracing

// Dependencies
dependencies {
    implementation 'io.micrometer:micrometer-tracing-bridge-brave'
    implementation 'io.zipkin.reporter2:zipkin-reporter-brave'
}

// Configuration
@Configuration
public class TracingConfig {
    @Bean
    public Tracing tracing() {
        return Tracing.newBuilder()
            .localServiceName("user-service")
            .spanReporter(AsyncReporter.create(
                HttpSender.create("http://jaeger:9411/api/v2/spans")
            ))
            .sampler(Sampler.create(1.0f))
            .build();
    }
}

// Automatic instrumentation via Spring Cloud Sleuth
// All HTTP calls, Kafka messages, database queries are traced
```

**Kubernetes Deployment**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jaeger
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: jaeger
        image: jaegertracing/all-in-one:latest
        ports:
        - containerPort: 16686  # UI
        - containerPort: 9411   # Zipkin compatible
```

**Benefits**:
- ✅ End-to-end request tracing
- ✅ Performance bottleneck identification
- ✅ Debugging distributed systems
- ✅ Service dependency visualization

---

### 9. SAGA Pattern for Distributed Transactions

**Problem**: No distributed transaction management for cross-service operations.

**Solution**: Implement SAGA pattern for long-running transactions.

**Choreography SAGA** (Event-driven):
```java
// ✅ Phase 3: SAGA Pattern (Choreography)

// Step 1: Create Order (Order Service)
@Service
public class OrderService {
    public void createOrder(CreateOrderRequest request) {
        Order order = new Order(request);
        orderRepository.save(order);
        
        // Publish event
        kafkaProducer.send(new OrderCreatedEvent(order.getId(), request.getItems()));
    }
}

// Step 2: Reserve Inventory (Inventory Service)
@KafkaListener(topics = "order.events")
public void handleOrderCreated(OrderCreatedEvent event) {
    try {
        inventoryService.reserve(event.getItems());
        kafkaProducer.send(new InventoryReservedEvent(event.getOrderId()));
    } catch (InsufficientInventoryException e) {
        kafkaProducer.send(new InventoryReservationFailedEvent(event.getOrderId()));
    }
}

// Step 3: Process Payment (Payment Service)
@KafkaListener(topics = "inventory.events")
public void handleInventoryReserved(InventoryReservedEvent event) {
    try {
        paymentService.charge(event.getOrderId(), event.getAmount());
        kafkaProducer.send(new PaymentProcessedEvent(event.getOrderId()));
    } catch (PaymentFailedException e) {
        // Compensate: Release inventory
        kafkaProducer.send(new ReleaseInventoryEvent(event.getOrderId()));
        kafkaProducer.send(new OrderFailedEvent(event.getOrderId()));
    }
}

// Compensation Handler
@KafkaListener(topics = "payment.events")
public void handlePaymentFailed(PaymentFailedEvent event) {
    // Compensate: Cancel order
    orderService.cancel(event.getOrderId());
    // Compensate: Release inventory
    inventoryService.release(event.getOrderId());
}
```

**Orchestration SAGA** (Centralized):
```java
// ✅ Phase 3: SAGA Pattern (Orchestration)

@Service
public class OrderSagaOrchestrator {
    
    @Autowired
    private OrderServiceClient orderService;
    @Autowired
    private InventoryServiceClient inventoryService;
    @Autowired
    private PaymentServiceClient paymentService;
    
    @Transactional
    public void processOrder(CreateOrderRequest request) {
        SagaContext context = new SagaContext();
        
        try {
            // Step 1: Create Order
            OrderDTO order = orderService.createOrder(request);
            context.setOrderId(order.getId());
            
            // Step 2: Reserve Inventory
            inventoryService.reserve(order.getItems());
            context.setInventoryReserved(true);
            
            // Step 3: Process Payment
            paymentService.charge(order.getId(), order.getTotal());
            context.setPaymentProcessed(true);
            
            // Complete
            orderService.confirmOrder(order.getId());
            
        } catch (Exception e) {
            // Compensate in reverse order
            compensate(context);
            throw new SagaExecutionException("Order processing failed", e);
        }
    }
    
    private void compensate(SagaContext context) {
        if (context.isPaymentProcessed()) {
            paymentService.refund(context.getOrderId());
        }
        if (context.isInventoryReserved()) {
            inventoryService.release(context.getOrderId());
        }
        if (context.getOrderId() != null) {
            orderService.cancel(context.getOrderId());
        }
    }
}
```

**SAGA State Management**:
```java
@Entity
@Table(name = "saga_instance")
public class SagaInstance {
    @Id
    private String sagaId;
    private String sagaType;
    private SagaStatus status;
    private String currentStep;
    private String compensationData; // JSON
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

**Benefits**:
- ✅ Handles distributed transactions
- ✅ Maintains data consistency across services
- ✅ Compensating transactions for rollback
- ✅ Suitable for long-running processes

---

### 10. Gateway-Level JWT Authentication

**Problem**: Currently, each microservice validates JWT tokens independently, causing:
- Redundant validation across services
- Higher latency (validation at each service)
- Inconsistent security policies
- Higher CPU usage

**Solution**: Centralize JWT validation at API Gateway level.

**Implementation**:
```java
// ✅ Phase 3: Gateway-Level JWT Validation

@Component
public class JwtAuthenticationGatewayFilter implements GatewayFilter, Ordered {
    
    @Autowired
    private JwtUtil jwtUtil;
    
    private static final List<String> PUBLIC_PATHS = List.of(
        "/api/v1/auth/login",
        "/api/v1/auth/register",
        "/api/v1/auth/refresh",
        "/api/v1/novels",  // Public browsing
        "/api/v1/categories",
        "/actuator/health"
    );
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpRequest request = exchange.getRequest();
        String path = request.getURI().getPath();
        
        // Skip validation for public endpoints
        if (PUBLIC_PATHS.stream().anyMatch(path::startsWith)) {
            return chain.filter(exchange);
        }
        
        // Extract token from Authorization header
        String authHeader = request.getHeaders().getFirst("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            return unauthorized(exchange, "Missing or invalid Authorization header");
        }
        
        String token = authHeader.substring(7);
        
        // Validate token at gateway
        if (!jwtUtil.validateToken(token)) {
            return unauthorized(exchange, "Invalid or expired token");
        }
        
        // Extract user info from token
        String userId = jwtUtil.extractUserId(token);
        String email = jwtUtil.extractEmail(token);
        List<String> roles = jwtUtil.extractRoles(token);
        
        // Add user info to request headers for downstream services
        ServerHttpRequest modifiedRequest = request.mutate()
            .header("X-User-Id", userId)
            .header("X-User-Email", email)
            .header("X-User-Roles", String.join(",", roles))
            .header("X-Gateway-Validated", "true")  // Mark as gateway-validated
            .build();
        
        return chain.filter(exchange.mutate().request(modifiedRequest).build());
    }
    
    private Mono<Void> unauthorized(ServerWebExchange exchange, String message) {
        ServerHttpResponse response = exchange.getResponse();
        response.setStatusCode(HttpStatus.UNAUTHORIZED);
        response.getHeaders().add("Content-Type", "application/json");
        
        String body = String.format("{\"error\": \"Unauthorized\", \"message\": \"%s\"}", message);
        DataBuffer buffer = response.bufferFactory().wrap(body.getBytes(StandardCharsets.UTF_8));
        return response.writeWith(Mono.just(buffer));
    }
    
    @Override
    public int getOrder() {
        return -100; // High priority, run early
    }
}
```

**Microservice Simplification**:
```java
// ✅ Phase 3: Simplified Microservice Authentication

// Microservices can trust gateway-validated requests
@Component
public class GatewayAuthenticationFilter extends OncePerRequestFilter {
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                   HttpServletResponse response, 
                                   FilterChain filterChain) throws ServletException, IOException {
        
        // Check if request is gateway-validated
        String gatewayValidated = request.getHeader("X-Gateway-Validated");
        if ("true".equals(gatewayValidated)) {
            // Extract user info from headers (set by gateway)
            String userId = request.getHeader("X-User-Id");
            String email = request.getHeader("X-User-Email");
            String roles = request.getHeader("X-User-Roles");
            
            // Set authentication context
            Authentication authentication = new PreAuthenticatedAuthenticationToken(
                userId, null, parseRoles(roles)
            );
            SecurityContextHolder.getContext().setAuthentication(authentication);
        } else {
            // Fallback: validate token directly (for direct service-to-service calls)
            // This should be rare in Phase 3
            validateTokenDirectly(request);
        }
        
        filterChain.doFilter(request, response);
    }
}
```

**Configuration**:
```yaml
# API Gateway application.yml
spring:
  cloud:
    gateway:
      default-filters:
        - name: JwtAuthentication
          args:
            jwtSecret: ${JWT_SECRET}
            publicPaths: /api/v1/auth/**,/api/v1/novels,/api/v1/categories
```

**Benefits**:
- ✅ **Single Validation Point**: Validate token once at gateway
- ✅ **Reduced Load**: Microservices don't need to validate tokens
- ✅ **Better Performance**: Lower latency (one validation vs. multiple)
- ✅ **Consistent Security**: Centralized security policy
- ✅ **Early Rejection**: Invalid tokens rejected before routing
- ✅ **Simplified Services**: Microservices can trust gateway-validated requests

**Trade-offs**:
- ⚠️ Gateway becomes critical security component (single point of failure)
- ⚠️ Need to ensure gateway is highly available
- ⚠️ Service-to-service calls may need alternative authentication

**Mitigation**:
- Use service mesh (Istio/Linkerd) for service-to-service authentication
- Implement gateway high availability (multiple instances)
- Keep fallback validation in services for direct calls

---

### 11. Inactive User Token Validation (Security Issue)

**Problem**: Currently, when a user becomes inactive/suspended/banned after a token is created, the token can still be used in some services because those services check status from the JWT token (old status) instead of from the database.

**Current Issue**:
- **User Service**: ✅ Checks status from database → Token is rejected when user is inactive
- **Other Services**: ❌ Check status from JWT token → Token still passes when user is inactive

**Security Risk**: User is suspended but can still use old token to call APIs (except User Service).

**Discussed Solutions** (decision pending):

#### Option A: Redis Cache - Full User Status
- **Approach**: Cache all user statuses in Redis
- **Key**: `user:status:{userId}`, Value: `status_code`
- **Update**: User Service updates Redis when status changes (direct or via Kafka)
- **Gateway**: Checks Redis cache when validating JWT
- **Pros**: 
  - Real-time updates
  - Fast lookup (< 1ms)
  - No dependency on User Service
- **Cons**: 
  - High memory usage (10M users = ~500MB)
  - Requires Redis infrastructure
  - Cache miss → Reject token

#### Option B: Redis Block List (Only Inactive Users)
- **Approach**: Only store list of blocked/inactive users in Redis Set
- **Key**: `user:blocklist`, Value: Set of `userId`
- **Update**: User Service adds/removes from block list when status changes
- **Gateway**: Checks block list when validating JWT
- **Logic**: Not in block list = active (assume active)
- **Pros**: 
  - Memory efficient (only inactive users, ~1-5MB for 100K blocked users)
  - Fast lookup O(1)
  - Scalable (block list size does not increase with total users)
- **Cons**: 
  - Eventual consistency window (if event not yet synced)
  - False negatives if event is lost
  - Need fallback for old tokens

#### Option C: Database Table in Gateway
- **Approach**: Gateway has its own database table to store user status
- **Table**: `user_status_cache` with columns: `user_id`, `status`, `last_updated`
- **Sync**: Async from User Service via Kafka events
- **Gateway**: Queries database table when validating JWT
- **Pros**: 
  - Persistent storage (data not lost on restart)
  - Query flexibility (SQL, indexes)
  - No need for separate Redis
- **Cons**: 
  - Higher latency than Redis (~1-5ms vs < 1ms)
  - Additional database dependency for Gateway
  - Database connection overhead

#### Option D: Direct Sync from User Service
- **Approach**: User Service directly calls Gateway API to update blocklist
- **Flow**: User Service → HTTP call → Gateway internal API → Update blocklist
- **Gateway**: Exposes internal API `/internal/blocklist/users/{userId}`
- **Pros**: 
  - Real-time sync (no lag)
  - Guaranteed delivery
  - Simple architecture
- **Cons**: 
  - Tight coupling (User Service depends on Gateway)
  - Synchronous dependency (increases latency)
  - Single point of failure
  - Multiple Gateway instances → Need to update all

#### Option E: Hybrid - Direct Redis Update + Kafka Event + Local Cache (Recommended)
- **Approach**: Combines multiple tiers
- **Tier 1**: Local in-memory cache (Caffeine) in Gateway - 100K users
- **Tier 2**: Redis Set blocklist (shared) - only inactive users
- **Tier 3**: User Service fallback (rare, for old tokens)
- **Sync**: User Service updates Redis directly + publishes Kafka event
- **Gateway**: Local cache → Redis → User Service (fallback)
- **Pros**: 
  - Ultra-fast (local cache < 0.1ms, 99% hit rate)
  - Real-time (direct Redis update)
  - Memory efficient (block list only inactive users)
  - Resilient (multiple tiers)
  - No tight coupling
- **Cons**: 
  - More complex (3 tiers)
  - Requires Redis infrastructure

#### Option F: Hybrid - Database Table + Local Cache
- **Approach**: Database table in Gateway + local cache
- **Tier 1**: Local in-memory cache (Caffeine)
- **Tier 2**: Database table (persistent)
- **Sync**: Kafka events from User Service
- **Pros**: 
  - Persistent (data not lost)
  - Fast with local cache
  - No Redis needed
- **Cons**: 
  - Database latency (~1-5ms)
  - Additional database for Gateway

**Considerations to decide**:
- [ ] Memory vs Latency trade-off
- [ ] Infrastructure preference (Redis vs Database)
- [ ] Coupling preference (direct sync vs event-driven)
- [ ] Scalability requirements (10M vs 100M users)
- [ ] Consistency requirements (real-time vs eventual)

**Note**: Detailed implementation and comparison available in `SECURITY_ISSUE_INACTIVE_USER_TOKEN.md`

---

## 📦 Technology Stack

### Orchestration & Service Discovery
- **Kubernetes**: Container orchestration (AWS EKS)
- **Kubernetes Service Discovery**: Native DNS-based (replacing Eureka)
- **Service Mesh**: Istio or Linkerd (optional)

### Distributed Systems
- **Distributed Tracing**: Jaeger or Zipkin
- **Message Queue**: AWS MSK (Kafka)
- **SAGA Pattern**: Custom implementation or Axon Framework

### Cloud Services (AWS)
- **Compute**: AWS EKS (Kubernetes)
- **Database**: AWS RDS (PostgreSQL)
- **Cache**: AWS ElastiCache (Redis)
- **Search**: AWS OpenSearch (Elasticsearch)
- **Storage**: AWS S3
- **Message Queue**: AWS MSK (Kafka)
- **Monitoring**: AWS CloudWatch, Prometheus, Grafana

### Resilience & Observability
- **Circuit Breaker**: Resilience4j
- **Rate Limiter**: Resilience4j
- **Distributed Tracing**: Micrometer Tracing + Jaeger/Zipkin
- **Monitoring**: Prometheus + Grafana
- **Logging**: ELK Stack or CloudWatch Logs

### Application Framework
- **Backend**: Spring Boot 3.x, Spring Cloud
- **Domain-Driven Design**: Custom implementation
- **Event Sourcing**: Kafka-based
- **Repository Pattern**: Custom implementation

---

## 🗂️ Repository Structure

### Phase 3 Development (Evolved from Phase 2 Repositories)

**Decision**: Phase 3 will be developed directly in Phase 2 repositories using Git branch strategy.

**Repositories** (same as Phase 2):
- `yushan-microservices-user-service`
- `yushan-microservices-content-service`
- `yushan-microservices-engagement-service`
- `yushan-microservices-gamification-service`
- `yushan-microservices-analytics-service`
- `yushan-microservices-api-gateway`
- `yushan-microservices-config-server`
- `yushan-microservices-service-registry`

**Repository Structure** (evolved):
```
yushan-microservices-user-service/
├── k8s/                          # Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   └── ingress.yaml
├── terraform/                    # AWS infrastructure
│   ├── eks.tf
│   ├── rds.tf
│   └── elastiCache.tf
├── src/
│   ├── main/java/
│   │   └── com/yushan/
│   │       ├── domain/           # Domain layer
│   │       │   ├── model/         # Rich domain models
│   │       │   ├── repository/   # Repository interfaces
│   │       │   ├── event/         # Domain events
│   │       │   └── service/       # Domain services
│   │       ├── application/      # Application layer
│   │       │   ├── service/       # Application services
│   │       │   └── dto/
│   │       ├── infrastructure/    # Infrastructure layer
│   │       │   ├── repository/    # Repository implementations
│   │       │   ├── mapper/        # MyBatis mappers
│   │       │   ├── cache/         # Cache tables
│   │       │   ├── tracing/       # Distributed tracing
│   │       │   └── saga/           # SAGA implementation
│   │       └── api/               # API layer
│   │           └── controller/
│   └── main/resources/
│       ├── application.yml
│       └── mapper/                # MyBatis XML
└── README.md
```

---

## 🚀 Migration Strategy

### Git Branch Strategy

**Approach**: Develop Phase 3 directly in Phase 2 repositories using feature branches.

**Branch Structure**:
```
main (Phase 2 - Production)
├── phase2-stable (Phase 2 stable releases)
└── phase3-kubernetes (Phase 3 development branch)
    ├── feature/rich-domain-model
    ├── feature/repository-pattern
    ├── feature/event-cache-tables
    ├── feature/idempotent-events
    ├── feature/kubernetes-migration
    ├── feature/saga-pattern
    └── feature/distributed-tracing
```

**Workflow**:
```bash
# Start Phase 3 development
git checkout -b phase3-kubernetes
git push -u origin phase3-kubernetes

# Create feature branches from phase3-kubernetes
git checkout -b feature/rich-domain-model
# ... implement feature ...
git checkout phase3-kubernetes
git merge feature/rich-domain-model

# Keep Phase 2 stable on main branch
# Phase 2 production deployments use main branch
# Phase 3 development happens on phase3-kubernetes branch
```

**Benefits**:
- ✅ Single source of truth
- ✅ Continuous git history
- ✅ Easy to backport bugfixes from Phase 3 to Phase 2
- ✅ Can merge Phase 3 to main when ready
- ✅ Phase 2 remains stable on main branch

### Step 1: Setup Development Branch
```bash
# For each Phase 2 service repository
cd yushan-microservices-user-service
git checkout main
git pull origin main

# Create Phase 3 development branch
git checkout -b phase3-kubernetes
git push -u origin phase3-kubernetes

# Create feature branch for first improvement
git checkout -b feature/rich-domain-model
```

### Step 2: Implement Domain-Driven Design (Feature Branch)
1. Convert Anemic Domain Models to Rich Domain Models
2. Implement Repository Pattern
3. Define Aggregate Boundaries
4. Implement Domain Events

### Step 3: Event-Driven Improvements (Feature Branch)
1. Create cache tables for cross-service data
2. Implement event listeners for cache updates
3. Add bootstrap mechanism for existing data
4. Implement idempotent event processing

### Step 4: Resilience & Observability (Feature Branch)
1. Add circuit breakers to all service calls
2. Implement rate limiting
3. Set up distributed tracing
4. Configure monitoring dashboards

### Step 5: Kubernetes Migration (Feature Branch)
1. Create Kubernetes manifests
2. Replace Eureka with Kubernetes Service Discovery
3. Configure auto-scaling
4. Set up service mesh (optional)

### Step 6: SAGA Pattern (Feature Branch)
1. Identify distributed transactions
2. Implement SAGA orchestrator or choreography
3. Add compensation logic
4. Test failure scenarios

### Step 7: AWS Deployment (Feature Branch)
1. Set up AWS EKS cluster
2. Migrate databases to RDS
3. Configure ElastiCache
4. Set up MSK for Kafka
5. Deploy services to EKS

**Production Merge**: After all features are merged to `phase3-kubernetes` branch and tested, merge to `main` for production deployment.

```bash
# When Phase 3 is ready for production
git checkout main
git merge phase3-kubernetes
git push origin main

# Tag Phase 3 release
git tag -a v3.0.0 -m "Phase 3: Kubernetes & AWS Deployment"
git push origin v3.0.0
```

---

## 📊 Comparison: Phase 2 vs Phase 3

| Aspect | Phase 2 | Phase 3 |
|--------|---------|---------|
| **Repository Strategy** | Separate repos | Same repos, branch-based |
| **Service Discovery** | Eureka | Kubernetes Native |
| **Domain Model** | Anemic | Rich Domain Model |
| **Data Access** | Direct Mapper | Repository Pattern |
| **Aggregate Boundaries** | Unclear | Well-defined |
| **Inter-Service Calls** | Many sync API calls | Event-driven cache tables |
| **Event Processing** | Basic | Idempotent |
| **Transactions** | Local only | SAGA Pattern |
| **Resilience** | Partial | Comprehensive |
| **Tracing** | None | Distributed Tracing |
| **Authentication** | Per-service JWT validation | Gateway-level JWT validation |
| **Orchestration** | Docker Compose | Kubernetes |
| **Cloud** | Digital Ocean | AWS |

## 🔄 Backporting Strategy

Since Phase 2 and Phase 3 share the same repositories, bugfixes can be easily backported:

**Backporting Bugfixes from Phase 3 to Phase 2**:
```bash
# Fix bug in Phase 3
git checkout phase3-kubernetes
git checkout -b fix/critical-bug
# ... fix bug ...
git commit -m "Fix: Critical bug in user service"
git checkout phase3-kubernetes
git merge fix/critical-bug

# Backport to Phase 2
git checkout main
git cherry-pick <commit-hash>
git push origin main
```

**Benefits**:
- ✅ Critical bugfixes can be applied to both phases
- ✅ Phase 2 production remains stable
- ✅ Phase 3 gets all improvements
- ✅ Single codebase, easier maintenance

## 🚢 Deployment Strategy

### Phase 2 Deployment (Current Production)
- **Branch**: `main`
- **Environment**: Digital Ocean
- **Deployment**: Uses `main` branch for production
- **Status**: ✅ Stable, continue using until Phase 3 is ready

### Phase 3 Deployment (Future Production)
- **Development Branch**: `phase3-kubernetes`
- **Production Branch**: `main` (after merge)
- **Environment**: AWS EKS
- **Deployment**: 
  - Development/Staging: Deploy from `phase3-kubernetes` branch
  - Production: Deploy from `main` branch after Phase 3 merge

### Parallel Deployment (Optional)
If you need to run both Phase 2 and Phase 3 simultaneously:

```bash
# Phase 2 deployment (Digital Ocean)
git checkout main
# Deploy to Digital Ocean

# Phase 3 deployment (AWS EKS)
git checkout phase3-kubernetes
# Deploy to AWS EKS for testing
```

**Note**: Once Phase 3 is production-ready and merged to `main`, Phase 2 deployment on Digital Ocean can be phased out.

---

## ✅ Implementation Checklist

### Domain-Driven Design
- [x] Convert Anemic Domain Models to Rich Domain Models
  - [x] user-service: User entity with business logic methods (changeStatus, upgradeToAuthor, promoteToAdmin, updateLastLogin, updateLastActive, etc.)
  - [x] content-service: Novel, Chapter, Category entities with business logic methods (changeStatus, publish, archive, updateContent, etc.)
  - [x] engagement-service: Comment, Review, Report, Vote entities with business logic methods (updateContent, incrementLikeCount, resolve, dismiss, etc.)
  - [x] gamification-service: (Skipped - mainly transaction records, minimal business logic needed)
  - [x] analytics-service: (Skipped - mainly tracking records, minimal business logic needed)
- [ ] Implement Repository Pattern for all aggregates
- [ ] Define clear aggregate boundaries
- [ ] Implement Domain Events (internal)
- [ ] Separate Domain Events from Integration Events

### Event-Driven Architecture
- [x] ~~Create cache tables for cross-service data~~ (Not needed - write operations optimized via Kafka)
- [x] ~~Implement event listeners for cache updates~~ (Not needed)
- [x] ~~Add bootstrap mechanism for existing data~~ (Not needed)
- [x] **Implement hybrid idempotent event processing** ✅ **COMPLETED**
  - Hybrid approach: Redis (fast checks <1ms) + Database table (persistent backup)
  - Created `processed_events` table in gamification-service, content-service, user-service
  - Implemented `IdempotencyService` for dual-layer idempotency checks
  - All Kafka event consumers now use hybrid idempotency:
    - gamification-service: UserEventListener, EngagementEventListener, InternalEventListener
    - content-service: EngagementEventListener (novel-rating-events, novel-vote-count-events)
    - user-service: UserActivityListener
  - Ensures idempotency even when Redis is restarted (data persisted in database)
- [x] **Optimize inter-service communication** ✅ **COMPLETED**
  - Migrated `updateNovelRatingAndCount` to Kafka (`novel-rating-events`)
  - Migrated `incrementVoteCount` to Kafka (`novel-vote-count-events`)
  - Response time improved from 600-700ms to <100ms

### Resilience & Observability
- [ ] Add circuit breakers to all service calls
- [ ] Implement rate limiting on critical endpoints
- [ ] Set up distributed tracing (Jaeger/Zipkin)
- [ ] Configure Prometheus metrics
- [ ] Set up Grafana dashboards

### Security Improvements
- [ ] Implement Gateway-Level JWT Authentication
- [ ] Add JWT validation filter to API Gateway
- [ ] Simplify microservice authentication (trust gateway-validated requests)
- [ ] Configure public endpoints whitelist
- [ ] Add fallback authentication for service-to-service calls
- [ ] Implement gateway high availability
- [ ] **Fix inactive user token validation issue** (choose one of options A-F)
  - [ ] Option A: Redis Cache - Full User Status
  - [ ] Option B: Redis Block List (only inactive users)
  - [ ] Option C: Database Table in Gateway
  - [ ] Option D: Direct Sync from User Service
  - [ ] Option E: Hybrid - Direct Redis Update + Kafka Event + Local Cache
  - [ ] Option F: Hybrid - Database Table + Local Cache

### Kubernetes & Cloud
- [ ] Create Kubernetes manifests for all services
- [ ] Replace Eureka with Kubernetes Service Discovery
- [ ] Configure auto-scaling (HPA)
- [ ] Set up service mesh (Istio/Linkerd) - optional
- [ ] Migrate to AWS EKS
- [ ] Configure AWS RDS
- [ ] Set up AWS ElastiCache
- [ ] Configure AWS MSK for Kafka
- [ ] Set up AWS S3 for file storage

### SAGA Pattern
- [ ] Identify distributed transactions
- [ ] Design SAGA flows (choreography or orchestration)
- [ ] Implement SAGA orchestrator/participants
- [ ] Add compensation logic
- [ ] Test failure and recovery scenarios

---

## 📚 References

- [Domain-Driven Design](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- [Repository Pattern](https://martinfowler.com/eaaCatalog/repository.html)
- [SAGA Pattern](https://microservices.io/patterns/data/saga.html)
- [Kubernetes Service Discovery](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Distributed Tracing](https://opentracing.io/)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)

---

**Last Updated**: November 2025 - Rich Domain Model refactoring + Inter-service communication optimization + Hybrid idempotency implementation completed (Redis + Database table for all event consumers)

