# System Design: URL Shortener (TinyURL)

## 1. Problem Statement and Requirements

Let's start by defining what we need to build.

### Functional Requirements
- Generate a short URL from a given long URL
- Redirect users to the original URL when they access the short URL
- Optional custom short URLs
- URL expiration (optional)

### Non-Functional Requirements
- High availability
- Low latency for redirections
- URL shortening should be real-time
- Short URLs should be unique and hard to predict

```mermaid
graph TD
    Client[Client]
    LB[Load Balancer]
    WebServ[Web Servers]
    Cache[Cache Layer]
    DB[(Database)]
    HashGen[Hash Generator]

    Client -->|1. HTTP Request| LB
    LB -->|2. Route Request| WebServ
    WebServ -->|3. Check Cache| Cache
    Cache -->|4. Cache Miss| WebServ
    WebServ -->|5. Query| DB
    WebServ -->|6. Generate Short URL| HashGen
    HashGen -->|7. Return Hash| WebServ
    WebServ -->|8. Store| DB
    WebServ -->|9. Update| Cache
    WebServ -->|10. Return Response| Client
```

## 2. Back-of-the-envelope Calculations

### Traffic Estimates
- Assuming 100 million URLs generated per month
- Read:Write ratio = 100:1
- QPS (Queries Per Second):
  - URL Creation: ~40 URLs/sec
  - URL Redirection: ~4,000 URLs/sec

### Storage Estimates
- Long URL average size: 100 bytes
- Short URL size: 7 bytes
- Metadata (timestamp, user_id, etc.): 50 bytes
- Total size per entry: ~160 bytes
- Storage needed for 5 years: 100M * 12 * 5 * 160 bytes ≈ 960GB

```mermaid
sequenceDiagram
    participant C as Client
    participant A as API Server
    participant H as Hash Generator
    participant D as Database
    participant R as Cache

    C->>A: POST /shorten {long_url}
    A->>H: Generate Hash
    H->>A: Return Short URL Hash
    A->>D: Store Mapping
    A->>R: Cache Mapping
    A->>C: Return Short URL
```

## 3. API Design

### REST API Endpoints

```plaintext
1. Create URL
POST /api/v1/shorten
Request: {"longUrl": "https://example.com", "customAlias": "myurl"}
Response: {"shortUrl": "http://tiny.url/abc123"}

2. Redirect URL
GET /:shortUrl
Response: HTTP 301 Redirect

3. URL Statistics (Optional)
GET /api/v1/stats/:shortUrl
Response: {"clicks": 100, "created": "2024-01-01"}
```

## 4. Database Design

```mermaid
erDiagram
    URLs {
        string short_url_hash PK
        string long_url
        timestamp created_at
        timestamp expires_at
        string user_id FK
        int clicks
    }
    Users {
        string user_id PK
        string email
        timestamp created_at
    }
    URLs }|--|| Users : belongs_to
```

## 5. URL Shortening Algorithm

```mermaid
graph LR
    A[Long URL] -->|1. Generate| B[MD5 Hash]
    B -->|2. Take First 7 Bytes| C[Short Hash]
    C -->|3. Base62 Encode| D[Short URL]
    D -->|4. Check Collision| E{Exists?}
    E -->|Yes| B
    E -->|No| F[Store URL]
```

## 6. Detailed Component Design

### Cache Architecture
- Use Redis for caching
- Cache the most frequently accessed URLs
- LRU eviction policy
- Cache size: 20% of daily active URLs

```mermaid
graph TD
    A[Request] -->|1. Check Cache| B{Cache Hit?}
    B -->|Yes| C[Return URL]
    B -->|No| D[Query DB]
    D -->|Found| E[Update Cache]
    E --> C
    D -->|Not Found| F[Return 404]
```

## 7. Scale and Optimization

### Data Partitioning

```mermaid
graph TD
    URL[URL Request] -->|Hash Function| Router[Hash Router]
    Router -->|Hash Range 1| DB1[(Database 1)]
    Router -->|Hash Range 2| DB2[(Database 2)]
    Router -->|Hash Range 3| DB3[(Database 3)]
    
    subgraph Consistent Hashing
    DB1
    DB2
    DB3
    end
```

## 8. Security and Rate Limiting

To prevent abuse, implement:
1. Rate limiting per IP/user
2. URL validation
3. Length restrictions
4. Blacklist functionality

```mermaid
graph TD
    A[Request] -->|1. Check| B{Rate Limit}
    B -->|Exceeded| C[429 Too Many Requests]
    B -->|OK| D{URL Validation}
    D -->|Invalid| E[400 Bad Request]
    D -->|Valid| F{Length Check}
    F -->|Too Long| G[400 Bad Request]
    F -->|OK| H{Blacklist Check}
    H -->|Blacklisted| I[403 Forbidden]
    H -->|OK| J[Process Request]
```

## 9. Monitoring and Analytics

Key metrics to monitor:
- Redirection latency
- Cache hit ratio
- Error rates
- Storage usage
- QPS by endpoint

## 10. Alternative Approaches

1. **UUID-based**: Generate UUID instead of hash
2. **Counter-based**: Use an auto-incrementing counter
3. **Base62 of timestamp + random string**

Each approach has its trade-offs in terms of:
- URL length
- Collision probability
- Predictability
- Database load
