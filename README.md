1. Product Overview
LinkForge is a highly scalable URL-shortening service that converts long URLs into short, unique links.

For example:
- Original URL: https://example.com/products/category/electronics/smartphones
- Short URL: https://lnk.to/aB92xK
When a user visits the short URL, the system redirects them to the original destination.

The platform will also provide link management, analytics, custom aliases, expiration settings, QR codes, and API access.
2. Problem Statement
Long URLs are difficult to share, remember, and manage. Businesses and individuals need a reliable platform that can:
1. Generate short and unique URLs.
2. Redirect users with minimal latency.
3. Handle millions of requests concurrently.
4. Provide real-time or near-real-time analytics.
5. Prevent malicious URL abuse.
6. Maintain high availability even during traffic spikes.
3. Product Goals
Goal	Target
Availability	99.99%
Redirect latency	p95 under 50 ms
Link creation latency	p95 under 200 ms
Scalability	100M+ active short links
Redirect capacity	100K requests/second target
Data durability	99.999999999% design objective
Analytics	Near real-time, under 60 seconds
Security	Abuse detection and rate limiting

These are proposed engineering targets, not measured performance claims.
4. Target Users
- Individual users: Shorten links for sharing.
- Marketing teams: Track campaign performance.
- Businesses: Manage branded short links.
- Developers: Integrate URL shortening through APIs.
- Enterprise organizations: Manage links, teams, permissions, and analytics.
5. Core Features and Requirements
5.1 URL Shortening







linkedin.com


Functional requirements:
- Accept long URLs.
- Validate URL format and allowed schemes.
- Generate a unique short code.
- Store original URL and short code.
- Return the shortened URL.
- Allow users to copy the generated link.
Example API:

POST /api/v1/urls

Request:
{
  "long_url": "https://example.com/products",
  "custom_alias": null,
  "expires_at": null
}


Response:
{
  "short_code": "aB92xK",
  "short_url": "https://lnk.to/aB92xK",
  "created_at": "2026-11-03T10:00:00Z"
}


5.2 URL Redirection







medium.com


When a visitor accesses a shortened URL:
1. Extract the short code.
2. Check Redis cache.
3. If found, retrieve the original URL.
4. If not found, query the database.
5. Update the cache.
6. Record the click asynchronously.
7. Return HTTP 302 redirect.
Example:

GET /aB92xK

Response:
HTTP/1.1 302 Found
Location: https://example.com/products


Use 302 as the default because destinations may change and analytics should capture repeated visits. Support 301 only as an explicit configuration.
5.3 Custom Aliases
Users can create memorable links.

Example:

https://lnk.to/summer-sale

Requirements:
- Check alias availability.
- Prevent duplicate aliases.
- Reserve protected system keywords.
- Apply ownership and authorization checks.
- Allow authorized users to edit or delete aliases.
5.4 Link Expiration
Users can configure:
- No expiration.
- Expiration after a specified date.
- Expiration after a specified duration.
Expired links must return an appropriate error page or HTTP 410 Gone.
5.5 Analytics Dashboard







buffer.com


Track:
Metric	Description
Total clicks	Number of redirect requests
Unique visitors	Estimated distinct visitors
Country	Visitor country based on IP
Device	Mobile, desktop, tablet
Browser	Chrome, Firefox, Safari
Referrer	Traffic source
Click timeline	Hourly/daily click activity

Analytics must be processed asynchronously to avoid slowing down redirects.

Note: IP addresses and visitor identifiers must be handled according to applicable privacy laws.
6. System Architecture







medium.com


High-level architecture:
                  USERS
                    |
                    v
              Load Balancer
                    |
                    v
                API Gateway
                    |
          +---------+---------+
          |                   |
          v                   v
    URL Creation         Redirect Service
          |                   |
          v                   v
      URL Service         Redis Cache
          |                   |
          v                   v
      Database <--------- Cache Miss
                              |
                              v
                           Database
                              |
                              v
                         Redirect 302

          Analytics Event Pipeline
                    |
                    v
                  Kafka
                    |
                    v
             Analytics Workers
                    |
                    v
            Analytics Database
                    |
                    v
             Analytics Dashboard


7. Recommended Technology Stack
Layer	Technology
Frontend	React + TypeScript
Backend	Python FastAPI
Database	PostgreSQL
Cache	Redis
Message Queue	Apache Kafka
Analytics	ClickHouse
Authentication	JWT + OAuth 2.0
Containerization	Docker
Orchestration	Kubernetes
Load Balancing	NGINX
Monitoring	Prometheus + Grafana
Cloud	AWS
CI/CD	GitHub Actions

Why this stack?
- FastAPI supports asynchronous APIs.
- PostgreSQL provides reliable transactional storage.
- Redis provides low-latency lookups.
- Kafka decouples analytics from redirect requests.
- ClickHouse supports large-scale analytical queries.
- Kubernetes supports horizontal scaling and automated recovery.
8. Database Design
URLs Table
Column	Type	Description
id	UUID	Primary key
short_code	VARCHAR(12)	Unique short code
original_url	TEXT	Destination URL
user_id	UUID	Owner
created_at	TIMESTAMP	Creation time
expires_at	TIMESTAMP	Expiration
is_active	BOOLEAN	Active status

Users Table
Column	Type
id	UUID
name	VARCHAR
email	VARCHAR
password_hash	TEXT
created_at	TIMESTAMP

Click Events Table
Column	Type
event_id	UUID
short_code	VARCHAR
timestamp	TIMESTAMP
country	VARCHAR
device	VARCHAR
browser	VARCHAR
referrer	TEXT

For production, keep high-volume click events in a dedicated analytical store rather than allowing the transactional URL database to become overloaded.
9. Short Code Generation Strategy
The system needs unique short codes.

Recommended approach: Base62 encoding of unique IDs.

Base62 uses:
- A–Z (26 characters)
- a–z (26 characters)
- 0–9 (10 characters)
Total: 62 characters.

Example:
Database ID: 125678
Base62:      W7e
Short URL:   lnk.to/W7e


For a production system:
1. Allocate unique IDs using a distributed ID generator or database sequence.
2. Encode the ID using Base62.
3. Enforce a unique database constraint.
4. Cache the short-code mapping.
An alternative is randomly generated codes with collision detection and retry.
10. Scalability and Performance
Challenge	Solution
High redirect traffic	Redis caching
Database overload	Read replicas
Traffic spikes	Horizontal scaling
Analytics bottleneck	Kafka event pipeline
Geographic latency	CDN and regional deployments
Database growth	Partitioning and archival
Hot URLs	Cache replication and TTL
Service failure	Health checks and failover

Important: Redirects are expected to be significantly more frequent than URL creation. Therefore, the architecture should optimize the read path.
11. Security Requirements
Security is a critical part of the platform.
- HTTPS everywhere.
- JWT-based authentication.
- Password hashing using Argon2id or bcrypt.
- Rate limiting per user and IP.
- SQL injection prevention.
- XSS protection.
- Malicious URL detection.
- Phishing and malware domain checks.
- Admin moderation tools.
- Audit logs.
- API key management.
- SSRF prevention.
Additionally, validate destination URLs carefully and block internal/private network destinations where appropriate. Avoid fetching arbitrary URLs from the backend.
12. API Requirements
Method	Endpoint	Purpose
POST	/api/v1/urls	Create short URL
GET	/{short_code}	Redirect
GET	/api/v1/urls	List user's URLs
GET	/api/v1/urls/{id}	Get URL details
PATCH	/api/v1/urls/{id}	Update URL
DELETE	/api/v1/urls/{id}	Delete URL
GET	/api/v1/urls/{id}/analytics	Retrieve analytics
POST	/api/v1/auth/register	Register
POST	/api/v1/auth/login	Login

13. Non-Functional Requirements
1. Availability: 99.99% uptime target.
2. Performance: Redirect p95 latency below 50 ms under defined benchmark conditions.
3. Reliability: No duplicate short codes.
4. Scalability: Support horizontal expansion.
5. Security: Protect user accounts and URLs.
6. Maintainability: Modular architecture.
7. Observability: Centralized logs, metrics, and tracing.
8. Disaster recovery: Database backups and tested restoration procedures.
14. Development Roadmap
Phase	Deliverables	Timeline
Phase 1	Requirements and architecture	Week 1
Phase 2	URL creation and redirection	Week 2
Phase 3	Authentication and database	Week 3
Phase 4	Redis caching	Week 4
Phase 5	Analytics and Kafka	Week 5
Phase 6	Dashboard and custom aliases	Week 6
Phase 7	Security and load testing	Week 7
Phase 8	Deployment and monitoring	Week 8

15. Success Metrics (KPIs)
- Number of URLs successfully created.
- Number of redirects per second.
- Average and p95 redirect latency.
- Cache hit ratio.
- System availability.
- Failed redirect percentage.
- Monthly active users.
- API error rate.
- Analytics processing delay.
- Malicious link detection rate.
16. Out of Scope for Version 1
- Paid subscription billing.
- Enterprise SSO.
- Affiliate marketing.
- Advanced marketing automation.
- AI-generated marketing campaigns.
- Native Android and iOS applications.
17. Acceptance Criteria
The product is ready for its first production release when:
- Users can register and authenticate.
- Users can create unique short URLs.
- Short URLs redirect correctly.
- Redis caching works.
- Expired and disabled URLs are handled correctly.
- Analytics events are processed asynchronously.
- Custom aliases are validated.
- Rate limiting and security controls are implemented.
- Load tests meet the agreed performance targets.
- Monitoring, alerting, backups, and recovery procedures are operational.

18. Final Product Vision







zij.it


LinkForge aims to become a reliable, developer-friendly URL management platform capable of supporting high-volume traffic.

The system should prioritize:
1. Fast redirection.
2. High availability.
3. Data consistency.
4. Secure link management.
5. Scalable analytics.
6. Developer-friendly APIs.