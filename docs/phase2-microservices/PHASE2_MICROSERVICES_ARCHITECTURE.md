## Yushan Platform – Index & Architecture Notes

This document captures the structure of the entire `yushan-micro` workspace, the available microservices, their APIs and shared infrastructure, together with the two React front-end applications (`yushan-platform-admin` and `yushan-platform-frontend`). Use it as the canonical reference when working on the stack.

---

### 1. High-Level Architecture

- **Domain Microservices (Business Layer)**
  - `yushan-user-service` – authentication, user/profile management, library, admin tools.
  - `yushan-content-service` – novels, chapters, search (Elasticsearch), moderation, file storage (S3/local).
  - `yushan-engagement-service` – comments, reviews, ratings, bookmarks, votes, notifications, social features.
  - `yushan-gamification-service` – XP, levels, Yuan currency, achievements, leaderboards, Kafka listeners.
  - `yushan-analytics-service` – reading history, rankings, platform analytics, Redis ranking caches.

- **DMZ / Platform Services**
  - `yushan-api-gateway` – Spring Cloud Gateway routing all `/api/v1/**` requests via Eureka discovery.
  - `yushan-config-server` – Spring Cloud Config (native profile with `configs/*.yml`), centralizes env.
  - `yushan-platform-service-registry` – Eureka server for service discovery.

- **Frontends**
  - `yushan-platform-admin` – React + Ant Design admin dashboard.
  - `yushan-platform-frontend` – Reader-facing React app (CRA) with Redux, Ant Design.

- **Infrastructure Support**
  - `Digital_Ocean_Deployment_with_Terraform` – Terraform scripts for services & monitoring (Prometheus/ELK).
  - `Yushan_Web_Novel_Design_Documents` – Additional diagrams & lifecycle notes.

Shared communication patterns:
- All services register with Eureka and fetch configs via config-server (`http://yushan-config-server:8888` default).
- Inter-service calls handled via OpenFeign clients with auth header forwarding.
- Kafka topics connect `user` → `gamification` and `engagement` → `gamification` flows.
- Redis is used for caching (user activity throttling, ranking sorted sets).

---

### 2. Microservice Notes

#### 2.1 `yushan-user-service`
- **Key Files & Directories**
  - `UserServiceApplication.java`
  - `config/` – security, Kafka producer, DB, OpenAPI, web config, UUID handler.
    - `SecurityConfig.java` – HTTP security config, public endpoints, role guards.
    - `KafkaProducerConfig.java` – Kafka template bean for event envelopes.
    - `DatabaseConfig.java` – MyBatis configuration, mapper scanning.
    - `OpenApiConfig.java` – Swagger/knife4j description.
  - `controller/` – `AuthController`, `UserController`, `LibraryController`, `AdminController`, `AuthorController`, `HealthController`.
    - `AuthController.java` – email verification, register, login, refresh.
    - `UserController.java` – `/me`, `/users/{id}`, ranking export, batch lookup.
    - `LibraryController.java` – add/remove/patch progress, batch delete.
    - `AdminController.java` – status update, promotion, paged filter.
    - `AuthorController.java` – author upgrade + verification email.
  - `service/` – `AuthService`, `UserService`, `LibraryService`, `AdminService`, `AuthorService`, `MailService`.
    - `AuthService.java` – password hashing, login, event publishing, refresh token validation.
    - `UserService.java` – profile mapping, email verification logic, Redis throttling for last active.
    - `LibraryService.java` – interop with content-service via Feign + fallback, progress checks.
    - `MailService.java` – email OTP send/verify (Redis).
  - `dao/` – `UserMapper`, `LibraryMapper`, `NovelLibraryMapper` plus XML mappers in `resources/mapper/`.
    - XML mappers define SQL for select/update operations with pagination.
  - `event/` – Kafka producers (`UserEventProducer`, `UserActivityEventProducer`) and event DTOs.
    - `UserEventProducer.java` – serializes `EventEnvelope` for Kafka topic `user.events`.
    - `UserActivityEventProducer.java` – writes to topic `active`.
  - `security/` – JWT filter, entry point, custom method expressions, custom user details.
    - `JwtAuthenticationFilter.java` – extracts token via `JwtUtil`, sets security context.
    - `CustomUserDetailsService.java` – loads user + roles for security context.
  - `dto/` – registration/login requests, profile responses, admin filters, library DTOs.
  - `entity/` – `User`, `Library`, `NovelLibrary`.
- **Tech**: Spring Boot 3, MyBatis, PostgreSQL, Redis, Kafka producers.
- **Inter-service Feign**:
  - `ContentServiceClient` for novel/chapter metadata validation.
- **Security**: JWT & refresh tokens, `SecurityConfig` with `JwtAuthenticationFilter`.
- **Key Controllers**
  - `AuthController` – `/api/v1/auth/**` (login, register, refresh, email verification).
  - `UserController` – `/api/v1/users/**` (profile, stats, library batch, ranking list).
  - `LibraryController` – `/api/v1/library/**` (add/remove/check progress).
  - `AdminController` – `/api/v1/admin/**` (promote admin, list/filter users, update status).
  - `AuthorController` – `/api/v1/author/**` (upgrade to author via verification).
- **Events**: `UserEventProducer` sends `UserRegisteredEvent` / `UserLoggedInEvent` to Kafka topic `user.events`.
- **Persistence**: `UserMapper`, `LibraryMapper`, `NovelLibraryMapper` (MyBatis XML).
- **Cache**: `RedisUtil` to rate-limit updates (e.g., last active).
- **Configuration**: `configs/user-service.yml` sets DB, Redis, Kafka, JWT, Resilience4j.

#### 2.2 `yushan-content-service`
- **Key Files & Directories**
  - `ContentServiceApplication.java`
  - `config/` – database, Redis, Elasticsearch (config + startup indexer), Kafka, security, storage (local/S3), static resources.
    - `ElasticsearchConfig.java` – RestHighLevelClient bean & index settings.
    - `ElasticsearchStartupIndexer.java` – bootstraps indexes on application start.
    - `StorageConfig.java` – toggles between `LocalFileStorageService` and `S3FileStorageService`.
  - `controller/` – `NovelController`, `ChapterController`, `CategoryController`, `SearchController`, `HealthController`, `TestController`.
    - `NovelController.java` – 30+ endpoints for CRUD, hide/unhide, approve, review, ranking.
    - `ChapterController.java` – chapter CRUD, publish/draft, reorder.
    - `SearchController.java` – combined search bridging SQL and Elasticsearch.
  - `service/` – `NovelService`, `ChapterService`, `CategoryService`, `SearchService`, `ElasticsearchIndexService`, `KafkaEventProducerService`, `FileStorageService`, `ImageProcessingService`.
    - `NovelService.java` – orchestrates novel state transitions, updates stats, emits `NovelCreatedEvent`.
    - `ChapterService.java` – ensures ordering, publishes events, handles batch create.
    - `ElasticsearchIndexService.java` – index maintenance (novel & chapter docs).
    - `KafkaEventProducerService.java` – `novel.events`, `chapter.events` topics.
  - `repository/elasticsearch/` – `NovelElasticsearchRepository`, `ChapterElasticsearchRepository`.
  - `entity/` – `Novel`, `Chapter`, `Category`, plus Elasticsearch documents.
  - `dto/` – shared/common responses, novel/chapter/category payloads, event payloads, search results.
  - `security/` – JWT filter, security expression root, `NovelGuard`, `UserActivityFilter`.
  - `resources/mapper/` – MyBatis XML definitions.
- **Tech**: Spring Boot, JPA + MyBatis hybrid, Elasticsearch.
- **Feign Clients**: calls `user-service` (Author validation), `engagement-service` (for ratings) if needed.
- **Controllers**: `NovelController`, `ChapterController`, `CategoryController`, `SearchController`, etc.
- **Security**: JWT, `NovelGuard` for author/ admin editing rights; method-level annotations (`@PreAuthorize`).
- **Search**: `ElasticsearchConfig`, `ElasticsearchIndexService`, auto-indexer on startup.
- **Events**: `KafkaEventProducerService` for content events (chapter/novel create/update).
- **File Storage**: `FileStorageService` with `LocalFileStorageService` / `S3FileStorageService`.
- **Config**: `configs/content-service.yml` defines DB, Redis, Elasticsearch, MinIO/S3.

#### 2.3 `yushan-engagement-service`
- **Key Files & Directories**
  - `EngagementServiceApplication.java`
  - `config/` – DB, Redis, Kafka, OpenAPI, security, Feign auth, UUID handler.
  - `controller/` – `CommentController`, `ReviewController`, `VoteController`, `ReportController`, `HealthController`, `TestController`.
    - `CommentController.java` – ~40 endpoints covering CRUD, admin moderation, spoiler toggles.
    - `ReviewController.java` – review lifecycle, rating summary.
    - `VoteController.java` – user vote history & create vote (calls gamification).
  - `service/` – `CommentService`, `ReviewService`, `VoteService`, `ReportService`, `KafkaEventProducerService`.
    - `CommentService.java` – ensures single comment per chapter, merges user names via `UserServiceClient`.
    - `ReviewService.java` – updates content-service rating counts.
    - `VoteService.java` – handles Yuan deduction, content-service vote count update.
  - `client/` – Feign clients (content, user, gamification).
  - `dao/` – `CommentMapper`, `ReviewMapper`, `VoteMapper`, `ReportMapper`.
  - `dto/` – comment/review/vote/report payloads, event DTOs, gamification vote checks.
  - `entity/` – `Comment`, `Review`, `Vote`, `Report`.
  - `security/` – JWT filter & entry point, `UserActivityFilter`, method expression handler.
- **Tech**: Spring Boot, MyBatis, Kafka, Redis (notifications), optional Elasticsearch.
- **Feign Clients**: `ContentServiceClient`, `UserServiceClient`, `GamificationServiceClient`.
- **Controllers**:
  - `CommentController` – CRUD comments + moderation endpoints.
  - `ReviewController`, `VoteController`, `ReportController`.
- **Services**:
  - `CommentService` – ensures one comment per chapter, fetches metadata, publishes `CommentCreatedEvent`.
  - `ReviewService` – rating updates, triggers content-service rating update.
  - `VoteService` – ensures user has Yuan via gamification, increments novel votes.
- **Kafka**: `KafkaEventProducerService` to topics `comment-events`, `review-events`, `vote-events`, `active`.
- **Security**: JWT auth, `UserActivityFilter` hooking to analytics topic.
- **Config**: `configs/engagement-service.yml`.

#### 2.4 `yushan-gamification-service`
- **Key Files & Directories**
  - `GamificationServiceApplication.java`
  - `config/` – DB, Redis, Kafka, security, Feign auth.
  - `controller/` – `GamificationController`, `GamificationStatsController`, `AdminGamificationController`, `TestController`.
    - `GamificationController.java` – reward endpoints (`/comments/{id}/reward`, `/votes/reward`), check/deduct vote.
    - `GamificationStatsController.java` – public stats/achievements (`/stats/me`, `/achievements/me`, `/stats/all`, batch).
    - `AdminGamificationController.java` – admin yuan transaction search + manual credit.
  - `service/` – `GamificationService`, `AchievementService`, `LevelService`, `KafkaEventProducerService`.
    - `GamificationService.java` – central ledger (EXP/Yuan), login reward, vote eligibility, daily streak.
    - `AchievementService.java` – unlocks on thresholds (login, comments, levels).
    - `LevelService.java` – threshold array for XP levels.
  - `listener/` – `UserEventListener`, `EngagementEventListener`, `InternalEventListener`.
  - `dto/` – achievements (`achievement/*`), events, stats, transactions, vote check DTOs.
  - `dao/` – transaction/achievement mappers (`ExpTransactionMapper`, `YuanTransactionMapper`, etc.).
  - `entity/` – `Achievement`, `UserAchievement`, `ExpTransaction`, `YuanTransaction`, `DailyRewardLog`.
  - `security/` – JWT components and custom user details.
- **Tech**: Spring Boot, MyBatis, Kafka listeners, Redis, PostgreSQL.
- **Listeners**:
  - `UserEventListener` (topic `user.events`) – registration & login rewards.
  - `EngagementEventListener` (topics `comment-events`, `review-events`, `vote-events`) – XP/Yuan.
  - `InternalEventListener` (topic `internal_gamification_events`) – level achievements.
- **Services**:
  - `GamificationService` – handles EXP/Yuan transactions, daily login reward, check vote eligibility (`VoteCheckResponseDTO`), level calculation, admin Yuan adjusts.
  - `AchievementService` – unlocks achievements on thresholds.
  - `LevelService` – static thresholds.
  - `KafkaEventProducerService` – `UserActivityEvent` topic `active`.
- **API**:
  - `/api/v1/gamification/**` – stats, achievements, rewards, vote check/deduct, admin endpoints (`/admin`).
  - `/vote` operations support engagement service.
- **Config**: `configs/gamification-service.yml`.

#### 2.5 `yushan-analytics-service`
- **Key Files & Directories**
  - `AnalyticsServiceApplication.java`
  - `config/` – database, Redis, `FeignAuthConfig`, `RestTemplateConfig`, `SecurityConfig`.
  - `controller/` – `AnalyticsController`, `RankingController`, `HistoryController`, `HealthController`, `TestController`.
    - `AnalyticsController.java` – trends/summary/platform stats/DAU/time-series endpoints protected with admin role.
    - `RankingController.java` – `/api/v1/ranking/{novel|user|author}` plus `/novel/{id}/rank`.
    - `HistoryController.java` – `POST /history/novels/{id}/chapters/{id}` reading events, list/clear history.
  - `service/` – `AnalyticsService`, `RankingService`, `RankingUpdateService`, `HistoryService`, `LibraryService`.
    - `AnalyticsService.java` – fetches metrics from local DB + remote services, calculates growth.
    - `RankingService.java` – reads Redis sorted sets (with mock fallback) and maps to DTOs.
    - `RankingUpdateService.java` – scheduled job to rebuild Redis rankings using content/user/gamification services.
    - `HistoryService.java` – persists reading history, syncs library progress if novel in library.
  - `client/` – Feign clients for content/user/gamification/engagement services.
  - `dao/` – `AnalyticsMapper`, `HistoryMapper`.
  - `dto/` – analytics summaries, daily trends, platform stats, ranking DTOs, history responses.
  - `entity/` – `History`.
  - `util/` – `RedisUtil`, `SecurityUtils`.
- **Tech**: Spring Boot, MyBatis, Redis (rankings), scheduled jobs.
- **Clients**: `ContentServiceClient`, `UserServiceClient`, `GamificationServiceClient`, `EngagementServiceClient`.
- **Controllers**:
  - `AnalyticsController` – `/api/v1/admin/analytics/**`.
  - `HistoryController` – `/api/v1/history/**`.
  - `RankingController` – `/api/v1/ranking/**`.
- **Ranking Flow**:
  - `RankingUpdateService` scheduled daily: fetches all novels/users/authors, stores scores in Redis sorted sets (keys `ranking:novel:view:*`, etc.).
  - `RankingService` queries Redis (with fallback to mock data) and returns sorted results.
- **History Service**: persists reading history, integrates with library progress update.
- **Dashboard Metrics**: aggregated counts, DAU/WAU/MAU, top content.
- **Config**: `configs/analytics-service.yml`.

#### 2.6 DMZ Services
- **API Gateway (`yushan-api-gateway`)**
  - Routes defined in `src/main/resources/application.yml`.
  - Path rewriting per service (`lb://service-name`) with `spring.cloud.gateway.discovery.locator`.
  - Logging filter for request/response metrics.
  - Key files: `YushanApiGatewayApplication.java`, `filter/LoggingFilter.java`, `config/CorsConfig.java`, `application.yml`, Dockerfile/Nginx config.
- **Config Server (`yushan-config-server`)**
  - Native profile serving from `configs/` folder.
  - `application.yml` uses `EUREKA_URL` env with default `http://yushan-eureka-registry:8761/eureka/`.
  - `YushanConfigServerApplication.java`, Dockerfile, `configs/*.yml`.
- **Service Registry (`yushan-platform-service-registry`)**
  - Standard Eureka server with `docker-compose.yml` for container deployment.
  - `ServiceRegistryApplication.java`, `application.yml`, Dockerfile.

---

### 3. Admin Frontend (`yushan-platform-admin`)

- **Stack**: React 18, Ant Design v5, Axios, Zustand (unused), CRA build.
- **Project Layout**
  - `src/App.js` – routing (`/admin/login`, `/admin/**`).
  - `contexts/admin/` – `adminauthcontext`, theme, notification contexts.
  - `components/admin/`
    - `common/` – `AdminHeader`, `AdminSidebar`, `ErrorBoundary`, `NotificationDropdown`, layout helpers.
    - `charts/` – `StatCard`, `LineChart`, `AreaChart`, `BarChart`, `PieChart`, wrappers/tests.
    - `tables/` – reusable data table, pagination, filters, export button, bulk actions.
    - `modals/` – library/novel/comment/report modals for CRU operations.
  - `hooks/admin/` – `useAdminAuth`, `useDashboardData`, `useDataTable`, `useExport`, `useFilters`, `usePagination`, `useResponsive`.
  - `pages/admin/`
    - `adminlayout.jsx` – shared layout with auth guard.
    - `dashboard/index.jsx` – metrics & charts.
    - `users/` – overview/readers/writers/change status/promote admin.
    - `novels/`, `chapters/`, `categories/` – content administration.
    - `comments/`, `reviews/`, `reports/` – moderation.
    - `yuan/` – Yuan transactions/stats.
    - `library/`, `rankings/`, `settings/`, `profile/`, `login`.
  - `services/admin/`
    - `api.js` (axios instance), `authservice.js`, `userservice.js`, `novelservice.js`, `chapterservice.js`, `commentservice.js`, `reviewservice.js`, `reportservice.js`, `libraryservice.js`, `analyticsservice.js`, `rankingservice.js`, `dashboardservice.js`, etc.
    - `mockapi.js` retained for fallback/testing.
  - `styles/admin/` – CSS for layout/sidebar/dashboard/modal/datatable/mobile.
  - `setupProxy.js` – proxies `/api/v1` to remote host for local dev.
  - Tests: nearly every component/hook/service has companion `.test.jsx`/`.test.js`.
- **Auth Flow**:
  - `authservice.js` – login on `/auth/login` returning `accessToken`, `refreshToken`; ensures `isAdmin`.
  - Refresh timer every 15 min via `setTimeout`, axios interceptor handles 401 by refreshing or redirecting.
  - `AdminAuthContext` stores admin account, `AdminLayout` enforces login check.
- **API Base**:
  - Defaults to `https://yushan.duckdns.org/api/v1`, override via `REACT_APP_API_BASE_URL`.
  - `setupProxy.js` proxies `/api/v1` to same host for local dev.
- **Routing**:
  - `/admin/login` (public), `/admin/**` behind `AdminLayout`.
  - Feature pages under `src/pages/admin` for users, novels, chapters, categories, comments, reviews, library, rankings, reports, settings, Yuan.
- **Dashboard**:
  - `dashboard/index.jsx` fetches stats via `dashboardService`, `analyticsservice`, `rankingservice`.
  - Has mock novel trends (hardcoded) since API unavailable; merges real analytics data where present.
- **Services**:
  - `services/admin/*` wrappers: `userservice`, `novelservice`, `chapterservice`, `commentservice`, `analyticsservice`, `rankingservice`, `libraryservice`, `reportservice`, `reviewservice`, etc.
  - Real endpoints; some methods fall back to mock data for unsupported features (noted inside modules).
- **Hooks**: `useDataTable`, `useDashboardData`, `useBulkActions`, `usePagination` standardize UI behaviour.
- **Testing**: Jest & RTL tests per component/hook/service with `__tests__` directories; mocks under `src/__mocks__`.
- **Styling**: CSS modules under `styles/admin` (layout, sidebar, modals).

---

### 4. User Frontend (`yushan-platform-frontend`)

- **Stack**: CRA, React 19 (compat), Redux Toolkit + Persist, Ant Design, axios, rate-limited http clients.
- **Project Layout**
  - `src/app.js` – global providers (Ant theme, PersistGate, ReadingSettingsProvider), routing (protected wrappers).
  - `store/`
    - `index.js` / `rootReducer.js` / `slices/user.js`.
    - `readingSettings.js` – context storing font settings.
    - `UserContext.js` – fetches `/users/me` for writer views.
  - `services/`
    - `_http.js`, `httpClient.js` – axios wrappers + refresh handling.
    - Feature services: `auth.js`, `novels.js`, `chapter.js`, `comments.js`, `reviews.js`, `reports.js`, `rankings.js`, `history.js`, `library.js`, `gamification.js`, `user.js`, `vote.js`, `search.js`, `categories.js`.
    - `api/novels.js` – home page novel fetchers using rate-limited client.
    - `mockRankings.js` – fallback data for empty ranking responses.
  - `components/`
    - `common/` – `Navbar`, `Footer`, `LayoutWrapper`, `ContentPopover`.
    - `auth/` – shared login/register form.
    - `novel/` – hero section, feature grids, novel cards, browse filters.
    - `leaderboard/` – filters/list components.
    - `user/icons` – icon set for profile menus.
    - `writer/` – writer navbar.
  - `pages/`
    - Auth: `login`, `register`, `writerauth`.
    - Reader: `home`, `browse`, `novel`, `reader`, `library`, `leaderboard`, `profile`, `editprofile`, `settings/reading-settings`.
    - Writer: `writerdashboard`, `writerworkspace`, `writercreate`, `writercreatechapters`, `writerinteraction`, `writerstoryprofile`.
    - Info: `terms`, `cookies`, `affliate-programme`.
  - `utils/`
    - `axios-interceptor.js`, `reader.js` (local progress persistence), `imageUtils.js`, `constants.js`, `helpers.js`, `time.js`, `token.js`, `validators.js`.
  - `assets/images/` – default covers, icons, logos.
- **Auth**:
  - `services/auth.js` (login/register/logout/refresh/email): tokens stored as `jwt_token`, `refresh_token`, `token_expiry`.
  - `utils/axios-interceptor.js` attaches tokens and handles refresh queue; on failure clears tokens and sends user to `/login?expired=true`.
  - `ProtectedRouteWrapper` in `app.js` ensures auth.
- **API Base**:
  - `process.env.REACT_APP_API_URL` (expected to be `https://.../api`); fallback to `/api`.
  - `IMAGE_BASE_URL` derived for cover images (fallback to `/api/v1/images`).

- **Global Providers**:
  - `ConfigProvider` (Ant theme).
  - `PersistGate` for Redux `user` slice.
  - `ReadingSettingsProvider` (font size/family stored in localStorage).
  - `UserProvider` fetches `/users/me` for writer context.

- **Core Pages**:
  - `/` – `Home`: loads featured/ongoing/completed/newest novels via `services/api/novels`.
  - `/browse` – filters/search.
  - `/novel/:novelId` – detail page (includes view counts, chapters).
  - `/read/:novelId/:chapterId` – interactive reader (auto-save progress, comments).
  - `/library` – list/infinite scroll with edit mode; `/history` tab.
  - `/rankings/*` – leaderboard (novels/users/authors) powered by `services/rankings`.
  - `/profile`, `/editprofile` – user details.
  - `/writer*` – writer dashboard/tools (create novels/chapters).

- **Services Layer**:
  - `_http.js` – base axios with refresh queue logic.
  - `httpClient.js` – rate-limited clients (default, heavy, light).
  - `novels.js`, `chapter.js`, `comments.js`, `reviews.js`, `vote.js`, `library.js`, `history.js`, `gamification.js`, `rankings.js`, `user.js`, `search.js`, etc.
  - Comments/likes handled with local liked-state caching.
  - History service updates library progress when recording read events.
  - Gamification service fetches stats/achievements for display (XP/Yuan).

- **Reading Experience** (`pages/reader/reader.jsx`):
  - Fetches chapter content via `novelsApi.getChapterContent`.
  - Records reading progress in localStorage (per novel) and backend history.
  - Comments: list/create/edit/delete, likes (with `likedComments_{uuid}` local storage).
  - Keyboard navigation (`[` `]`), reading settings panel (font size/family).
  - Progress autosave every 2.5s and on unload.

- **Library** (`pages/library/library.jsx`):
  - Tabs: library vs history with infinite scroll; edit mode to remove novels; delete/clear history.
  - Utilizes `libraryService.getLibraryNovels` and `historyService.getHistoryNovels`.

- **Redux**:
  - `store/index.js` configures persisted store with `user` slice.
  - Actions: `login`, `logout`, `setAuthenticated`, `updateUser`, `updateYuan`.

- **Miscellaneous**:
  - `services/api/novels.js` normalizes novel data, handles image fallback.
  - `services/mockRankings.js` provides data when backend returns empty results.
  - `utils/imageUtils.js` ensures default covers.
  - `config/images.js` centralizes image URL base.
  - Writer components rely on `UserContext` for avatar/username in nav.

---

### 5. Communication & Data Flows

- **JWT Propagation**: Every service Feign client installs a request interceptor forwarding `Authorization` from incoming requests. Frontends attach tokens via axios interceptors and refresh automatically.

- **Kafka Eventing**:
  - `user-service` → topic `user.events` → `gamification-service` applies initial rewards & daily login.
  - `engagement-service` → topics `comment-events`, `review-events`, `vote-events` → `gamification-service` increments EXP/Yuan and triggers achievements.
  - `gamification-service` publishes internal level-up events to `internal_gamification_events` consumed by itself.
  - `user-service` & `gamification-service` send `UserActivityEvent` to topic `active` (analytics/monitoring).

- **Redis Usage**:
  - Caching (user activity throttling, library check, session data).
  - Analytics service stores ranking sorted sets (novel views/votes, user EXP, author stats).
  - Gamification may use Redis for leaderboard caching (per YAML config).

- **Inter-service REST Calls**:
  - User ↔ Content (author identity, library operations).
  - Engagement → Content/User/Gamification for validations and vote checks.
  - Gamification → (none via Feign) listens only to Kafka, but frontends hit gamification APIs directly.
  - Analytics → Content/User/Gamification/Engagement to aggregate stats and ranking data.

- **Database Schema Highlights**:
  - User Service: `users`, `library`, `novel_library`.
  - Content Service: `novels`, `chapters`, `categories`, join tables for tags/genres, Elasticsearch indexes `novels`, `chapters`.
  - Engagement Service: `comments`, `reviews`, `votes`, `reports`.
  - Gamification Service: `achievements`, `user_achievements`, `exp_transactions`, `yuan_transactions`, `daily_reward_log`.
  - Analytics Service: `history` table storing reading sessions.

- **Config Server**:
  - Provides environment-specific DB credentials (`DB_HOST`, `DB_PORT`, etc.), Redis settings, Kafka servers, JWT secrets, Resilience4j configs.
  - Each service reads from `configs/<service>.yml` and environment overrides.

---

### 6. Frontend ↔ Backend Endpoint Mapping (Key)

| Frontend Module                              | Backend Service & Endpoint(s)                                                                 |
|----------------------------------------------|-------------------------------------------------------------------------------------------------|
| `admin/services/admin/authservice`           | `user-service`: `/auth/login`, `/auth/logout`, `/auth/refresh`                                 |
| `admin/services/admin/userservice`           | `user-service`: `/users/me`, `/admin/users`, `/admin/promote-to-admin`, `/admin/users/{uuid}`  |
| `admin/services/admin/novelservice`          | `content-service`: `/admin/novels`, `/chapters`, moderation endpoints                         |
| `admin/services/admin/commentservice`        | `engagement-service`: `/comments/**` (moderation)                                              |
| `admin/services/admin/analyticsservice`      | `analytics-service`: `/admin/analytics/**`, `/admin/analytics/platform/dau`                   |
| `admin/services/admin/rankingservice`        | `analytics-service`: `/ranking/{novel|user|author}`                                            |
| `frontend/services/auth`                     | `user-service`: `/auth/login`, `/auth/register`, `/auth/refresh`, `/auth/send-email`, `/auth/logout` |
| `frontend/services/novels` / `_api/novels`   | `content-service`: `/novels`, `/novels/{id}`, `/chapters/**`, `/novels/{id}/view`, `/novels/{id}/vote` |
| `frontend/services/chapter`                  | `content-service`: `/chapters`, `/chapters/{uuid}`, `/chapters/{uuid}/view`                    |
| `frontend/services/comments`                 | `engagement-service`: `/comments/**`                                                           |
| `frontend/services/review(s)`                | `engagement-service`: `/reviews/**`                                                            |
| `frontend/services/history`                  | `analytics-service`: `/history/**`                                                             |
| `frontend/services/library`                  | `user-service`: `/library/**`                                                                  |
| `frontend/services/gamification`             | `gamification-service`: `/gamification/stats/me`, `/achievements/me`                           |
| `frontend/services/rankings`                 | `analytics-service`: `/ranking/{novel|user|author}`                                            |
| `frontend/services/user`                     | `user-service`: `/users/me`, `/author/send-email-author-verification`, `/author/upgrade-to-author` |
| `frontend/services/vote`                     | `engagement-service` + `gamification-service`: `/votes/novels/{id}`, `/users/votes`            |

---

### 7. Environment & Deployment

- **Environment Variables (Frontends)**:
  - Admin: `REACT_APP_API_BASE_URL` (default `https://yushan.duckdns.org/api/v1`), `PUBLIC_URL` for GH Pages.
  - Frontend: `REACT_APP_API_URL` (base `/api` by default), `REACT_APP_IMAGE_BASE_URL`, `REACT_APP_APP_NAME`.

- **Frontends Deployment**:
  - Dockerfile (Nginx) + `nginx.conf`.
  - `package.json` `deploy` script uses GitHub Pages (branch `gh-pages`).

- **Microservices Deployment**:
  - Standard Spring Boot artifacts (Maven wrapper) with Dockerfiles, `docker-compose.yml`.
  - Terraform scripts in `Digital_Ocean_Deployment_with_Terraform` manage infrastructure, load balancer, monitoring stack (Prometheus, Grafana, ELK, Alertmanager).

---

### 8. Useful References

- `configs/*.yml` – authoritative configuration for each service.
- `src/main/java/.../config/SecurityConfig.java` (per service) – outlines whitelisted endpoints.
- Frontend `services/` directory – view which endpoints are consumed.
- Kafka topics defined in `Kafka...Config` and services’ `KafkaEventProducer/Listener`.
- Redis keys: ranking keys (`ranking:*`), library caches (`active{uuid}`), etc.
- Readiness: `README.md` in each service covers prerequisites, setup, API summary.

---

Keep this document updated when adding new endpoints, services, or major architectural changes.

---

### 9. Supplemental Repositories in Workspace

- **Digital_Ocean_Deployment_with_Terraform/**
  - `yushan-services/` – Terraform modules for app services, load balancer (`load-balancer.tf`), environment templates (`templates/nginx.conf.tftpl`), deployment scripts (`deploy.sh`).
    - `business-services.tf` – defines Droplets/Kubernetes for user/content/engagement/gamification/analytics services.
    - `databases.tf` – managed PostgreSQL/Redis resources.
    - `infrastructure.tf` – VPC, firewall, droplet provisioning basics.
    - `env.example` – expected environment variables for provisioning scripts.
    - `deploy.sh` – CLI helper to init/apply terraform & sync env files.
  - `yushan-monitoring/` – Terraform + docker-compose for monitoring stack (Prometheus, Grafana, Alertmanager, Logstash); templates for `alertmanager.yml`, `filebeat.yml`.
    - `monitoring.tf` – creates monitoring Droplet, security groups.
    - `docker-compose.yml` – runs Prometheus/Grafana/ELK stack.
    - `templates/` – configuration for Prometheus scrape targets (microservices), Filebeat shipping, Logstash pipeline.
  - Shared files: `providers.tf`, `variables.tf`, `outputs.tf`, `README.md`, `DEPLOYMENT_GUIDE.md`, `env.example`.
    - `README.md`/`DEPLOYMENT_GUIDE.md` – step-by-step instructions (Terraform init/apply, environment export, SSH).
- **Deployment Flow Overview**
  1. Prepare `.tfvars` from `env.example`, configure DO token & service endpoints.
  2. Run `./deploy.sh` (invokes `terraform init/plan/apply`) per module.
  3. Terraform provisions:
     - Droplets (or Kubernetes) for services with load balancer pointing to API gateway.
     - Managed PostgreSQL/Redis (if configured).
     - Monitoring stack droplet with docker-compose.
  4. Nginx template (`templates/nginx.conf.tftpl`) used for gateway/ingress.
  5. Outputs include IPs/URLs; GitHub Actions or manual `docker-compose` used to deploy microservices onto droplets.
- **Yushan_Web_Novel_Design_Documents/**
  - Markdown design docs: `OVERVIEW.md`, `LOGICAL_ARCHITECTURE_DESIGN.md`, `PHYSICAL_ARCHITECTURE_DESIGN.md`, `DEVOPS_DEVELOPMENT_LIFECYCLE.md`, visual diagrams (`YUSHAN_*_DIAGRAM*.md`), `PROJECT_CONDUCT.md`.


