# Bajaj_Life_Assignment
Develop a “Enterprise Usage Monitoring &amp; Admin Platform”

**Functiional Requirenment**

1. Ingest and track user and feature usage across enterprises in near real time.
2. Usage dashboards showing active users, feature usage and trends
3. Ability to define usage limits and access policies
4. Alerts for high usage or abnormal behavior
5. Maintain audit logs for user and admin actions

**Non Functional Requirenments:**

1. System should scale to large enterprises and high data volume
2.Strong security with tenant isolation and encryption
3.Low latency for ingestion and dashboards
4.Compliant with enterprise privacy and retention requirements
5.Good observability for monitoring system health
6.Highly reliable with no loss of usage data


**Capacity Estimation :**
Target Customers: Large,small,Mid Scale enterprise

Assumptions:1000Enterprise(small:50%,Mid:30%,Large:20%)
Average user per Enterprise:
small=500
mid=2000
large=10000

Avg usage events per user per day: 40

Avg event size: 1 KB

Peak traffic factor: 5× average
Total small enterprises=500
Total Mid Enterprises=300
Total Large Enterprises=200
Total users= 250000+600000+2000000
≈2.85x10^6 

Events per user(Assumption)=40 events/day
Per day usage=2.85x10^6 x40
=114 M/day
Lets Assume 1 req containsMetadata (userId, orgId, timestamp, action, payload)=1KB/event
required daily storage(1copy)=114GB/day
Assume replication factor=3

total storage required=342 GB/day
Monthly req storage=10.2 TB/month
yearly storage req=120.2 Tb/yr
Replicated primary storage ≈ 123 TB
Backup storage              ≈ 123 TB

Total yearly storage Required=256 TB/yr


**Assumption**: Designing a system to track usage of multiple enterprises

**Overall System design:**

The Enterprise Usage Monitoring and Admin Platform supports two types of clients: web users (employees) who generate usage events and admin users who manage policies, limits, and dashboards. Both clients access the system through a web interface, with static assets and cached dashboard responses served via a CDN to reduce latency and backend load. Requests then pass through a Load Balancer, which distributes traffic across multiple backend instances to ensure high availability and horizontal scalability.

Behind the load balancer, multiple stateless API Gateway instances act as a single entry point to the system. The API Gateway handles authentication, authorization, rate limiting, and request routing, keeping security and cross-cutting concerns centralized. Requests are then forwarded to internal microservices. The Identity and Access Service manages user authentication, token validation, roles, and access policies, while the Usage Ingestion Service accepts usage events, performs validation and normalization, applies rate limits, and publishes events to a message queue for asynchronous processing.

The message queue decouples ingestion from downstream processing and helps absorb traffic spikes. A Usage Processing service consumes events, aggregates usage by user, feature, and time window, and checks usage against configured limits. When limits are exceeded, violations are recorded and alerts are triggered. Dashboard and Admin APIs provide read-optimized access to usage analytics, trends, violations, and audit logs using pre-aggregated data and caching. The system uses a relational database for transactional data such as users, policies, limits, violations, and audit logs, along with an analytics store for time-series usage data, enabling a scalable, secure, and enterprise-ready monitoring platform.

**API Gateway**

1.Identity and Access service
    -Auth api
    -admin api
    
    Core logic
    Token manager
    Policy engine
    Role Resolver
    
    POST /v1/auth/login
    GET  /v1/auth/validate
    POST /v1/admin/policies

**Schema design:**
  1.User
    1.user_id           UUID PRIMARY KEY,
    2.enterprise_id     UUID NOT NULL,
    3. email             VARCHAR UNIQUE,
    4. password_hash     VARCHAR,
    5. status            
    6.created_at        TIMESTAMP,
    7..last_login_at     TIMESTAMP

  2.Roles
    1.  role_id       UUID PRIMARY KEY,
    2. enterprise_id UUID,
    3.role_name     VARCHAR,
    4.description   VARCHAR



  usage_alerts
    1.alert_id       UUID PRIMARY KEY,
    2.enterprise_id  UUID,
    3.feature_name   VARCHAR,threshold_pct  INT,
    4.window_type    5.ENUM('HOURLY','DAILY','MONTHLY'),
    6.notification   ENUM('EMAIL')
    7.status         ENUM('ACTIVE','DISABLED'),
    8.created_at     TIMESTAMP



2.Usage Ingestion Service
    1.Usage api
    2.Adimin Usage API
    Validation & Normalization  
    Rate limitor
    
    POST /v1/usage/events
    POST /v1/admin/usage/limits
    PUT /v1/admin/usage/limits/{limitId}
    
    POST /v1/admin/alerts

3.Admin & Dashboard APIs
  
  GET /v1/dashboard/summary
  GET /v1/dashboard/usage/trends
  GET /v1/dashboard/users/{userId}/analytics
  GET /v1/dashboard/usage/by-feature
  GET /v1/dashboard/users/exceeded-limits
  GET /v1/dashboard/violations

  

