# Phase 3: Kubernetes & AWS Deployment

> 🚀 **Advanced microservices architecture with Kubernetes orchestration, distributed tracing, Saga pattern, and cloud-native improvements**

## 📋 Overview

**Status**: 🔄 Planning Phase | **Target**: AWS EKS Deployment

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
- ✅ Rich Domain Models (fixing Anemic Domain Model)
- ✅ Repository Pattern implementation
- ✅ Aggregate boundaries and Domain Events
- ✅ Event-driven cache tables for eventual consistency
- ✅ Idempotent event consumption
- ✅ SAGA pattern for distributed transactions

### Resilience & Observability
- ✅ Circuit breakers (comprehensive coverage)
- ✅ Rate limiting
- ✅ Distributed tracing
- ✅ Enhanced monitoring and observability

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

### 4. Event-Driven Cache Tables (Eventual Consistency)

**Problem**: Too many synchronous API calls between services (e.g., Engagement Service checking if novel exists via Content Service API).

**Solution**: Event-driven cache tables with eventual consistency.

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

**Benefits**:
- ✅ Reduces synchronous API calls
- ✅ Faster response times (local cache lookup)
- ✅ Better resilience (works even if Content Service is down)
- ✅ Eventual consistency (acceptable for most use cases)

**Trade-offs**:
- ⚠️ Eventual consistency (cache might be slightly stale)
- ⚠️ Additional storage per service
- ⚠️ Need bootstrap mechanism for existing data

---

### 5. Idempotent Event Consumption

**Problem**: Events might be processed multiple times, causing duplicate side effects.

**Solution**: Implement idempotency checks in event consumers.

**Implementation**:
```java
// ✅ Phase 3: Idempotent Event Processing

@Entity
@Table(name = "processed_events")
public class ProcessedEvent {
    @Id
    private String eventId; // Unique event identifier
    private String eventType;
    private LocalDateTime processedAt;
    private String serviceName;
}

@Repository
public interface ProcessedEventRepository extends JpaRepository<ProcessedEvent, String> {
    boolean existsByEventId(String eventId);
}

@Component
public class IdempotentGamificationEventListener {
    
    @Autowired
    private ProcessedEventRepository processedEventRepository;
    
    @KafkaListener(topics = "user.events")
    public void handleUserEvent(UserEvent event) {
        // ✅ Idempotency check
        if (processedEventRepository.existsByEventId(event.getEventId())) {
            log.info("Event {} already processed, skipping", event.getEventId());
            return; // Idempotent - same result
        }
        
        try {
            // Process event
            processUserEvent(event);
            
            // Mark as processed
            processedEventRepository.save(new ProcessedEvent(
                event.getEventId(),
                event.getType(),
                LocalDateTime.now(),
                "gamification-service"
            ));
        } catch (Exception e) {
            // If processing fails, event is not marked as processed
            // Will be retried by Kafka
            throw e;
        }
    }
    
    private void processUserEvent(UserEvent event) {
        // Idempotent operation
        switch (event.getType()) {
            case USER_REGISTERED:
                // Check if already processed
                if (!hasReceivedRegistrationReward(event.getUserId())) {
                    grantRegistrationReward(event.getUserId());
                }
                break;
        }
    }
}
```

**Idempotency Strategies**:
1. **Event ID Tracking**: Store processed event IDs
2. **Idempotent Operations**: Design operations to be naturally idempotent
3. **Idempotency Keys**: Use idempotency keys for API calls

**Guarantees**:
- ✅ **At-least-once delivery**: Event processed at least once
- ✅ **Idempotent processing**: Multiple processing = same result
- ✅ **No duplicate side effects**: Even if event is retried

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
- [ ] Convert Anemic Domain Models to Rich Domain Models
- [ ] Implement Repository Pattern for all aggregates
- [ ] Define clear aggregate boundaries
- [ ] Implement Domain Events (internal)
- [ ] Separate Domain Events from Integration Events

### Event-Driven Architecture
- [ ] Create cache tables for cross-service data
- [ ] Implement event listeners for cache updates
- [ ] Add bootstrap mechanism for existing data
- [ ] Implement idempotent event processing
- [ ] Add event deduplication mechanism

### Resilience & Observability
- [ ] Add circuit breakers to all service calls
- [ ] Implement rate limiting on critical endpoints
- [ ] Set up distributed tracing (Jaeger/Zipkin)
- [ ] Configure Prometheus metrics
- [ ] Set up Grafana dashboards

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

**Last Updated**: Planning Phase - 2025

