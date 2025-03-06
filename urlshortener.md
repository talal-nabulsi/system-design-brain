# URL Shortener (Bit.ly) System Design Summary

<img width="1247" alt="Screenshot 2025-03-05 at 4 39 40 PM" src="https://github.com/user-attachments/assets/a0fe5e05-8963-417d-b42f-0998060a238c" />



## System Overview
- **Definition**: Service that converts long URLs into shorter, manageable links
- **Read-to-Write Ratio**: Heavily skewed toward reads (approximately 1000:1)

## Functional Requirements
- Create shortened URLs from long URLs
- Optional custom aliases for shortened URLs
- Optional expiration dates for shortened URLs
- Access original URL by using the shortened version
- **Out of Scope**: User authentication, click analytics

## Non-Functional Requirements
- Uniqueness of short codes (no collisions)
- Low latency redirection (<100ms)
- High reliability (99.99% availability)
- Scalability to 1B shortened URLs and 100M daily active users
- **Out of Scope**: Real-time analytics consistency, advanced security

## Core Entities
- Original URL: The long URL being shortened
- Short URL: The shortened version with unique code
- User: The creator of the shortened URL

## API Design
- **Create Short URL**:
  ```
  POST /urls
  {
    "long_url": "https://www.example.com/some/very/long/url",
    "custom_alias": "optional_custom_alias", 
    "expiration_date": "optional_expiration_date"
  }
  ```
  Response: `{ "short_url": "http://short.ly/abc123" }`

- **Access Original URL**:
  ```
  GET /{short_code}
  ```
  Response: HTTP 302 Redirect to original URL

## High-Level Design Components
- **Client**: Web/mobile app for user interaction
- **Primary Server**: Handles URL validation, short code generation, and redirection
- **Database**: Stores mappings between short codes and long URLs
- **Cache**: Improves read performance for frequent redirects
- **CDN/Edge Computing**: Optional for further latency reduction

## Short Code Generation Approach
- **Counter-Based with Base62 Encoding**:
  - Use an incremental counter (stored in Redis)
  - Encode counter value with base62 (a-z, A-Z, 0-9)
  - Guarantees uniqueness with no collision checking
  - 6 characters can support 1B URLs (scales to 7 chars for trillions)
  
- **Tradeoffs**:
  - **Pros**: Guaranteed uniqueness, computationally efficient, no need for collision checks
  - **Cons**: Single point of failure with centralized counter, synchronization challenges in distributed environments
  - **Why Redis works**: Single-threaded with atomic operations, eliminates race conditions

## Performance Optimization
- **Database Indexing**:
  - Create indexes on short_code column for fast lookups
  - **Tradeoffs**:
    - **Pros**: Dramatically faster reads, eliminates full table scans
    - **Cons**: Slight write performance impact, additional storage overhead
    - Critical given read-heavy workload (prevents O(n) lookup complexity)

- **Caching Strategy**:
  - Cache frequently accessed mappings
  - Implement LRU (Least Recently Used) eviction policy
  - Write-through cache pattern for consistency
  - **Tradeoffs**:
    - **Pros**: Reduces database load, faster responses for popular URLs
    - **Cons**: Increased system complexity, potential consistency issues
    - **Memory constraints**: Must balance cache size with hit ratio

- **CDN & Edge Computing** (optional):
  - Use CDN with global Points of Presence
  - Deploy redirect logic at edge locations
  - Reduces latency by handling redirects closer to users
  
- **CDN Tradeoffs**:
  - **Pros**: Significant latency reduction, offloads traffic from origin servers
  - **Cons**: Cache invalidation complexity, additional configuration overhead
  - **Cost vs. Performance**: Higher costs but better user experience
  - **Debugging challenges**: More difficult in distributed edge environments

## Scaling Approach
- **Data Sizing**: ~500GB for 1B URLs (500 bytes per record)
- **Database Considerations**:
  - Most DB technologies work due to modest write load (~1 write/second)
  - PostgreSQL, MySQL, DynamoDB all viable options
  - Read-heavy workload (1000:1 read-to-write ratio) influences choice

- **High Availability Tradeoffs**:
  - **Database Replication**: Multiple copies on different servers
    - **Pros**: High availability if primary fails
    - **Cons**: Adds complexity, operational overhead
  - **Database Backup**:
    - **Pros**: Recovery option for catastrophic failures
    - **Cons**: Potential data loss between backups

- **Microservice Architecture**:
  - Separate read service (handles redirects)
  - Separate write service (creates short URLs)
  - **Pros**: Independent scaling, specialized optimization
  - **Cons**: Increased system complexity, service coordination challenges

- **Counter Scaling Challenges**:
  - Single Redis instance creates bottleneck/single point of failure
  - **Solution**: Counter batching (request 1000 values at once)
    - **Pros**: Reduces network requests, improves throughput
    - **Cons**: Potentially wasted counter values if service restarts
  - Redis replication for high availability

## Redirection Process
- **Redirect Options**:
  - **301 (Permanent Redirect)**: Indicates resource permanently moved
  - **302 (Temporary Redirect)**: Indicates resource temporarily located elsewhere

- **Tradeoffs**:
  - **301 Pros**: Better for SEO, slightly faster for repeat visits (browser caching)
  - **301 Cons**: Browsers cache the response, bypassing our server on subsequent visits
  - **302 Pros**: Greater control, prevents caching, enables analytics, allows updating links
  - **302 Cons**: Slightly higher load on servers (all requests go through our system)
  
- **Recommendation**: Use 302 redirects for most URL shortener use cases

## Final Architecture Highlights
- Split read/write paths for optimal scaling
- Horizontally scaled services behind load balancers
- Redis for distributed counter with batching
- Caching layer for frequent URL lookups
- Optional CDN integration for edge redirection
