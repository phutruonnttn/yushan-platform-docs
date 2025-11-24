# Phase 3: Kubernetes & AWS Deployment

> 🚀 **Advanced microservices architecture with Kubernetes orchestration, distributed tracing, Saga pattern, and cloud-native improvements**

## 📋 Overview

**Status**: 🔄 In Progress (40% Complete) | **Target**: AWS EKS Deployment

**Progress**: 
- Rich Domain Model refactoring completed for 3 services (user-service, content-service, engagement-service)
- Inter-service communication optimization completed (blocking write operations migrated to Kafka events)
- Hybrid idempotency implementation completed (Redis + Database table for all event consumers)
- Repository Pattern implementation completed for all services (user-service, content-service, engagement-service, gamification-service, analytics-service)
- Aggregate boundaries and Domain Events implementation completed for content-service (Novel and Chapter aggregates separated, using internal Domain Events)

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
- ✅ Repository Pattern implementation (all services completed)
  - ✅ user-service: UserRepository interface and MyBatisUserRepository implementation (341 tests passing: 309 unit + 32 integration)
  - ✅ content-service: NovelRepository, ChapterRepository, CategoryRepository with MyBatis implementations + Elasticsearch repositories
  - ✅ engagement-service: CommentRepository, ReviewRepository, VoteRepository, ReportRepository with MyBatis implementations
  - ✅ gamification-service: UserProgressRepository with MyBatis implementation
  - ✅ analytics-service: AnalyticsRepository, HistoryRepository with MyBatis implementations
  - ✅ All services now depend on Repository interfaces instead of Mapper directly
- [x] Aggregate boundaries and Domain Events ✅ **COMPLETED (content-service)**
  - [x] content-service: Novel and Chapter aggregates separated
    - Defined clear aggregate boundaries (Novel aggregate root, Chapter aggregate root)
    - Implemented internal Domain Events (`ChapterStatisticsChangedEvent`)
    - Replaced direct cross-aggregate calls with Domain Event publishing
    - Created `ChapterDomainEventPublisher` and `ChapterDomainEventListener` for event-driven communication
    - All tests passing (571 unit + 53 integration tests)
  - [x] user-service: Acceptable as-is (Library and NovelLibrary are child entities of User aggregate, no cross-aggregate issues)
  - [x] gamification-service: Acceptable as-is (UserProgress is well-defined aggregate root, no cross-aggregate issues)
  - [x] engagement-service: Acceptable as-is (Comment, Review, Vote aggregates are well-separated, no cross-aggregate issues)
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

### 2. Repository Pattern Implementation ✅ **COMPLETED**

**Problem**: Services directly call mappers, violating separation of concerns.

**Solution**: Introduce Repository pattern to abstract data access.

**Status**: 
- ✅ **user-service**: Repository Pattern fully implemented
  - Created `UserRepository` interface and `MyBatisUserRepository` implementation
  - Migrated all controllers (UserController, AuthController, AuthorController) to use `UserRepository`
  - Migrated security components (JwtAuthenticationFilter, CustomUserDetailsService) to use `UserRepository`
  - Fixed `save()` method to correctly handle insert/update logic
  - Updated all tests (unit and integration) to mock `UserRepository` instead of `UserMapper`
  - All 341 tests passing (309 unit + 32 integration)

- ✅ **content-service**: Repository Pattern fully implemented
  - `NovelRepository` interface with `MyBatisNovelRepository` implementation
  - `ChapterRepository` interface with `MyBatisChapterRepository` implementation
  - `CategoryRepository` interface with `MyBatisCategoryRepository` implementation
  - Elasticsearch repositories: `NovelElasticsearchRepository`, `ChapterElasticsearchRepository`
  - All services (NovelService, ChapterService, CategoryService) use Repository interfaces

- ✅ **engagement-service**: Repository Pattern fully implemented
  - `CommentRepository` interface with `MyBatisCommentRepository` implementation
  - `ReviewRepository` interface with `MyBatisReviewRepository` implementation
  - `VoteRepository` interface with `MyBatisVoteRepository` implementation
  - `ReportRepository` interface with `MyBatisReportRepository` implementation
  - All services (CommentService, ReviewService, VoteService, ReportService) use Repository interfaces

- ✅ **gamification-service**: Repository Pattern fully implemented
  - `UserProgressRepository` interface with `MyBatisUserProgressRepository` implementation
  - GamificationService uses `UserProgressRepository` instead of mapper directly

- ✅ **analytics-service**: Repository Pattern fully implemented
  - `AnalyticsRepository` interface with `MyBatisAnalyticsRepository` implementation
  - `HistoryRepository` interface with `MyBatisHistoryRepository` implementation
  - All services (AnalyticsService, HistoryService) use Repository interfaces

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

### 3. Aggregate Boundaries & Domain Events ✅ **COMPLETED (content-service)**

**Problem**: Services cross aggregate boundaries with direct calls, violating DDD principles.

**Solution**: Define clear aggregate boundaries and use Domain Events for inter-aggregate communication.

**Status**: ✅ **COMPLETED for content-service**
- Novel and Chapter are now separate aggregates with clear boundaries
- Cross-aggregate communication uses internal Domain Events (Spring ApplicationEventPublisher)
- All chapter operations that affect Novel statistics publish `ChapterStatisticsChangedEvent`
- `ChapterDomainEventListener` handles events and updates Novel statistics within the same transaction
- All tests passing (571 unit + 53 integration tests)

**Aggregate Boundaries**:
```
Content Service:
├── Novel Aggregate (root) ✅ Refactored
│   ├── Novel entity
│   └── Category references
├── Chapter Aggregate (root) ✅ Refactored
│   └── Chapter entity
└── Domain Events: ChapterStatisticsChangedEvent (internal, same transaction)

User Service:
├── User Aggregate (root) ✅ Acceptable as-is
│   ├── User entity
│   ├── Library (child entity)
│   └── NovelLibrary (child entity)
└── No cross-aggregate issues

Engagement Service:
├── Comment Aggregate (root) ✅ Acceptable as-is
│   └── Comment entity
├── Review Aggregate (root) ✅ Acceptable as-is
│   └── Review entity
├── Vote Aggregate (root) ✅ Acceptable as-is
│   └── Vote entity
└── No cross-aggregate issues

Gamification Service:
├── UserProgress Aggregate (root) ✅ Acceptable as-is
│   ├── UserProgress entity
│   ├── Achievement entities
│   └── Transaction entities
└── No cross-aggregate issues
```

**Domain Events vs Integration Events**:
```java
// ✅ Domain Event (internal to aggregate, same transaction)
// Example: ChapterStatisticsChangedEvent in content-service
@Component
public class ChapterDomainEventPublisher {
    private final ApplicationEventPublisher eventPublisher;
    
    public void publishChapterStatisticsChanged(Integer novelId) {
        eventPublisher.publishEvent(new ChapterStatisticsChangedEvent(novelId));
    }
}

@EventListener
@Transactional
public class ChapterDomainEventListener {
    public void handleChapterStatisticsChanged(ChapterStatisticsChangedEvent event) {
        // Update Novel statistics within same transaction
        novelService.updateNovelStatistics(event.getNovelId(), ...);
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

**Implementation Example (content-service)**:
```java
// ✅ Before: Direct cross-aggregate call (violates DDD)
@Service
public class ChapterService {
    public void createChapter(ChapterCreateRequestDTO req) {
        Chapter chapter = new Chapter(...);
        chapterRepository.save(chapter);
        novelService.updateNovelStatistics(req.getNovelId()); // ❌ Direct call
    }
}

// ✅ After: Domain Event (respects aggregate boundaries)
@Service
public class ChapterService {
    private final ChapterDomainEventPublisher eventPublisher;
    
    public void createChapter(ChapterCreateRequestDTO req) {
        Chapter chapter = new Chapter(...);
        chapterRepository.save(chapter);
        eventPublisher.publishChapterStatisticsChanged(req.getNovelId()); // ✅ Event
    }
}

// Event Listener handles the update
@EventListener
@Transactional
public class ChapterDomainEventListener {
    public void handleChapterStatisticsChanged(ChapterStatisticsChangedEvent event) {
        // Recalculate and update Novel statistics
        long chapterCount = chapterRepository.countPublishedByNovelId(event.getNovelId());
        long wordCount = chapterRepository.sumPublishedWordCountByNovelId(event.getNovelId());
        novelService.updateNovelStatistics(event.getNovelId(), chapterCount, wordCount);
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

### Phase 3 Development (Separate Repositories Cloned from Phase 2)

**Decision**: Phase 3 is developed in separate repositories cloned from Phase 2 original repositories. This allows independent development while maintaining the ability to reference the original Phase 2 codebase.

**Phase 2 Original Repositories** (NUS-ISS team):
- [yushan-user-service](https://github.com/maugus0/yushan-user-service)
- [yushan-content-service](https://github.com/maugus0/yushan-content-service)
- [yushan-engagement-service](https://github.com/maugus0/yushan-engagement-service)
- [yushan-gamification-service](https://github.com/maugus0/yushan-gamification-service)
- [yushan-analytics-service](https://github.com/maugus0/yushan-analytics-service)
- [yushan-api-gateway](https://github.com/maugus0/yushan-api-gateway)
- [yushan-config-server](https://github.com/maugus0/yushan-config-server)
- [yushan-platform-service-registry](https://github.com/maugus0/yushan-platform-service-registry)

**Phase 3 Development Repositories** (phutruonnttn - cloned from Phase 2):
- [yushan-microservices-user-service](https://github.com/phutruonnttn/yushan-microservices-user-service)
- [yushan-microservices-content-service](https://github.com/phutruonnttn/yushan-microservices-content-service)
- [yushan-microservices-engagement-service](https://github.com/phutruonnttn/yushan-microservices-engagement-service)
- [yushan-microservices-gamification-service](https://github.com/phutruonnttn/yushan-microservices-gamification-service)
- [yushan-microservices-analytics-service](https://github.com/phutruonnttn/yushan-microservices-analytics-service)
- [yushan-microservices-api-gateway](https://github.com/phutruonnttn/yushan-microservices-api-gateway)
- [yushan-microservices-config-server](https://github.com/phutruonnttn/yushan-microservices-config-server)
- [yushan-microservices-service-registry](https://github.com/phutruonnttn/yushan-microservices-service-registry)

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

### Repository Strategy

**Approach**: Phase 3 is developed in separate repositories cloned from Phase 2 original repositories. Each Phase 3 repository maintains its own git history and can be developed independently.

**Repository Structure**:
```
Phase 2 (Original - NUS ISS team):
├── yushan-user-service (main branch - production)
├── yushan-content-service (main branch - production)
├── yushan-engagement-service (main branch - production)
├── yushan-gamification-service (main branch - production)
├── yushan-analytics-service (main branch - production)
├── yushan-api-gateway (main branch - production)
├── yushan-config-server (main branch - production)
└── yushan-platform-service-registry (main branch - production)

Phase 3 (Development - phutruonnttn):
├── yushan-microservices-user-service (cloned from Phase 2)
│   ├── main (Phase 3 development)
│   └── feature/* branches
├── yushan-microservices-content-service (cloned from Phase 2)
├── yushan-microservices-engagement-service (cloned from Phase 2)
├── yushan-microservices-gamification-service (cloned from Phase 2)
├── yushan-microservices-analytics-service (cloned from Phase 2)
├── yushan-microservices-api-gateway (cloned from Phase 2)
├── yushan-microservices-config-server (cloned from Phase 2)
└── yushan-microservices-service-registry (cloned from Phase 2)
```

**Workflow**:
```bash
# Phase 3 repositories are cloned from Phase 2
# Each Phase 3 repository is developed independently

# Example: Working on user-service Phase 3
cd yushan-microservices-user-service
git checkout main

# Create feature branch for Phase 3 improvements
git checkout -b feature/rich-domain-model
# ... implement feature ...
git commit -m "feat: Implement Rich Domain Model"
git push origin feature/rich-domain-model

# Merge to main when ready
git checkout main
git merge feature/rich-domain-model
git push origin main
```

**Benefits**:
- ✅ Independent development - Phase 3 doesn't affect Phase 2 production
- ✅ Clear separation between Phase 2 (stable) and Phase 3 (development)
- ✅ Can reference Phase 2 codebase when needed
- ✅ Phase 2 remains stable and production-ready
- ✅ Phase 3 can experiment with breaking changes

### Step 1: Clone Phase 2 Repositories
```bash
# Clone Phase 2 repositories to create Phase 3 development repos
# (Already done - repositories are at phutruonnttn/yushan-microservices-*)

# Example structure:
# Phase 2: maugus0/yushan-user-service
# Phase 3: phutruonnttn/yushan-microservices-user-service (cloned)
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

**Production Deployment**: After all Phase 3 features are implemented and tested in the Phase 3 repositories, they can be deployed to AWS EKS for production.

```bash
# Phase 3 repositories are ready for production deployment
# Each service is deployed independently from its Phase 3 repository

# Tag Phase 3 release
git tag -a v3.0.0 -m "Phase 3: Kubernetes & AWS Deployment"
git push origin v3.0.0
```

---

## 📊 Comparison: Phase 2 vs Phase 3

| Aspect | Phase 2 | Phase 3 |
|--------|---------|---------|
| **Repository Strategy** | Original repos (maugus0) | Separate repos cloned from Phase 2 (phutruonnttn) |
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

## 🔄 Bugfix Strategy

Since Phase 2 and Phase 3 are in separate repositories, bugfixes need to be applied independently:

**Applying Bugfixes**:

**Option 1: Fix in Phase 2 (if bug exists in production)**
```bash
# Fix in Phase 2 original repository
cd yushan-user-service  # Phase 2 repo (maugus0)
git checkout main
git checkout -b fix/critical-bug
# ... fix bug ...
git commit -m "Fix: Critical bug in user service"
git push origin fix/critical-bug
# Create PR to Phase 2 main branch
```

**Option 2: Fix in Phase 3 (if bug discovered during Phase 3 development)**
```bash
# Fix in Phase 3 repository
cd yushan-microservices-user-service  # Phase 3 repo (phutruonnttn)
git checkout main
git checkout -b fix/critical-bug
# ... fix bug ...
git commit -m "Fix: Critical bug in user service"
git push origin fix/critical-bug
# Merge to Phase 3 main branch
```

**Option 3: Backport from Phase 3 to Phase 2 (if applicable)**
```bash
# If bugfix in Phase 3 is also needed in Phase 2
# Manually apply the same fix to Phase 2 repository
# Or cherry-pick if the change is compatible
cd yushan-user-service  # Phase 2 repo
git checkout main
# Manually apply the fix or cherry-pick if compatible
```

**Considerations**:
- ⚠️ Phase 2 and Phase 3 are separate repositories - no automatic sync
- ⚠️ Bugfixes need to be applied manually to both if needed
- ✅ Phase 2 production remains stable and independent
- ✅ Phase 3 can experiment without affecting Phase 2
- ✅ Can reference Phase 2 codebase when needed

## 🚢 Deployment Strategy

### Phase 2 Deployment (Current Production)
- **Repository**: Original repositories (maugus0/yushan-*)
- **Branch**: `main`
- **Environment**: Digital Ocean
- **Deployment**: Uses `main` branch from Phase 2 repositories for production
- **Status**: ✅ Stable, continue using until Phase 3 is ready

### Phase 3 Deployment (Future Production)
- **Repository**: Development repositories (phutruonnttn/yushan-microservices-*)
- **Branch**: `main` (Phase 3 development)
- **Environment**: AWS EKS
- **Deployment**: 
  - Development/Staging: Deploy from Phase 3 repositories `main` branch
  - Production: Deploy from Phase 3 repositories `main` branch when ready

### Parallel Deployment
Phase 2 and Phase 3 can run simultaneously as they are in separate repositories:

```bash
# Phase 2 deployment (Digital Ocean)
cd yushan-user-service  # Phase 2 repo (maugus0)
git checkout main
# Deploy to Digital Ocean from Phase 2 repository

# Phase 3 deployment (AWS EKS)
cd yushan-microservices-user-service  # Phase 3 repo (phutruonnttn)
git checkout main
# Deploy to AWS EKS from Phase 3 repository
```

**Note**: 
- Phase 2 and Phase 3 are completely independent deployments
- Phase 2 continues running on Digital Ocean (stable production)
- Phase 3 will run on AWS EKS (new production when ready)
- Both can coexist during migration period

---

## ✅ Implementation Checklist

### Domain-Driven Design
- [x] Convert Anemic Domain Models to Rich Domain Models
  - [x] user-service: User entity with business logic methods (changeStatus, upgradeToAuthor, promoteToAdmin, updateLastLogin, updateLastActive, etc.)
  - [x] content-service: Novel, Chapter, Category entities with business logic methods (changeStatus, publish, archive, updateContent, etc.)
  - [x] engagement-service: Comment, Review, Report, Vote entities with business logic methods (updateContent, incrementLikeCount, resolve, dismiss, etc.)
  - [x] gamification-service: (Skipped - mainly transaction records, minimal business logic needed)
  - [x] analytics-service: (Skipped - mainly tracking records, minimal business logic needed)
- [x] Implement Repository Pattern for all aggregates ✅ **COMPLETED**
  - [x] user-service: UserRepository interface and MyBatisUserRepository implementation
    - All controllers and security components migrated from UserMapper to UserRepository
    - All tests updated and passing (341 tests: 309 unit + 32 integration)
  - [x] content-service: NovelRepository, ChapterRepository, CategoryRepository with MyBatis implementations + Elasticsearch repositories
    - All services (NovelService, ChapterService, CategoryService) use Repository interfaces
  - [x] engagement-service: CommentRepository, ReviewRepository, VoteRepository, ReportRepository with MyBatis implementations
    - All services (CommentService, ReviewService, VoteService, ReportService) use Repository interfaces
  - [x] gamification-service: UserProgressRepository with MyBatis implementation
    - GamificationService uses UserProgressRepository
  - [x] analytics-service: AnalyticsRepository, HistoryRepository with MyBatis implementations
    - All services (AnalyticsService, HistoryService) use Repository interfaces
- [x] Define clear aggregate boundaries ✅ **COMPLETED**
  - [x] content-service: Novel and Chapter are separate aggregates with clear boundaries
  - [x] user-service: Acceptable as-is (Library and NovelLibrary are child entities of User aggregate)
  - [x] gamification-service: Acceptable as-is (UserProgress is well-defined aggregate root)
  - [x] engagement-service: Acceptable as-is (Comment, Review, Vote aggregates are well-separated)
- [x] Implement Domain Events (internal) ✅ **COMPLETED (content-service only)**
  - [x] content-service: `ChapterStatisticsChangedEvent` for Novel statistics updates
  - [x] content-service: `ChapterDomainEventPublisher` and `ChapterDomainEventListener` for event handling
  - [x] content-service: All chapter operations (create, update, delete, publish) publish Domain Events
  - [x] user-service: Not needed (no cross-aggregate issues)
  - [x] gamification-service: Not needed (no cross-aggregate issues)
  - [x] engagement-service: Not needed (no cross-aggregate issues)
- [x] Separate Domain Events from Integration Events ✅ **COMPLETED (content-service)**
  - [x] content-service: Domain Events (internal, same transaction) vs Integration Events (Kafka, cross-service)
  - [x] content-service: `ChapterStatisticsChangedEvent` is Domain Event (Spring ApplicationEventPublisher)
  - [x] content-service: Kafka events remain as Integration Events for cross-service communication
  - [x] Other services: No changes needed (no cross-aggregate issues requiring Domain Events)

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

**Last Updated**: November 2025 - Rich Domain Model refactoring + Inter-service communication optimization + Hybrid idempotency implementation + Repository Pattern (all services) + Aggregate Boundaries & Domain Events (content-service) completed. Content-service now uses internal Domain Events for cross-aggregate communication instead of direct calls.

