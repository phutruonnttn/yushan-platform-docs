# Yushan Platform Documentation

> 📚 **Complete documentation and architecture guide for Yushan Platform** - A gamified web novel reading platform with monolithic (Phase 1) and microservices (Phase 2) architectures, with Phase 3 planned for Kubernetes and AWS deployment.

## 📋 Overview

This repository contains comprehensive documentation for the **Yushan Platform** - a gamified web novel reading platform that transforms reading into an engaging, social experience. The platform has been developed in multiple phases, each representing a significant architectural evolution.

## 🎯 Project Phases

### Phase 1: Monolithic Architecture ✅ **Complete**

**Status**: ✅ Production Ready | **Deployment**: Railway (Backend)

**Description**: Initial implementation with all features in a single application.

**Repositories**:
- [yushan-monolithic-backend](https://github.com/phutruonnttn/yushan-monolithic-backend) - Spring Boot 3.5.7, Java 21, PostgreSQL, Redis
- [yushan-monolithic-frontend](https://github.com/phutruonnttn/yushan-monolithic-frontend) - React 19, Redux Toolkit, Ant Design
- [yushan-monolithic-admin-dashboard](https://github.com/phutruonnttn/yushan-monolithic-admin-dashboard) - React 18, Ant Design

**Key Features**:
- Complete user management and authentication
- Novel and chapter management
- Social features (comments, reviews, votes)
- Gamification (XP, achievements, leaderboards)
- Analytics and rankings
- Admin dashboard

**Tech Stack**:
- Backend: Spring Boot 3.5.7, Java 21, PostgreSQL, Redis, JWT
- Frontend: React 19, Redux Toolkit, Ant Design
- Deployment: Railway (Backend), GitHub Pages (Frontend)

---

### Phase 2: Microservices Architecture ✅ **Complete**

**Status**: ✅ Production Ready | **Deployment**: Digital Ocean (Backend)

**Description**: Refactored into microservices architecture for better scalability and maintainability.

**Infrastructure Services**:
- [yushan-microservices-service-registry](https://github.com/phutruonnttn/yushan-microservices-service-registry) - Eureka Service Discovery
- [yushan-microservices-config-server](https://github.com/phutruonnttn/yushan-microservices-config-server) - Spring Cloud Config Server
- [yushan-microservices-api-gateway](https://github.com/phutruonnttn/yushan-microservices-api-gateway) - Spring Cloud Gateway

**Business Services**:
- [yushan-microservices-user-service](https://github.com/phutruonnttn/yushan-microservices-user-service) - User management & authentication
- [yushan-microservices-content-service](https://github.com/phutruonnttn/yushan-microservices-content-service) - Novel & chapter management
- [yushan-microservices-engagement-service](https://github.com/phutruonnttn/yushan-microservices-engagement-service) - Comments, reviews, votes
- [yushan-microservices-gamification-service](https://github.com/phutruonnttn/yushan-microservices-gamification-service) - XP, achievements, Yuan
- [yushan-microservices-analytics-service](https://github.com/phutruonnttn/yushan-microservices-analytics-service) - Analytics & rankings

**Frontend Applications**:
- [yushan-microservices-frontend](https://github.com/phutruonnttn/yushan-microservices-frontend) - Reader-facing React app
- [yushan-microservices-admin-dashboard](https://github.com/phutruonnttn/yushan-microservices-admin-dashboard) - Admin dashboard

**Key Features**:
- Service discovery with Eureka
- Centralized configuration
- API Gateway for routing
- Event-driven architecture with Kafka
- Distributed caching with Redis
- Full-text search with Elasticsearch
- Circuit breakers with Resilience4j

**Tech Stack**:
- Backend: Spring Boot 3.4.10, Spring Cloud 2024.0.2, Java 21
- Database: PostgreSQL 15+ (per service)
- Cache: Redis 7
- Search: Elasticsearch 8.11
- Message Queue: Apache Kafka
- Deployment: Digital Ocean (Terraform), Docker

**Deployment & Infrastructure**:
- [Digital Ocean Deployment with Terraform](https://github.com/phutruonnttn/Digital_Ocean_Deployment_with_Terraform) - Infrastructure as Code
  - Service deployment automation
  - Monitoring stack (Prometheus, Grafana, ELK)
  - Load balancer configuration

**Architecture Documentation**:
- [Phase 2 Microservices Architecture](./docs/PHASE2_MICROSERVICES_ARCHITECTURE.md) - Detailed architecture notes

---

### Phase 3: Kubernetes & AWS Deployment 🔄 **Planned**

**Status**: 🔄 Planning Phase

**Description**: Advanced microservices architecture with Kubernetes orchestration, distributed tracing, Saga pattern, and AWS deployment.

**Planned Features**:
- Kubernetes orchestration
- Distributed tracing (Jaeger/Zipkin)
- Saga pattern for distributed transactions
- Service mesh (Istio/Linkerd)
- Advanced monitoring and observability
- AWS deployment (EKS, RDS, ElastiCache, etc.)
- Fix outstanding issues from Phase 2
- Performance optimizations
- Enhanced security

**Target Technologies**:
- Orchestration: Kubernetes (EKS)
- Tracing: Jaeger or Zipkin
- Service Mesh: Istio or Linkerd
- Cloud Provider: AWS
- Database: AWS RDS (PostgreSQL)
- Cache: AWS ElastiCache (Redis)
- Message Queue: AWS MSK (Kafka)
- Search: AWS OpenSearch (Elasticsearch)
- Storage: AWS S3
- Monitoring: AWS CloudWatch, Prometheus, Grafana

---

## 📚 Documentation Structure

```
yushan-platform-docs/
├── README.md (this file)
├── docs/
│   ├── phase1-monolithic/
│   │   ├── architecture.md
│   │   ├── deployment.md
│   │   └── api-documentation.md
│   ├── phase2-microservices/
│   │   ├── PHASE2_MICROSERVICES_ARCHITECTURE.md
│   │   ├── deployment.md
│   │   └── service-communication.md
│   ├── phase3-kubernetes/
│   │   ├── planning.md
│   │   └── roadmap.md
│   └── guides/
│       ├── getting-started.md
│       ├── development-setup.md
│       └── contributing.md
└── diagrams/
    └── architecture-diagrams/
```

## 🔗 Quick Links

### Phase 1 (Monolithic)
- **Backend**: [yushan-monolithic-backend](https://github.com/phutruonnttn/yushan-monolithic-backend)
- **Frontend**: [yushan-monolithic-frontend](https://github.com/phutruonnttn/yushan-monolithic-frontend)
- **Admin Dashboard**: [yushan-monolithic-admin-dashboard](https://github.com/phutruonnttn/yushan-monolithic-admin-dashboard)

### Phase 2 (Microservices)
- **Service Registry**: [yushan-microservices-service-registry](https://github.com/phutruonnttn/yushan-microservices-service-registry)
- **Config Server**: [yushan-microservices-config-server](https://github.com/phutruonnttn/yushan-microservices-config-server)
- **API Gateway**: [yushan-microservices-api-gateway](https://github.com/phutruonnttn/yushan-microservices-api-gateway)
- **User Service**: [yushan-microservices-user-service](https://github.com/phutruonnttn/yushan-microservices-user-service)
- **Content Service**: [yushan-microservices-content-service](https://github.com/phutruonnttn/yushan-microservices-content-service)
- **Engagement Service**: [yushan-microservices-engagement-service](https://github.com/phutruonnttn/yushan-microservices-engagement-service)
- **Gamification Service**: [yushan-microservices-gamification-service](https://github.com/phutruonnttn/yushan-microservices-gamification-service)
- **Analytics Service**: [yushan-microservices-analytics-service](https://github.com/phutruonnttn/yushan-microservices-analytics-service)
- **Frontend**: [yushan-microservices-frontend](https://github.com/phutruonnttn/yushan-microservices-frontend)
- **Admin Dashboard**: [yushan-microservices-admin-dashboard](https://github.com/phutruonnttn/yushan-microservices-admin-dashboard)

### Infrastructure & Deployment
- **Terraform Deployment**: [Digital_Ocean_Deployment_with_Terraform](https://github.com/phutruonnttn/Digital_Ocean_Deployment_with_Terraform)
  - [Services Deployment](https://github.com/phutruonnttn/Digital_Ocean_Deployment_with_Terraform/tree/main/yushan-services)
  - [Monitoring Stack](https://github.com/phutruonnttn/Digital_Ocean_Deployment_with_Terraform/tree/main/yushan-monitoring)

### Design Documents
- **Design Documents**: [Yushan_Web_Novel_Design_Documents](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents)
  - [Overview](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents/blob/main/OVERVIEW.md)
  - [Logical Architecture](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents/blob/main/LOGICAL_ARCHITECTURE_DESIGN.md)
  - [Physical Architecture](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents/blob/main/PHYSICAL_ARCHITECTURE_DESIGN.md)
  - [DevOps Lifecycle](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents/blob/main/DEVOPS_DEVELOPMENT_LIFECYCLE.md)
  - [Project Conduct](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents/blob/main/PROJECT_CONDUCT.md)

## 🏗️ Architecture Overview

### Phase 1: Monolithic Architecture

```
┌─────────────────────────────────────┐
│      Yushan Monolithic Backend      │
│  (Spring Boot - Single Application) │
│                                     │
│  ┌───────────────────────────────┐  │
│  │  All Services in One App      │  │
│  │  - User Management           │  │
│  │  - Content Management         │  │
│  │  - Engagement                 │  │
│  │  - Gamification               │  │
│  │  - Analytics                  │  │
│  └───────────────────────────────┘  │
│                                     │
│  Database: PostgreSQL               │
│  Cache: Redis                       │
└─────────────────────────────────────┘
```

### Phase 2: Microservices Architecture

```
┌─────────────────────────────────────────────────────────┐
│         YUSHAN MICROSERVICES PLATFORM                  │
├─────────────────────────────────────────────────────────┤
│  Infrastructure Services (3)                           │
│  ├─ Eureka Registry (:8761)                           │
│  ├─ Config Server (:8888)                              │
│  └─ API Gateway (:8080)                                │
├─────────────────────────────────────────────────────────┤
│  Business Services (5)                                │
│  ├─ User Service (:8081)                              │
│  ├─ Content Service (:8082)                           │
│  ├─ Engagement Service (:8084)                         │
│  ├─ Gamification Service (:8085)                       │
│  └─ Analytics Service (:8083)                         │
├─────────────────────────────────────────────────────────┤
│  Supporting Infrastructure                             │
│  ├─ PostgreSQL 16 (Per Service)                       │
│  ├─ Redis 7 (Caching & Sessions)                     │
│  ├─ Elasticsearch 8.11 (Search)                       │
│  ├─ Apache Kafka (Event Streaming)                    │
│  ├─ Digital Ocean Spaces (File Storage)               │
│  ├─ Prometheus + Grafana (Monitoring)                 │
│  └─ ELK Stack (Logging)                               │
└─────────────────────────────────────────────────────────┘
```

### Phase 3: Kubernetes Architecture (Planned)

```
┌─────────────────────────────────────────────────────────┐
│         YUSHAN KUBERNETES PLATFORM (Planned)            │
├─────────────────────────────────────────────────────────┤
│  Kubernetes Cluster (AWS EKS)                          │
│  ├─ Service Mesh (Istio/Linkerd)                       │
│  ├─ Distributed Tracing (Jaeger/Zipkin)                │
│  └─ Observability Stack                                 │
├─────────────────────────────────────────────────────────┤
│  Microservices (Containerized)                          │
│  ├─ User Service Pods                                  │
│  ├─ Content Service Pods                               │
│  ├─ Engagement Service Pods                            │
│  ├─ Gamification Service Pods                          │
│  └─ Analytics Service Pods                             │
├─────────────────────────────────────────────────────────┤
│  AWS Services                                           │
│  ├─ RDS (PostgreSQL)                                   │
│  ├─ ElastiCache (Redis)                                │
│  ├─ MSK (Kafka)                                        │
│  ├─ OpenSearch (Elasticsearch)                         │
│  ├─ S3 (File Storage)                                  │
│  └─ CloudWatch (Monitoring)                           │
└─────────────────────────────────────────────────────────┘
```

## 📖 Documentation by Phase

### Phase 1 Documentation
- **Architecture**: Monolithic Spring Boot application
- **Deployment**: Railway (Backend), GitHub Pages (Frontend)
- **Status**: ✅ Production Ready

### Phase 2 Documentation
- **Architecture**: [Phase 2 Microservices Architecture](./docs/PHASE2_MICROSERVICES_ARCHITECTURE.md)
- **Deployment**: [Digital Ocean Deployment Guide](https://github.com/phutruonnttn/Digital_Ocean_Deployment_with_Terraform)
- **Design Documents**: [Yushan Design Documents](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents)
- **Status**: ✅ Production Ready

### Phase 3 Documentation
- **Planning**: [Phase 3 Planning](./docs/phase3-kubernetes/planning.md) (Coming Soon)
- **Roadmap**: [Phase 3 Roadmap](./docs/phase3-kubernetes/roadmap.md) (Coming Soon)
- **Status**: 🔄 Planning Phase

## 🚀 Getting Started

### For New Developers

1. **Start with Phase 1** (Monolithic):
   - Read: [yushan-monolithic-backend README](https://github.com/phutruonnttn/yushan-monolithic-backend)
   - Understand the complete system in one codebase
   - Deploy locally or on Railway

2. **Move to Phase 2** (Microservices):
   - Read: [Phase 2 Architecture](./docs/PHASE2_MICROSERVICES_ARCHITECTURE.md)
   - Review: [Design Documents](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents)
   - Setup: [Deployment Guide](https://github.com/phutruonnttn/Digital_Ocean_Deployment_with_Terraform)

3. **Plan for Phase 3** (Kubernetes):
   - Review Phase 2 issues and improvements
   - Study Kubernetes and service mesh patterns
   - Prepare for AWS deployment

### For Architects

1. **Overview**: [Design Documents - Overview](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents/blob/main/OVERVIEW.md)
2. **Logical Architecture**: [Logical Architecture Design](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents/blob/main/LOGICAL_ARCHITECTURE_DESIGN.md)
3. **Physical Architecture**: [Physical Architecture Design](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents/blob/main/PHYSICAL_ARCHITECTURE_DESIGN.md)
4. **Phase 2 Details**: [Phase 2 Microservices Architecture](./docs/PHASE2_MICROSERVICES_ARCHITECTURE.md)

### For DevOps Engineers

1. **Deployment**: [Digital Ocean Terraform Deployment](https://github.com/phutruonnttn/Digital_Ocean_Deployment_with_Terraform)
2. **Monitoring**: [Monitoring Stack Setup](https://github.com/phutruonnttn/Digital_Ocean_Deployment_with_Terraform/tree/main/yushan-monitoring)
3. **CI/CD**: [DevOps Lifecycle](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents/blob/main/DEVOPS_DEVELOPMENT_LIFECYCLE.md)

## 📊 Project Status Summary

| Phase | Status | Completion | Deployment | Notes |
|-------|--------|------------|------------|-------|
| **Phase 1** | ✅ Complete | 100% | Railway (BE) | Monolithic architecture, fully functional |
| **Phase 2** | ✅ Complete | 95% | Digital Ocean | Microservices, production-ready |
| **Phase 3** | 🔄 Planned | 0% | AWS (Planned) | Kubernetes, distributed tracing, Saga pattern |

## 🔧 Technology Evolution

### Phase 1 → Phase 2
- **Architecture**: Monolithic → Microservices
- **Deployment**: Railway → Digital Ocean (Terraform)
- **Communication**: Direct calls → Service discovery + API Gateway
- **Data**: Single database → Database per service
- **Events**: Synchronous → Asynchronous (Kafka)

### Phase 2 → Phase 3 (Planned)
- **Orchestration**: Docker Compose → Kubernetes
- **Deployment**: Digital Ocean → AWS
- **Tracing**: None → Distributed tracing (Jaeger/Zipkin)
- **Transactions**: Local → Saga pattern
- **Service Mesh**: None → Istio/Linkerd
- **Monitoring**: Prometheus/Grafana → Enhanced observability

## 📝 Key Documents

### Architecture & Design
- [Phase 2 Microservices Architecture](./docs/PHASE2_MICROSERVICES_ARCHITECTURE.md) - Detailed microservices architecture notes
- [Design Documents Repository](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents) - Complete design documentation

### Deployment
- [Digital Ocean Deployment](https://github.com/phutruonnttn/Digital_Ocean_Deployment_with_Terraform) - Infrastructure as Code
- [Monitoring Stack](https://github.com/phutruonnttn/Digital_Ocean_Deployment_with_Terraform/tree/main/yushan-monitoring) - ELK + Prometheus/Grafana

### Development
- [Project Conduct](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents/blob/main/PROJECT_CONDUCT.md) - Project status, issues, milestones
- [DevOps Lifecycle](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents/blob/main/DEVOPS_DEVELOPMENT_LIFECYCLE.md) - CI/CD pipeline details

## 🎯 Platform Features

### Core Features (All Phases)
- 📖 Novel reading and management
- 👤 User authentication and profiles
- 💬 Social interactions (comments, reviews, votes)
- 🎮 Gamification (XP, achievements, leaderboards)
- 📊 Analytics and rankings
- 🔍 Full-text search
- 🛠️ Admin dashboard

### Phase 2 Enhancements
- ⚡ Event-driven architecture
- 🔄 Circuit breakers and resilience
- 📈 Advanced monitoring and logging
- 🔍 Elasticsearch search
- 🚀 Independent service scaling

### Phase 3 Planned Enhancements
- 🔍 Distributed tracing
- 🔄 Saga pattern for transactions
- 🌐 Service mesh
- ☁️ Cloud-native AWS services
- 📊 Enhanced observability

## 🤝 Contributing

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## 📄 License

This project is part of the Yushan Platform ecosystem.

## 🔗 Related Repositories

### Phase 1 (Monolithic)
- [Backend](https://github.com/phutruonnttn/yushan-monolithic-backend)
- [Frontend](https://github.com/phutruonnttn/yushan-monolithic-frontend)
- [Admin Dashboard](https://github.com/phutruonnttn/yushan-monolithic-admin-dashboard)

### Phase 2 (Microservices)
- [Service Registry](https://github.com/phutruonnttn/yushan-microservices-service-registry)
- [Config Server](https://github.com/phutruonnttn/yushan-microservices-config-server)
- [API Gateway](https://github.com/phutruonnttn/yushan-microservices-api-gateway)
- [User Service](https://github.com/phutruonnttn/yushan-microservices-user-service)
- [Content Service](https://github.com/phutruonnttn/yushan-microservices-content-service)
- [Engagement Service](https://github.com/phutruonnttn/yushan-microservices-engagement-service)
- [Gamification Service](https://github.com/phutruonnttn/yushan-microservices-gamification-service)
- [Analytics Service](https://github.com/phutruonnttn/yushan-microservices-analytics-service)
- [Frontend](https://github.com/phutruonnttn/yushan-microservices-frontend)
- [Admin Dashboard](https://github.com/phutruonnttn/yushan-microservices-admin-dashboard)

### Infrastructure & Documentation
- [Terraform Deployment](https://github.com/phutruonnttn/Digital_Ocean_Deployment_with_Terraform)
- [Design Documents](https://github.com/phutruonnttn/Yushan_Web_Novel_Deisgn_Documents)

---

**Yushan Platform Documentation** - Complete guide to the gamified web novel reading platform 🚀

**Last Updated**: November 2025

