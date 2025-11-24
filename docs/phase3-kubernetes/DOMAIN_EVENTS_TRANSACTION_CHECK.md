# Domain Events Transaction Boundary Verification

## ✅ Checklist: Domain Events & Transaction Boundaries

- [x] **content-service**: Domain Events trong cùng transaction ✅
- [ ] **content-service**: Kafka Events (Integration Events) - Cần verify publish sau transaction commit
- [ ] **engagement-service**: Kiểm tra Kafka events publish trong/sau transaction
- [ ] **user-service**: Kiểm tra Kafka events publish trong/sau transaction
- [ ] **gamification-service**: Kiểm tra Kafka events publish trong/sau transaction
- [ ] **analytics-service**: Kiểm tra Kafka events publish trong/sau transaction

---

## 📋 Detailed Analysis

### 1. content-service ✅ **PASS**

#### Domain Events (Internal - Same Transaction)
- **Implementation**: `ChapterDomainEventPublisher` + `ChapterDomainEventListener`
- **Event Type**: `ChapterStatisticsChangedEvent` (Spring ApplicationEventPublisher)
- **Transaction Boundary**: ✅ **CORRECT**
  - `ChapterService.createChapter()` có `@Transactional`
  - `chapterDomainEventPublisher.publishChapterStatisticsChanged()` được gọi TRONG transaction
  - `ChapterDomainEventListener.handleChapterStatisticsChanged()` có `@EventListener` + `@Transactional`
  - Event listener chạy TRONG CÙNG transaction với aggregate modification
  - Nếu transaction rollback, event handling cũng rollback ✅

**Code Verification**:
```java
@Transactional  // ✅ Transaction boundary
public ChapterDetailResponseDTO createChapter(...) {
    chapterRepository.save(chapter);  // Aggregate modification
    chapterDomainEventPublisher.publishChapterStatisticsChanged(...);  // ✅ Domain Event trong transaction
}

@EventListener
@Transactional  // ✅ Event handler trong cùng transaction
public void handleChapterStatisticsChanged(ChapterStatisticsChangedEvent event) {
    // Update Novel statistics - trong cùng transaction
}
```

#### Kafka Events (Integration Events - Cross-Service)
- **Implementation**: `KafkaEventProducerService`
- **Transaction Boundary**: ⚠️ **POTENTIAL ISSUE**
  - Kafka events được publish TRONG transaction (line 110, 537, etc.)
  - Nếu transaction rollback, Kafka event đã được gửi nhưng data không được commit
  - **Recommendation**: Nên dùng `@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)` hoặc publish SAU transaction commit

**Current Code**:
```java
@Transactional
public ChapterDetailResponseDTO createChapter(...) {
    chapterRepository.save(chapter);
    chapterDomainEventPublisher.publishChapterStatisticsChanged(...);  // ✅ Domain Event
    
    // ⚠️ Kafka event TRONG transaction
    kafkaEventProducerService.publishChapterCreatedEvent(...);  // Should be AFTER_COMMIT
}
```

**Recommendation**: 
- Domain Events: ✅ Đúng (trong transaction)
- Integration Events (Kafka): ⚠️ Nên publish sau transaction commit

---

### 2. engagement-service ⚠️ **NEEDS FIX**

#### Kafka Events (Integration Events)
- **Implementation**: `KafkaEventProducerService`
- **Transaction Boundary**: ❌ **ISSUE FOUND**
  - `CommentService.createComment()` (line 38-68): Kafka event publish TRONG transaction (line 62)
  - `ReviewService.createReview()` (line 40-80): Kafka event publish TRONG transaction (line 70)
  - `VoteService`: Kafka events publish TRONG transaction (line 88, 91)
  - `ReviewService.updateNovelRatingAndCount()`: Kafka event publish TRONG transaction (line 340)

**Code Verification**:
```java
@Transactional  // ✅ Transaction boundary
public CommentResponseDTO createComment(...) {
    commentRepository.save(comment);  // Aggregate modification
    // ❌ Kafka event TRONG transaction
    kafkaEventProducerService.publishCommentCreatedEvent(...);  // Should be AFTER_COMMIT
}
```

**Action Required**: Refactor để publish Kafka events SAU transaction commit

---

### 3. user-service ⚠️ **NEEDS FIX**

#### Kafka Events (Integration Events)
- **Implementation**: `UserEventProducer`, `UserActivityEventProducer`
- **Transaction Boundary**: ❌ **ISSUE FOUND**
  - `AuthService.registerAndCreateResponse()` (line 89-103): Kafka event publish TRONG transaction (line 103)
  - `AuthService.login()` (line 163): Kafka event publish TRONG transaction

**Code Verification**:
```java
@Transactional(rollbackFor = Exception.class)  // ✅ Transaction boundary
public UserAuthResponseDTO registerAndCreateResponse(...) {
    User user = register(registrationDTO);  // Aggregate modification
    
    UserRegisteredEvent event = new UserRegisteredEvent(...);
    // ❌ Kafka event TRONG transaction
    userEventProducer.sendUserRegisteredEvent(event);  // Should be AFTER_COMMIT
}
```

**Action Required**: Refactor để publish Kafka events SAU transaction commit

---

### 4. gamification-service ⚠️ **NEEDS VERIFICATION**

#### Kafka Events (Integration Events)
- **Implementation**: `KafkaEventProducerService`
- **Transaction Boundary**: Cần kiểm tra xem Kafka events có được publish trong hay sau transaction

**Files to Check**:
- `GamificationService` methods với `@Transactional`

**Action Required**: Verify Kafka events được publish sau transaction commit

---

### 5. analytics-service ⚠️ **NEEDS VERIFICATION**

#### Kafka Events (Integration Events - Consumer Only)
- **Implementation**: Kafka listeners
- **Transaction Boundary**: Cần kiểm tra xem event processing có `@Transactional` không

**Action Required**: Verify event listeners có transaction boundaries đúng

---

## 🔍 Summary

### ✅ Domain Events (Internal - Same Transaction)
- **content-service**: ✅ **PASS** - Domain Events được thực hiện trong cùng transaction

### ⚠️ Integration Events (Kafka - Cross-Service)
- **content-service**: ⚠️ **NEEDS FIX** - Kafka events được publish TRONG transaction (nên publish SAU transaction commit)
- **engagement-service**: ❌ **NEEDS FIX** - Kafka events được publish TRONG transaction (CommentService, ReviewService, VoteService)
- **user-service**: ❌ **NEEDS FIX** - Kafka events được publish TRONG transaction (AuthService.registerAndCreateResponse, login)
- **gamification-service**: ⚠️ **NEEDS VERIFICATION** - Cần kiểm tra (chủ yếu là consumer, ít producer)
- **analytics-service**: ⚠️ **NEEDS VERIFICATION** - Cần kiểm tra (chủ yếu là consumer)

---

## 📝 Recommendations

### For Domain Events (Internal - Same Transaction)
✅ **Current Implementation is CORRECT**:
- `@EventListener` + `@Transactional` đảm bảo event handling trong cùng transaction
- Nếu transaction rollback, event handling cũng rollback
- Strong consistency được đảm bảo

### For Integration Events (Kafka - Cross-Service)
⚠️ **Should publish AFTER transaction commit**:

**Option 1: Use @TransactionalEventListener**
```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void handleDomainEvent(DomainEvent event) {
    // Publish Kafka event AFTER transaction commit
    kafkaProducer.send(event);
}
```

**Option 2: Use TransactionSynchronizationManager**
```java
@Transactional
public void createChapter(...) {
    chapterRepository.save(chapter);
    
    // Register callback to run AFTER transaction commit
    TransactionSynchronizationManager.registerSynchronization(
        new TransactionSynchronization() {
            @Override
            public void afterCommit() {
                kafkaProducer.send(event);
            }
        }
    );
}
```

**Option 3: Publish in separate transaction (after commit)**
```java
@Transactional
public void createChapter(...) {
    chapterRepository.save(chapter);
    // Don't publish Kafka event here
}

// In controller or service layer, after transaction commits
public void createChapterWithEvent(...) {
    Chapter chapter = chapterService.createChapter(...);  // Transaction commits here
    kafkaProducer.send(new ChapterCreatedEvent(chapter));  // Publish after commit
}
```

---

## ✅ Action Items

1. ✅ **content-service Domain Events**: Đã đúng - không cần sửa
2. ❌ **content-service Kafka Events**: Cần refactor để publish sau transaction commit
3. ❌ **engagement-service Kafka Events**: Cần refactor để publish sau transaction commit
4. ❌ **user-service Kafka Events**: Cần refactor để publish sau transaction commit
5. ⚠️ **gamification-service & analytics-service**: Cần verify (chủ yếu là consumer)

---

## 📊 Summary Table

| Service | Domain Events (Internal) | Integration Events (Kafka) | Status |
|---------|------------------------|---------------------------|--------|
| **content-service** | ✅ PASS (trong transaction) | ❌ FAIL (trong transaction) | ⚠️ Needs Fix |
| **engagement-service** | N/A (không có Domain Events) | ❌ FAIL (trong transaction) | ❌ Needs Fix |
| **user-service** | N/A (không có Domain Events) | ❌ FAIL (trong transaction) | ❌ Needs Fix |
| **gamification-service** | N/A (không có Domain Events) | ⚠️ Unknown | ⚠️ Needs Verification |
| **analytics-service** | N/A (không có Domain Events) | ⚠️ Unknown | ⚠️ Needs Verification |

**Conclusion**: 
- ✅ Domain Events (nếu có) đều đúng - trong cùng transaction
- ❌ Integration Events (Kafka) đều publish TRONG transaction - cần fix để publish SAU transaction commit

