# Bajaj_Life_Assignment
**Develop a “Enterprise Usage Monitoring &amp; Admin Platform”**

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

**1.Identity and Access service**
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
    
 3. Permissions
    1.permission_id UUID PRIMARY KEY,
    2. permission    VARCHAR

4. Role
   role_id        UUID,PRIMARY KEY 
   permission_id  UUID,PRIMARY KEY 

**2.Usage Ingestion Service**
Api design:
    1.Usage api
    2.Adimin Usage API
    Validation & Normalization  
    Rate limitor
    
    POST /v1/usage/events
    POST /v1/admin/usage/limits
    PUT /v1/admin/usage/limits/{limitId}
    
    POST /v1/admin/alerts

1. usage_alerts
    1.alert_id       UUID PRIMARY KEY,
    2.enterprise_id  UUID,
    3.feature_name   VARCHAR,threshold_pct  INT,
    4.window_type    5.ENUM('HOURLY','DAILY','MONTHLY'),
    6.notification   ENUM('EMAIL')
    7.status         ENUM('ACTIVE','DISABLED'),
    8.created_at     TIMESTAMP
    
2. Usage Event
    1.event_id        UUID (PK),
    2. enterprise_id   UUID,
    3.user_id         UUID,
    4.feature_name    VARCHAR,
    5.event_type      VARCHAR,
    6.event_timestamp TIMESTAMP,
    7.received_at     TIMESTAMP

3.Usage Limit
    1.limit_id       UUID PRIMARY KEY,
    2.enterprise_id  UUID NOT NULL,
    3.  feature_name   VARCHAR NOT NULL,
    4.window_type    ENUM('HOURLY','DAILY','MONTHLY'),
    5.action         ENUM('ALERT','BLOCK'),
    6.status         ENUM('ACTIVE','INACTIVE'),
    7.created_at     TIMESTAMP,
    8.updated_at     TIMESTAMP
    
**3.Admin & Dashboard APIs**
  
  GET /v1/dashboard/summary
  GET /v1/dashboard/usage/trends
  GET /v1/dashboard/users/{userId}/analytics
  GET /v1/dashboard/usage/by-feature
  GET /v1/dashboard/users/exceeded-limits
  GET /v1/dashboard/violations
  
**Schema design**
1.usage_aggregates
  1.enterprise_id   UUID,
  2.metric_type     VARCHAR,   -- 3.active_users, usage_count
  4.feature_name    VARCHAR,
  5.time_bucket     TIMESTAMP,
  6.value           BIGINT

**alert_events**
  1.enterprise_id UUID,
  2.feature_name  VARCHAR,
  3.triggered_at  TIMESTAMP,
  4.severity      ENUM('LOW','HIGH')
  
**user_usage_aggregates **
  enterprise_id   UUID,
  user_id         UUID,
  feature_name    VARCHAR,
  time_bucket     DATE,       -- day/hour
  usage_count     BIGINT,
  PRIMARY KEY (enterprise_id, user_id, feature_name, time_bucket)

**usage_violations **
  violation_id   UUID PRIMARY KEY,
  enterprise_id  UUID,
  user_id        UUID,
  feature_name   VARCHAR,
  limit_value    BIGINT,
  actual_value  BIGINT,
  window_type    ENUM('DAILY','MONTHLY'),
  detected_at    TIMESTAMP,
  status         ENUM('OPEN','RESOLVED')

**Database**

The system uses PostgreSQL as the primary database for all transactional and critical data such as users, enterprises, roles, access policies, usage limits, usage violations, and audit logs. PostgreSQL is chosen because it provides strong ACID guarantees, supports complex relationships and queries, and ensures data consistency, which is essential for enterprise-grade security, policy enforcement, and auditing.

Redis is used as an in-memory cache and fast-access store for frequently accessed and time-sensitive data such as authentication tokens, role and policy lookups, rate-limiting counters, and cached dashboard summaries. Using Redis reduces load on PostgreSQL, improves API response times, and supports low-latency operations required for authentication, rate limiting, and real-time dashboards.

**Read request flow**

When a client requests data (for example, dashboard metrics or policy details), the system first checks Redis cache for the requested information using a well-defined cache key (such as enterpriseId + feature + window). If the data is found in Redis (cache hit), it is returned immediately to the client, ensuring low latency and reduced backend load.

If the data is not present in Redis (cache miss), the request is forwarded to PostgreSQL, where the data is fetched from the persistent store. Once retrieved, the data is returned to the client and simultaneously written back to Redis with an appropriate TTL. This ensures that subsequent reads are served from cache. This cache-first read flow improves performance, reduces database load, and supports scalable read-heavy dashboard traffic.

**Write Request Flow:**
For write requests (such as updating usage limits, policies, or recording violations), the system first writes the data to PostgreSQL as the source of truth to ensure durability and consistency. After a successful database write, the system either updates the corresponding cache entry or invalidates it in Redis. This guarantees that subsequent read requests fetch fresh data while maintaining high performance and data consistency.
