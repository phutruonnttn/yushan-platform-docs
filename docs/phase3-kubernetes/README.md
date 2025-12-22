# Phase 3: Kubernetes & AWS Deployment

> 🚀 **Advanced microservices architecture with Kubernetes orchestration, distributed tracing, Saga pattern, and cloud-native improvements**

## 📋 Overview

**Status**: 🔄 In Progress (75% Complete) | **Target**: AWS EKS Deployment

**Progress**: 
- Rich Domain Model refactoring completed for 3 services (user-service, content-service, engagement-service)
- Inter-service communication optimization completed (blocking write operations migrated to Kafka events)
- Hybrid idempotency implementation completed (Redis + Database table for all event consumers)
- Repository Pattern implementation completed for all services (user-service, content-service, engagement-service, gamification-service, analytics-service)
- Aggregate boundaries and Domain Events implementation completed for content-service (Novel and Chapter aggregates separated, using internal Domain Events)
- Kafka Events Transaction Boundary Fix completed for all services (events now publish after transaction commit)
- Gateway-Level JWT Authentication with HMAC Signature completed for all services (centralized validation with cryptographic signature protection to prevent header forgery attacks)
- Inactive User Token Validation issue resolved (Option B - Redis Block List implemented with real-time updates via Kafka events)
- Circuit Breakers & Rate Limiters completed (comprehensive coverage: engagement-service, analytics-service, api-gateway)
- SAGA Pattern for distributed transactions completed (Vote Creation Flow with Choreography pattern, Yuan Reservation System, balance check at reserve time, compensation logic)
- ✅ **AWS Infrastructure Deployment (Task 1) Complete** - All infrastructure deployed (EKS, RDS, ElastiCache, Kafka, S3, ALB), all 6 microservices deployed and running, Kafka installed and configured (3 brokers with Zookeeper), all APIs functional with Kafka enabled, all Swagger UIs accessible (6/6)

Phase 3 represents a significant evolution from Phase 2, focusing on:
- **Cloud-Native Architecture**: Kubernetes-native service discovery and orchestration
- **Domain-Driven Design**: Rich domain models with proper aggregate boundaries
- **Event-Driven Excellence**: Improved eventual consistency patterns
- **Distributed Systems**: Saga pattern, distributed tracing, service mesh
- **Production Hardening**: Circuit breakers, rate limiting, idempotency

## 🎯 Planned Features

### Infrastructure & Deployment
- ✅ Kubernetes orchestration (AWS EKS) - **Deployed and Running**
- ✅ Kubernetes-native service discovery (replacing Eureka) - **Implemented**
- ✅ Auto-scaling and container orchestration - **EKS Cluster with 2 nodes (t3.small)**
- ✅ AWS cloud services integration - **Complete (RDS, ElastiCache, S3, ALB, ECR)**
- ✅ EC2 Kafka Cluster - **3 brokers (t3.small, Multi-AZ) deployed and running**
- ⬜ Service mesh (Istio/Linkerd)
- ⬜ Distributed tracing (Jaeger/Zipkin)

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
- ✅ Aggregate boundaries and Domain Events (content-service)
  - ✅ content-service: Novel and Chapter aggregates separated
    - Defined clear aggregate boundaries (Novel aggregate root, Chapter aggregate root)
    - Implemented internal Domain Events (`ChapterStatisticsChangedEvent`)
    - Replaced direct cross-aggregate calls with Domain Event publishing
    - Created `ChapterDomainEventPublisher` and `ChapterDomainEventListener` for event-driven communication
    - All tests passing (571 unit + 53 integration tests)
  - ✅ user-service: Acceptable as-is (Library and NovelLibrary are child entities of User aggregate, no cross-aggregate issues)
  - ✅ gamification-service: Acceptable as-is (UserProgress is well-defined aggregate root, no cross-aggregate issues)
  - ✅ engagement-service: Acceptable as-is (Comment, Review, Vote aggregates are well-separated, no cross-aggregate issues)
- ✅ **SAGA pattern for distributed transactions** (Vote Creation Flow)
  - ✅ Implemented Choreography SAGA pattern for Vote Creation Flow
    - Yuan Reservation System with reservation table in gamification-service
    - Balance check at reserve time (fail fast pattern)
    - Multi-step transaction flow: Reserve Yuan → Create Vote → Confirm Yuan
    - Compensation logic for rollback on failures
    - Scheduled cleanup job for expired reservations
  - ✅ engagement-service: VoteSagaListener for handling SAGA events
  - ✅ gamification-service: VoteSagaListener for Yuan reservation and confirmation
  - ✅ Hybrid idempotency for all SAGA events (prevents duplicate processing)
  - ✅ Feature flag for gradual rollout (`saga.vote-creation.enabled`)
  - ✅ API contract fix: Balance check before SAGA starts (returns 400 when insufficient balance)
  - ✅ Tested and verified: Works correctly with sufficient balance and properly rejects when balance = 0

### Resilience & Observability
- ✅ **Circuit Breakers** (comprehensive coverage)
  - ✅ engagement-service: 3 Feign clients (ContentServiceClient, UserServiceClient, GamificationServiceClient)
  - ✅ analytics-service: 4 Feign clients (ContentServiceClient, UserServiceClient, GamificationServiceClient, EngagementServiceClient)
  - ✅ api-gateway: 1 Feign client (UserServiceClient)
  - ✅ user-service: 1 Feign client (ContentServiceClient) - already had Circuit Breaker
  - ✅ All Feign client methods have fallback methods for graceful degradation
- ✅ **Rate Limiting**
  - ✅ engagement-service: Rate limiter on comment/review creation endpoints (10/60s and 5/60s)
  - ✅ api-gateway: Global rate limiter (100 requests/60s) via `RateLimiterGatewayFilter`
- ⬜ Distributed tracing (Jaeger/Zipkin)
- ⬜ Enhanced monitoring and observability

### Security Improvements
- ✅ Gateway-level JWT authentication (centralized validation)
- ✅ Token validation at API Gateway (reduce microservice load)
- ✅ Consistent security policy across all services
- ✅ HMAC Signature implementation (prevent header forgery attacks)
  - ✅ Gateway generates HMAC-SHA256 signatures for validated requests
  - ✅ Services verify HMAC signatures before trusting gateway headers
  - ✅ Timestamp validation prevents replay attacks (5-minute tolerance)
  - ✅ Constant-time comparison prevents timing attacks
  - ✅ Shared secret configuration (`GATEWAY_HMAC_SECRET`) across Gateway and all services
- ✅ Inactive user token validation (fix security issue where inactive users can still use tokens)

## 🏗️ Architecture Improvements

### 1. Rich Domain Model (Fixing Anemic Domain Model)

**Problem**: Current entities are Anemic Domain Models - they only contain data without business logic.

**Solution**: Move business logic into domain entities and aggregates (services call rich domain methods instead of setting fields directly).

**Benefits**:
- Encapsulation of business rules
- Self-documenting domain logic
- Easier to test and maintain
- Prevents invalid state transitions

---

### 2. Repository Pattern Implementation

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

### 3. Aggregate Boundaries & Domain Events (content-service)

**Problem**: Services cross aggregate boundaries with direct calls, violating DDD principles.

**Solution**: Define clear aggregate boundaries and use Domain Events for inter-aggregate communication.

**Status**: ✅ Completed for content-service
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

### 4. Inter-Service Communication Optimization

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

### 5. Hybrid Idempotency for Event Consumption

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

**Solution**: Comprehensive resilience patterns implemented across all services.

**Status**: ✅ Completed
- ✅ engagement-service: Circuit Breaker for 3 Feign clients + Rate Limiter on comment/review creation
- ✅ **analytics-service**: Circuit Breaker for 4 Feign clients
- ✅ **api-gateway**: Circuit Breaker for UserServiceClient + Global Rate Limiter
- ✅ **user-service**: Already had Circuit Breaker (no changes needed)

**Circuit Breaker Implementation**:

**Key Implementation Details**:
- Circuit Breakers are enabled via `spring.cloud.openfeign.circuitbreaker.enabled=true` in configuration
- **No `@CircuitBreaker` annotations on Feign client methods** - Feign's automatic integration handles Circuit Breaker wrapping
- Fallback classes (implementing Feign client interfaces) are used for graceful degradation
- Circuit Breaker state can be monitored via Actuator endpoints (`/actuator/health` and `/actuator/metrics`)

**Engagement Service**:
```java
@FeignClient(
    name = "content-service", 
    url = "${services.content.url:http://yushan-content-service:8082}",
    fallback = ContentServiceClient.ContentServiceFallback.class
)
public interface ContentServiceClient {
    
    @GetMapping("/api/v1/novels/{novelId}")
    // No @CircuitBreaker annotation - handled by Feign's automatic integration
    ApiResponse<NovelDetailResponseDTO> getNovelById(@PathVariable("novelId") Integer novelId);
    
    @Component
    class ContentServiceFallback implements ContentServiceClient {
        @Override
        public ApiResponse<NovelDetailResponseDTO> getNovelById(Integer novelId) {
            log.error("Circuit breaker opened for content-service. Falling back for getNovelById request with {} id.", novelId);
            return ApiResponse.error(503, "Content service temporarily unavailable", null);
        }
        // Fallback implementations for all other methods
    }
}
```

**Analytics Service**:
- 4 Feign clients with Circuit Breaker: ContentServiceClient (11 methods), UserServiceClient (3 methods), GamificationServiceClient (3 methods), EngagementServiceClient (4 methods)
- All methods have fallback methods and fallback classes

**API Gateway**:
- UserServiceClient with Circuit Breaker for `getBlockedUsers()` method
- Fallback uses cached blocklist data when User Service is unavailable

**Configuration**:
```yaml
resilience4j:
  circuitbreaker:
    instances:
      content-service:
        registerHealthIndicator: true
        slidingWindowType: COUNT_BASED
        slidingWindowSize: 20
        minimumNumberOfCalls: 10
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 5
        automaticTransitionFromOpenToHalfOpenEnabled: true
        slowCallRateThreshold: 100
        slowCallDurationThreshold: 5s
  retry:
    instances:
      content-service:
        maxAttempts: 3
        waitDuration: 1000ms
        retryExceptions:
          - java.net.SocketTimeoutException
          - java.util.concurrent.TimeoutException
```

**Rate Limiter Implementation**:

**Engagement Service**:
- Uses `RateLimiterInterceptor` (HandlerInterceptor pattern) to handle `@RateLimiter` annotations
- Interceptor checks for `@RateLimiter` annotation on controller methods and applies rate limiting before method execution
- Similar to API Gateway's `RateLimiterGatewayFilter` but for Spring MVC controllers

```java
// Controller with @RateLimiter annotation
@RestController
@RequestMapping("/api/v1/comments")
public class CommentController {
    
    @PostMapping
    @PreAuthorize("hasAnyRole('USER','AUTHOR','ADMIN')")
    @RateLimiter(name = "comment-creation")
    public ApiResponse<CommentResponseDTO> createComment(@RequestBody CommentCreateRequestDTO request) {
        // Rate limited: 10 requests/60s (enforced by RateLimiterInterceptor)
        return ApiResponse.success("Comment created successfully", commentService.createComment(userId, request));
    }
}

@RestController
@RequestMapping("/api/v1/reviews")
public class ReviewController {
    
    @PostMapping
    @PreAuthorize("hasAnyRole('USER','AUTHOR','ADMIN')")
    @RateLimiter(name = "review-creation")
    public ApiResponse<ReviewResponseDTO> createReview(@RequestBody ReviewCreateRequestDTO request) {
        // Rate limited: 5 requests/60s (enforced by RateLimiterInterceptor)
        return ApiResponse.success("Review created successfully", reviewService.createReview(userId, request));
    }
}

// RateLimiterInterceptor implementation
@Component
public class RateLimiterInterceptor implements HandlerInterceptor {
    
    @Autowired
    private RateLimiterRegistry rateLimiterRegistry;
    
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) {
        // Check for @RateLimiter annotation and apply rate limiting
        // Returns false (stops request) if rate limit exceeded (HTTP 429)
        // Returns true (continues) if permit acquired
    }
}
```

**API Gateway**:
```java
@Component
public class RateLimiterGatewayFilter implements GlobalFilter, Ordered {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        RateLimiter rateLimiter = rateLimiterRegistry.rateLimiter("api-gateway-global");
        boolean permitAcquired = rateLimiter.acquirePermission();
        
        if (!permitAcquired) {
            return rateLimitExceeded(exchange); // 429 Too Many Requests
        }
        return chain.filter(exchange);
    }
}
```

**Rate Limiter Configuration**:
```yaml
resilience4j:
  ratelimiter:
    instances:
      comment-creation:
        limitForPeriod: 10
        limitRefreshPeriod: 60s
        timeoutDuration: 0ms
      review-creation:
        limitForPeriod: 5
        limitRefreshPeriod: 60s
        timeoutDuration: 0ms
      api-gateway-global:
        limitForPeriod: 100
        limitRefreshPeriod: 60s
        timeoutDuration: 0ms
```

**Conflict Resolution: Bootstrap Retry vs Circuit Breaker (API Gateway)**:

The API Gateway's `UserBlocklistBootstrapService` has a custom exponential backoff retry mechanism for syncing blocked users. When the `UserServiceClient` (protected by Circuit Breaker) fails and the circuit opens, its fallback method returns an `ApiResponse.success()` with an empty list and a message like "User service temporarily unavailable...". 

**Problem**: The bootstrap service would interpret this as a successful (though empty) response and stop retrying, effectively bypassing its exponential backoff logic.

**Solution**: The `fetchBlockedUsers()` method in `UserBlocklistBootstrapService` explicitly detects Circuit Breaker fallback responses by checking the response message. If the message contains "temporarily unavailable", it throws a `RuntimeException`, which is then caught by `syncBlocklistWithRetry()`, allowing the custom exponential backoff retry logic to proceed as intended.

**Implementation**:
```java
private Set<UUID> fetchBlockedUsers() {
    ApiResponse<List<UUID>> response = userServiceClient.getBlockedUsers();
    
    // Detect Circuit Breaker fallback response
    String message = response.getMessage();
    if (message != null && message.contains("temporarily unavailable")) {
        log.warn("Circuit Breaker fallback detected - User Service is unavailable");
        throw new RuntimeException("User Service unavailable (Circuit Breaker fallback)");
    }
    
    // Process normal response...
}
```

**Benefits**:
- ✅ **Fault Isolation**: Prevents cascading failures across services
- ✅ **Graceful Degradation**: Fallback methods ensure services continue operating
- ✅ **Spam Prevention**: Rate limiting prevents abuse on critical endpoints
- ✅ **DDoS Protection**: Global rate limiter at gateway protects all downstream services
- ✅ **Resource Protection**: Limits calls to failing services, reducing resource consumption
- ✅ **Bootstrap Resilience**: Custom retry logic works correctly with Circuit Breaker fallbacks

**Testing & Verification**:
- ✅ **Engagement Service**: Circuit Breaker opens correctly when content-service is down (state = 1.0 = OPEN), fallback methods invoked, logs confirm "Circuit breaker opened"
- ✅ **Engagement Service**: Rate Limiter triggers correctly (HTTP 429) when limits exceeded (comment: 10/60s, review: 5/60s)
- ✅ **Analytics Service**: Circuit Breaker opens correctly when upstream services are down, fallback data returned (totalNovels: 0, totalComments: 0)
- ✅ **API Gateway**: Circuit Breaker opens correctly when user-service is down (state = 1.0 = OPEN), bootstrap retry continues with exponential backoff

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

### 9. SAGA Pattern for Distributed Transactions (Vote Creation Flow)

**Problem**: Vote creation requires atomicity across Engagement Service (vote record) and Gamification Service (Yuan deduction). The previous flow had risks:
- Vote could be created without Yuan being deducted (if Gamification Service was down)
- Kafka event loss could cause data inconsistency
- No rollback mechanism if vote creation failed after Yuan deduction

**Solution**: Implemented Choreography SAGA pattern for Vote Creation Flow with Yuan Reservation System.

**Implementation Overview**:
- **Pattern**: Choreography SAGA (event-driven, decentralized)
- **Use Case**: Vote Creation Flow (Engagement Service + Gamification Service)
- **Flow**: Reserve Yuan → Create Vote → Confirm Yuan Deduction + Award EXP
- **Balance Check**: Performed at reserve time (fail fast pattern)
- **Compensation**: Automatic rollback via reservation release

**SAGA Flow**:
```
1. User requests vote → Engagement Service
2. Balance check (synchronous) → Gamification Service (if fails, return 400)
3. Publish VoteSagaStartEvent → Kafka topic "vote-saga.start"
4. Gamification Service: Reserve Yuan (status: RESERVED, expires in 5 minutes)
5. Publish VoteSagaYuanReservedEvent → Kafka topic "vote-saga.yuan-reserved"
6. Engagement Service: Create vote in database
7. Publish VoteSagaVoteCreatedEvent → Kafka topic "vote-saga.vote-created"
8. Gamification Service: Confirm Yuan reservation (deduct Yuan, award EXP)
9. Reservation status: CONFIRMED
```

**Compensation Flow**:
```
If vote creation fails → Publish VoteSagaFailedEvent
If Yuan confirmation fails → Publish VoteSagaCompensateYuanEvent
→ Gamification Service releases reserved Yuan (status: RELEASED)
```

**Key Components**:

**1. Yuan Reservation System** (Gamification Service):
```java
// Database table: yuan_reservation
CREATE TABLE yuan_reservation (
    id SERIAL PRIMARY KEY,
    reservation_id UUID UNIQUE NOT NULL,
    user_id UUID NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    saga_id VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL, -- RESERVED, CONFIRMED, RELEASED
    expires_at TIMESTAMP NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    confirmed_at TIMESTAMP,
    released_at TIMESTAMP
);

// Service: YuanReservationService
@Service
public class YuanReservationService {
    // Check balance BEFORE reserving (fail fast)
    public UUID reserveYuan(UUID userId, Double amount, String sagaId) {
        // Calculate available balance = total balance - already reserved
        double availableBalance = totalBalance - totalReserved;
        if (availableBalance < amount) {
            throw new ValidationException("Insufficient Yuan balance");
        }
        // Create reservation with status RESERVED
    }
    
    public void confirmReservation(UUID reservationId) {
        // Deduct Yuan from user, award EXP
        // Update reservation status to CONFIRMED
    }
    
    public void releaseReservation(UUID reservationId) {
        // Release reserved Yuan (compensation)
        // Update reservation status to RELEASED
    }
}
```

**2. SAGA Listeners**:

**Gamification Service Listener**:
```java
@KafkaListener(topics = "vote-saga.start", groupId = "gamification-service-vote-saga")
public void handleVoteSagaStart(VoteSagaStartEvent event) {
    // Reserve Yuan (balance already checked)
    UUID reservationId = yuanReservationService.reserveYuan(
        event.getUserId(), 1.0, event.getSagaId()
    );
    // Publish VoteSagaYuanReservedEvent
}

@KafkaListener(topics = "vote-saga.vote-created", groupId = "gamification-service-vote-saga")
public void handleVoteSagaVoteCreated(VoteSagaVoteCreatedEvent event) {
    // Confirm reservation: deduct Yuan + award EXP
    yuanReservationService.confirmReservation(event.getReservationId());
    gamificationService.awardExpForVote(event.getUserId());
}

@KafkaListener(topics = "vote-saga.compensate-yuan", groupId = "gamification-service-vote-saga")
public void handleSagaCompensation(VoteSagaCompensateYuanEvent event) {
    // Release reserved Yuan (rollback)
    yuanReservationService.releaseReservation(event.getReservationId());
}
```

**Engagement Service Listener**:
```java
@KafkaListener(topics = "vote-saga.yuan-reserved", groupId = "engagement-service-vote-saga")
public void handleVoteSagaYuanReserved(VoteSagaYuanReservedEvent event) {
    // Create vote in database
    Vote vote = new Vote();
    vote.setUserId(event.getUserId());
    vote.setNovelId(event.getNovelId());
    voteRepository.save(vote);
    // Publish VoteSagaVoteCreatedEvent
}
```

**3. Balance Check at Reserve Time** (Fail Fast Pattern):
```java
// VoteService.createVoteWithSaga()
private VoteResponseDTO createVoteWithSaga(Integer novelId, UUID userId) {
    // Check balance BEFORE publishing SAGA event
    ApiResponse<VoteCheckResponseDTO> voteCheckResponse = 
        gamificationServiceClient.checkVoteEligibility();
    if (!voteCheckResponse.getData().isCanVote()) {
        throw new ValidationException("Insufficient Yuan balance");
    }
    
    // Only publish SAGA event if balance is sufficient
    kafkaEventProducerService.publishVoteSagaStartEvent(sagaId, userId, novelId);
}
```

**4. Expired Reservation Cleanup**:
```java
@Scheduled(cron = "0 */5 * * * ?") // Every 5 minutes
public void cleanupExpiredReservations() {
    // Release expired RESERVED reservations
    // Prevents Yuan from being held indefinitely
}
```

**5. Hybrid Idempotency**:
- All SAGA events use hybrid idempotency (Redis + Database)
- Prevents duplicate processing of SAGA events
- Ensures exactly-once semantics

**6. Feature Flag**:
- Configurable via `saga.vote-creation.enabled` property
- Allows gradual rollout and easy rollback

**Kafka Topics**:
- `vote-saga.start` - Initiates SAGA
- `vote-saga.yuan-reserved` - Yuan reserved successfully
- `vote-saga.vote-created` - Vote created successfully
- `vote-saga.failed` - SAGA failed
- `vote-saga.compensate-yuan` - Compensation event (release Yuan)

**Benefits**:
- ✅ **Atomicity**: Vote creation and Yuan deduction are atomic (both succeed or both fail)
- ✅ **Data Consistency**: No votes without Yuan deduction, no Yuan deduction without votes
- ✅ **Automatic Compensation**: Failed steps trigger automatic rollback
- ✅ **Fail Fast**: Balance check before SAGA starts (returns 400 if insufficient balance)
- ✅ **Expired Reservation Cleanup**: Scheduled job prevents indefinite Yuan holding
- ✅ **Idempotent**: All events are idempotent (prevents duplicate processing)
- ✅ **Tested**: Verified with sufficient balance and insufficient balance scenarios

**Status**: ✅ Completed - Vote Creation Flow with SAGA pattern is fully implemented and tested.

---

### 10. Gateway-Level JWT Authentication

**Problem**: Currently, each microservice validates JWT tokens independently, causing:
- Redundant validation across services
- Higher latency (validation at each service)
- Inconsistent security policies
- Higher CPU usage
- Security risk: Headers can be forged by attackers

**Solution**: Centralize JWT validation at API Gateway level with HMAC signature protection.

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
        String username = jwtUtil.extractUsername(token);
        String role = jwtUtil.extractRole(token);
        Integer status = jwtUtil.extractStatus(token);  // Extract user status
        
        // Generate HMAC signature to prevent header forgery
        // Signature includes: userId|email|role|status|timestamp
        long timestamp = System.currentTimeMillis();
        String signature = HmacUtil.generateSignature(userId, email, role, status, timestamp, hmacSecret);
        
        // Add user info and HMAC signature to request headers for downstream services
        ServerHttpRequest modifiedRequest = request.mutate()
            .header("X-User-Id", userId)
            .header("X-User-Email", email)
            .header("X-User-Username", username != null ? username : "")
            .header("X-User-Role", role != null ? role : "USER")
            .header("X-User-Status", status != null ? String.valueOf(status) : "0")  // Forward user status
            .header("X-Gateway-Validated", "true")  // Mark as gateway-validated
            .header("X-Gateway-Timestamp", String.valueOf(timestamp))  // Timestamp for signature verification
            .header("X-Gateway-Signature", signature)  // HMAC signature to prevent forgery
            .header("Authorization", authHeader)  // Keep original token for backward compatibility
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

**Microservice Simplification with HMAC Verification**:
```java
// ✅ Phase 3: Simplified Microservice Authentication with HMAC Signature Verification

// Microservices verify HMAC signature before trusting gateway-validated requests
@Component
public class GatewayAuthenticationFilter extends OncePerRequestFilter {
    
    @Value("${gateway.hmac.secret}")
    private String hmacSecret;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request, 
                                   HttpServletResponse response, 
                                   FilterChain filterChain) throws ServletException, IOException {
        
        // Check if request is gateway-validated
        String gatewayValidated = request.getHeader("X-Gateway-Validated");
        if ("true".equals(gatewayValidated)) {
            // Extract user info and signature from headers
            String userId = request.getHeader("X-User-Id");
            String email = request.getHeader("X-User-Email");
            String username = request.getHeader("X-User-Username");
            String role = request.getHeader("X-User-Role");
            String statusStr = request.getHeader("X-User-Status");  // Extract user status
            String timestampStr = request.getHeader("X-Gateway-Timestamp");
            String signature = request.getHeader("X-Gateway-Signature");
            
            // Security: Verify HMAC signature to prevent header forgery
            if (userId == null || email == null || timestampStr == null || signature == null) {
                response.setStatus(HttpStatus.FORBIDDEN.value());
                response.getWriter().write("{\"error\":\"Forbidden\",\"message\":\"Invalid gateway headers\"}");
                return;
            }
            
            try {
                long timestamp = Long.parseLong(timestampStr);
                
                if (!HmacUtil.verifySignature(userId, email, role, statusStr, timestamp, signature, hmacSecret)) {
                    // Invalid signature - reject request
                    response.setStatus(HttpStatus.FORBIDDEN.value());
                    response.getWriter().write("{\"error\":\"Forbidden\",\"message\":\"Invalid gateway signature\"}");
                    return;
                }
            } catch (NumberFormatException e) {
                response.setStatus(HttpStatus.FORBIDDEN.value());
                response.getWriter().write("{\"error\":\"Forbidden\",\"message\":\"Invalid timestamp format\"}");
                return;
            }
            
            // Check user status - ensure user is enabled (not suspended/banned)
            Integer status = 0; // Default to NORMAL/ACTIVE
            if (statusStr != null && !statusStr.isBlank()) {
                try {
                    status = Integer.parseInt(statusStr);
                } catch (NumberFormatException e) {
                    status = 0; // Default to active
                }
            }
            
            CustomUserDetails userDetails = new CustomUserDetails(
                userId, email, username, role != null ? role : "USER", status
            );
            
            if (!userDetails.isEnabled()) {
                // User is disabled, reject with 403 Forbidden
                response.setStatus(HttpStatus.FORBIDDEN.value());
                response.setContentType("application/json");
                response.getWriter().write("{\"error\":\"Forbidden\",\"message\":\"User account is disabled or suspended\",\"status\":403}");
                return;
            }
            
            // Signature verified and user is enabled - trust gateway headers
            // Set authentication context
            UsernamePasswordAuthenticationToken authentication = 
                new UsernamePasswordAuthenticationToken(
                    userDetails, 
                    null, 
                    userDetails.getAuthorities()
                );
            authentication.setDetails(
                new WebAuthenticationDetailsSource().buildDetails(request)
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
- ✅ **HMAC Signature Protection**: Prevents header forgery attacks with cryptographic signatures
- ✅ **Replay Attack Prevention**: Timestamp validation prevents reuse of old signatures
- ✅ **Constant-Time Comparison**: Prevents timing attacks during signature verification
- ✅ **User Status Forwarding**: Gateway forwards `X-User-Status` header from JWT token
- ✅ **User Status Check**: Services verify user is enabled (`isEnabled()`) before authenticating
- ✅ **Disabled User Rejection**: Disabled/suspended users are rejected with **403 Forbidden** response

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

**Solution Implemented**: ✅ **Option B - Redis Block List (Only Inactive Users)**

**Implementation**:
- **API Gateway**: ✅ Maintains Redis Set blocklist of inactive users (SUSPENDED or BANNED)
- **Bootstrap Service**: ✅ Syncs existing blocked users from User Service on startup (with exponential backoff retry)
- **Kafka Event Listener**: ✅ Updates Redis blocklist in real-time when user status changes
- **JWT Filter**: ✅ Checks Redis blocklist before forwarding requests → Rejects blocked users with 403 Forbidden
- **User Service**: ✅ Publishes `UserStatusChangedEvent` to Kafka when status changes
- **Internal Endpoint**: ✅ `/api/v1/internal/blocked-users` for Gateway bootstrap

**Architecture**:
```
User Service → updateUserStatus() → Kafka Event (user-status-events)
   ↓
Gateway → UserStatusEventListener → Update Redis blocklist (real-time)
   ↓
Request → JWT Filter → Check Redis blocklist → Reject if blocked (403)
```

**Benefits**:
- ✅ Real-time updates via Kafka events (<1s latency)
- ✅ Memory efficient (only inactive users, ~1-5MB for 100K blocked users)
- ✅ Fast lookup O(1) Redis Set (<1ms)
- ✅ Scalable (blocklist size doesn't increase with total users)
- ✅ Graceful degradation (Gateway works even if blocklist not synced)
- ✅ Bootstrap retry handles startup order issues (30s → 60s → 120s → 240s → 480s)

**Other Options Considered**:

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

#### Option B: Redis Block List (Only Inactive Users) ✅ **IMPLEMENTED**
- **Approach**: Only store list of blocked/inactive users in Redis Set
- **Key**: `user:blocklist`, Value: Set of `userId`
- **Update**: User Service publishes Kafka event → Gateway updates Redis blocklist
- **Gateway**: Checks block list when validating JWT
- **Logic**: Not in block list = active (assume active)
- **Bootstrap**: Gateway syncs existing blocked users on startup (with retry)
- **Pros**: 
  - ✅ Memory efficient (only inactive users, ~1-5MB for 100K blocked users)
  - ✅ Fast lookup O(1) Redis Set (<1ms)
  - ✅ Scalable (block list size does not increase with total users)
  - ✅ Real-time updates via Kafka events
  - ✅ Graceful degradation (Gateway works even if sync fails)
- **Cons**: 
  - ⚠️ Eventual consistency window (if event not yet synced, ~1-2s delay - acceptable)
  - ⚠️ Bootstrap retry needed if Gateway starts before User Service (handled with exponential backoff)

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

**Decision**: ✅ **Option B - Redis Block List** (Implemented)

**Rationale**:
- Memory efficient (only inactive users)
- Fast performance (<1ms lookup)
- Real-time updates via Kafka
- Simple implementation
- Event-driven architecture (loose coupling)
- Bootstrap retry handles startup order issues gracefully

**Implementation Details**:

**User Service**:
- Internal endpoint: `GET /api/v1/internal/blocked-users` (no auth required, internal network only)
- Method: `AdminService.getBlockedUserIds()` - returns `List<UUID>` of SUSPENDED/BANNED users
- Event: `UserStatusChangedEvent` published to Kafka topic `user-status-events` when status changes
- Event published AFTER transaction commit (using `TransactionAwareKafkaPublisher`)

**Gateway**:
- **Redis Configuration**: Dedicated Redis instance for Gateway (port 6384 in Docker)
- **UserBlocklistService**: Manages Redis Set operations (`user:blocklist` key)
  - `isBlocked(UUID userId)`: Check if user is in blocklist
  - `addToBlocklist(UUID userId)`: Add user to blocklist
  - `removeFromBlocklist(UUID userId)`: Remove user from blocklist
  - `syncBlocklist(Set<UUID>)`: Sync entire blocklist (bootstrap)
- **UserBlocklistBootstrapService**: 
  - Syncs blocked users on startup (background thread, doesn't block Gateway)
  - Exponential backoff retry: 30s → 60s → 120s → 240s → 480s (max 5 attempts)
  - Uses Feign Client (`UserServiceClient`) to call User Service internal endpoint
  - Graceful degradation: Gateway works even if sync fails
- **UserStatusEventListener**: 
  - Listens to `user-status-events` Kafka topic
  - Updates Redis blocklist in real-time (add/remove based on status)
  - Deserializes `UserStatusChangedEvent` from JSON
- **JwtAuthenticationGatewayFilter**: 
  - Checks Redis blocklist after JWT validation
  - Rejects blocked users with 403 Forbidden
  - Graceful fallback if blocklist check fails (continues with request)

**Flow**:
1. **Bootstrap**: Gateway startup → Background thread → Retry with backoff → Call User Service → Sync to Redis
2. **Real-time**: User status changes → Kafka event → Gateway listener → Update Redis
3. **Request**: JWT validated → Check Redis blocklist → Reject if blocked (403) → Forward if not blocked

**Configuration**:
- Redis Key: `user:blocklist` (Redis Set)
- Kafka Topic: `user-status-events`
- Bootstrap retry delays: 30s, 60s, 120s, 240s, 480s
- Max retry attempts: 5

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
- [yushan-microservices-config-server](https://github.com/phutruonnttn/yushan-microservices-config-server) - Config Server (API layer)
- [yushan-microservices-config-data](https://github.com/phutruonnttn/yushan-microservices-config-data) - Config Repository (Git storage for all configs)
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

---

## ✅ Implementation Checklist

### Domain-Driven Design
- ✅ Convert Anemic Domain Models to Rich Domain Models
  - ✅ user-service: User entity with business logic methods (changeStatus, upgradeToAuthor, promoteToAdmin, updateLastLogin, updateLastActive, etc.)
  - ✅ content-service: Novel, Chapter, Category entities with business logic methods (changeStatus, publish, archive, updateContent, etc.)
  - ✅ engagement-service: Comment, Review, Report, Vote entities with business logic methods (updateContent, incrementLikeCount, resolve, dismiss, etc.)
  - ✅ gamification-service: (Skipped - mainly transaction records, minimal business logic needed)
  - ✅ analytics-service: (Skipped - mainly tracking records, minimal business logic needed)
- ✅ Implement Repository Pattern for all aggregates
  - ✅ user-service: UserRepository interface and MyBatisUserRepository implementation
    - All controllers and security components migrated from UserMapper to UserRepository
    - All tests updated and passing (341 tests: 309 unit + 32 integration)
  - ✅ content-service: NovelRepository, ChapterRepository, CategoryRepository with MyBatis implementations + Elasticsearch repositories
    - All services (NovelService, ChapterService, CategoryService) use Repository interfaces
  - ✅ engagement-service: CommentRepository, ReviewRepository, VoteRepository, ReportRepository with MyBatis implementations
    - All services (CommentService, ReviewService, VoteService, ReportService) use Repository interfaces
  - ✅ gamification-service: UserProgressRepository with MyBatis implementation
    - GamificationService uses UserProgressRepository
  - ✅ analytics-service: AnalyticsRepository, HistoryRepository with MyBatis implementations
    - All services (AnalyticsService, HistoryService) use Repository interfaces
- ✅ Define clear aggregate boundaries
  - ✅ content-service: Novel and Chapter are separate aggregates with clear boundaries
  - ✅ user-service: Acceptable as-is (Library and NovelLibrary are child entities of User aggregate)
  - ✅ gamification-service: Acceptable as-is (UserProgress is well-defined aggregate root)
  - ✅ engagement-service: Acceptable as-is (Comment, Review, Vote aggregates are well-separated)
- ✅ Implement Domain Events (internal) (content-service only)
  - ✅ content-service: `ChapterStatisticsChangedEvent` for Novel statistics updates
  - ✅ content-service: `ChapterDomainEventPublisher` and `ChapterDomainEventListener` for event handling
  - ✅ content-service: All chapter operations (create, update, delete, publish) publish Domain Events
  - ✅ user-service: Not needed (no cross-aggregate issues)
  - ✅ gamification-service: Not needed (no cross-aggregate issues)
  - ✅ engagement-service: Not needed (no cross-aggregate issues)
- ✅ Separate Domain Events from Integration Events (content-service)
  - ✅ content-service: Domain Events (internal, same transaction) vs Integration Events (Kafka, cross-service)
  - ✅ content-service: `ChapterStatisticsChangedEvent` is Domain Event (Spring ApplicationEventPublisher)
  - ✅ content-service: Kafka events remain as Integration Events for cross-service communication
  - ✅ Other services: No changes needed (no cross-aggregate issues requiring Domain Events)
- ✅ Kafka Events Transaction Boundary Fix
  - ✅ content-service: All Kafka events publish AFTER transaction commit
  - ✅ engagement-service: All Kafka events publish AFTER transaction commit
  - ✅ user-service: All Kafka events publish AFTER transaction commit
  - ✅ gamification-service: LevelUpEvent publishes AFTER transaction commit
  - ✅ analytics-service: No changes needed (consumer only)
  - ✅ Created `TransactionAwareKafkaPublisher` helper service for all services
  - ✅ Used `TransactionSynchronizationManager` to ensure events publish after commit
  - ✅ Ensures consistency: events only published when transaction commits successfully

### Event-Driven Architecture
- ✅ ~~Create cache tables for cross-service data~~ (Not needed - write operations optimized via Kafka)
- ✅ ~~Implement event listeners for cache updates~~ (Not needed)
- ✅ ~~Add bootstrap mechanism for existing data~~ (Not needed)
- ✅ Implement hybrid idempotent event processing
  - Hybrid approach: Redis (fast checks <1ms) + Database table (persistent backup)
  - Created `processed_events` table in gamification-service, content-service, user-service
  - Implemented `IdempotencyService` for dual-layer idempotency checks
  - All Kafka event consumers now use hybrid idempotency:
    - gamification-service: UserEventListener, EngagementEventListener, InternalEventListener
    - content-service: EngagementEventListener (novel-rating-events, novel-vote-count-events)
    - user-service: UserActivityListener
  - Ensures idempotency even when Redis is restarted (data persisted in database)
- ✅ Optimize inter-service communication
  - Migrated `updateNovelRatingAndCount` to Kafka (`novel-rating-events`)
  - Migrated `incrementVoteCount` to Kafka (`novel-vote-count-events`)
  - Response time improved from 600-700ms to <100ms

### Resilience & Observability
- ✅ Add circuit breakers to all service calls
  - ✅ engagement-service: Circuit Breaker for 3 Feign clients (ContentServiceClient, UserServiceClient, GamificationServiceClient)
  - ✅ analytics-service: Circuit Breaker for 4 Feign clients (ContentServiceClient, UserServiceClient, GamificationServiceClient, EngagementServiceClient)
  - ✅ api-gateway: Circuit Breaker for UserServiceClient (blocked users sync)
  - ✅ user-service: Already had Circuit Breaker for ContentServiceClient
  - ✅ All Feign client methods have fallback methods/classes for graceful degradation
  - ✅ Circuit Breakers enabled via `spring.cloud.openfeign.circuitbreaker.enabled=true` (no `@CircuitBreaker` annotations on Feign methods)
  - ✅ Resilience4j configuration added to all service config files
  - ✅ Circuit Breaker state monitoring via Actuator endpoints
  - ✅ Conflict resolution between bootstrap retry and Circuit Breaker fallback in API Gateway
- ✅ Implement rate limiting on critical endpoints
  - ✅ engagement-service: Rate limiter on comment creation (10/60s) and review creation (5/60s) via `RateLimiterInterceptor` (HandlerInterceptor pattern)
  - ✅ api-gateway: Global rate limiter (100 requests/60s) via `RateLimiterGatewayFilter` (GlobalFilter)
  - ✅ Resilience4j Rate Limiter configuration added
  - ✅ All Rate Limiters tested and verified working correctly (HTTP 429 when limits exceeded)
- ⬜ Set up distributed tracing (Jaeger/Zipkin)
- ⬜ Configure Prometheus metrics
- ⬜ Set up Grafana dashboards

### Security Improvements
- ✅ Implement Gateway-Level JWT Authentication
  - ✅ Add JWT validation filter to API Gateway (`JwtAuthenticationGatewayFilter`)
  - ✅ Implement HMAC signature generation in Gateway (`HmacUtil`)
  - ✅ Add HMAC signature verification in all services (`GatewayAuthenticationFilter`)
  - ✅ Configure shared secret (`GATEWAY_HMAC_SECRET`) across Gateway and all services
  - ✅ Implement timestamp validation (5-minute tolerance) to prevent replay attacks
  - ✅ Implement constant-time comparison to prevent timing attacks
  - ✅ Simplify microservice authentication (trust gateway-validated requests with signature verification)
  - ✅ Configure public endpoints whitelist (comprehensive list in `JwtAuthenticationGatewayFilter`)
  - ✅ Add fallback authentication for service-to-service calls (JWT validation for backward compatibility)
  - ✅ Update Feign clients to forward HMAC signature headers in inter-service calls
- ⬜ Implement gateway high availability
- ✅ Fix inactive user token validation issue (Option B)
  - ✅ Option B: Redis Block List (only inactive users) - implemented
    - ✅ Created Redis blocklist in Gateway (Redis Set: `user:blocklist`)
    - ✅ Created internal endpoint in User Service (`/api/v1/internal/blocked-users`)
    - ✅ Created `UserStatusChangedEvent` DTO and publish from AdminService
    - ✅ Created `UserBlocklistBootstrapService` with exponential backoff retry (30s → 480s)
    - ✅ Created `UserStatusEventListener` to update blocklist from Kafka events
    - ✅ Updated `JwtAuthenticationGatewayFilter` to check Redis blocklist before forwarding
    - ✅ Configured Redis and Kafka in Gateway
    - ✅ Graceful degradation: Gateway works even if blocklist not synced

### Kubernetes & Cloud
- ✅ Create Kubernetes manifests for all services - **Completed (all 6 services)**
- ✅ Replace Eureka with Kubernetes Service Discovery - **Implemented**
- ⬜ Configure auto-scaling (HPA) - **Next: Task 2**
- ⬜ Set up service mesh (Istio/Linkerd) - optional
- ✅ **Migrate to AWS EKS - Task 1 Complete**: [AWS Deployment Repository](https://github.com/phutruonnttn/yushan-AWS-deployment)
  - ✅ VPC Infrastructure (Subtask 3)
  - ✅ Security Groups (Subtask 4)
  - ✅ EKS Cluster (Subtask 5) - 2 nodes (t3.small), 4GB total memory
  - ✅ RDS PostgreSQL (Subtask 6) - 5x instances (Database-per-Service pattern)
  - ✅ ElastiCache Redis (Subtask 7) - 5x clusters (Database-per-Service pattern)
  - ✅ EC2 Kafka Cluster (Subtask 8) - 3 brokers (t3.small, Multi-AZ), Zookeeper installed
  - ✅ S3 Buckets (Subtask 9)
  - ✅ Application Load Balancer (Subtask 10)
  - ✅ ECR Repositories (Subtask 11)
  - ✅ Service Configs Update (Subtask 12)
  - ✅ K8s Manifests (Subtask 13) - All 6 services deployed
  - ✅ Testing & Validation (Subtask 14) - All APIs functional, Kafka enabled
- ✅ Configure AWS RDS - **Complete**: 5x PostgreSQL instances (Database-per-Service)
- ✅ Set up AWS ElastiCache - **Complete**: 5x Redis clusters (Database-per-Service)
- ✅ Configure EC2 Kafka (3 brokers Multi-AZ) - **Complete**: Installed, configured, and running
- ✅ Set up AWS S3 for file storage - **Complete**: Bucket created with CORS and lifecycle policies

### SAGA Pattern
- ✅ Identify distributed transactions
  - ✅ Vote Creation Flow identified as requiring distributed transaction management
- ✅ Design SAGA flows (choreography)
  - ✅ Choreography pattern chosen for Vote Creation Flow
  - ✅ Event-driven flow with Kafka topics
  - ✅ Multi-step transaction: Reserve Yuan → Create Vote → Confirm Yuan
- ✅ Implement SAGA orchestrator/participants
  - ✅ Yuan Reservation System implemented (gamification-service)
  - ✅ VoteSagaListener in gamification-service (reserve, confirm, compensate)
  - ✅ VoteSagaListener in engagement-service (create vote)
  - ✅ Balance check at reserve time (fail fast pattern)
  - ✅ Feature flag for gradual rollout (`saga.vote-creation.enabled`)
- ✅ Add compensation logic
  - ✅ Automatic Yuan release on SAGA failures
  - ✅ Reservation status management (RESERVED → CONFIRMED/RELEASED)
  - ✅ Scheduled cleanup job for expired reservations
- ✅ Test failure and recovery scenarios
  - ✅ Tested with sufficient balance (vote created successfully)
  - ✅ Tested with insufficient balance (returns 400, no vote created)
  - ✅ Verified compensation logic (Yuan released on failures)
  - ✅ API contract fixed (returns 400 when balance = 0)

---

## 📚 References

- [Domain-Driven Design](https://martinfowler.com/bliki/DomainDrivenDesign.html)
- [Repository Pattern](https://martinfowler.com/eaaCatalog/repository.html)
- [SAGA Pattern](https://microservices.io/patterns/data/saga.html)
- [Kubernetes Service Discovery](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Distributed Tracing](https://opentracing.io/)
- [Circuit Breaker Pattern](https://martinfowler.com/bliki/CircuitBreaker.html)

---

**Last Updated**: January 2025 - **AWS Infrastructure Deployment (Task 1) Complete**: All infrastructure deployed (EKS, RDS, ElastiCache, Kafka, S3, ALB), all 6 microservices deployed and running on AWS EKS, Kafka installed and configured (3 brokers with Zookeeper), all APIs functional with Kafka enabled, all Swagger UIs accessible (6/6). Previous completions: SAGA Pattern for distributed transactions (Vote Creation Flow with Choreography pattern, Yuan Reservation System), Repository Pattern (all services), Aggregate Boundaries & Domain Events (content-service), Kafka Events Transaction Boundary Fix (all services), Gateway-Level JWT Authentication with HMAC Signature (all services), Inactive User Token Validation (Redis Block List), Circuit Breakers & Rate Limiters (comprehensive coverage). **Phase 3 Progress: 75% Complete**. Next: Kubernetes migration (Task 2), Configuration management (Task 3), Distributed tracing (Task 4).

