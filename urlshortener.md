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

## Performance Optimization
- **Database Indexing**:
  - Create indexes on short_code column for fast lookups
  - Removes need for full table scans

- **Caching Strategy**:
  - Cache frequently accessed mappings
  - Implement LRU (Least Recently Used) eviction policy
  - Write-through cache pattern for consistency

- **CDN & Edge Computing** (optional):
  - Use CDN with global Points of Presence
  - Deploy redirect logic at edge locations
  - Reduces latency by handling redirects closer to users

## Scaling Approach
- **Data Sizing**: ~500GB for 1B URLs (500 bytes per record)
- **Database Options**: PostgreSQL, MySQL, DynamoDB (most options work)
- **High Availability**:
  - Database replication for redundancy
  - Periodic database backups

- **Microservice Architecture**:
  - Separate read service (handles redirects)
  - Separate write service (creates short URLs)
  - Scale each service independently based on demand

- **Counter Scaling**:
  - Centralized Redis instance for counter
  - Counter batching (each service requests 1000 values at once)
  - Redis replication for counter high availability

## Redirection Process
- Prefer 302 (Temporary Redirect) over 301 (Permanent Redirect)
- Advantages of 302:
  - Greater control over redirection
  - Prevents browser caching
  - Enables click tracking if needed
  - Allows updating/expiring links

## Final Architecture Highlights
- Split read/write paths for optimal scaling
- Horizontally scaled services behind load balancers
- Redis for distributed counter with batching
- Caching layer for frequent URL lookups
- Optional CDN integration for edge redirection
