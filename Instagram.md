# **Instagram System Design Breakdown**


<img width="1296" alt="Screenshot 2025-03-05 at 4 26 46 PM" src="https://github.com/user-attachments/assets/f06caf90-c825-46e9-aade-07eaf77db377" />


# Instagram System Design Summary

## System Overview
- **Definition**: Visual content-focused social media platform for sharing photos and videos
- **Scale**: 500M daily active users (DAU), 100M posts per day

## Functional Requirements
- Users can create posts (photos/videos with captions)
- Users can follow other users
- Users can view chronological feed of posts from followed users
- **Out of Scope**: Likes, comments, search, stories, live streaming

## Non-Functional Requirements
- High availability (prioritize availability over consistency; 2-min eventual consistency acceptable)
- Low latency feed delivery (<500ms end-to-end)
- Instant photo/video rendering
- Scalable to support 500M DAU
- **Out of Scope**: Security, fault tolerance, analytics

## Core Entities
- **User**: Profile information, authentication details
- **Post**: Media reference, caption, creator reference
- **Media**: Actual media bytes (stored in S3)
- **Follow**: Relationship between follower and followee (unidirectional)

## API Design
- `POST /posts`: Create new post (returns postId)
- `POST /follows`: Follow another user
- `GET /feed?cursor={cursor}&limit={page_size}`: Get paginated feed

## High-Level Architecture

### Post Creation Flow
1. Client uploads media + caption via API Gateway
2. Post Service stores metadata in Posts DB
3. Media bytes stored in Blob Store (S3)
4. Return postId to client

### Follow User Flow
1. Client sends follow request to Follow Service
2. Follow Service adds entry to Followers table (followerId, followedId)

### Feed Generation
1. Client requests feed with pagination parameters
2. Post Service retrieves posts from followed users
3. Posts are sorted chronologically and returned to client

## Deep Dive: Efficient Feed Generation

### Problem with Simple Approach
- Read amplification (1000+ followed accounts = 1000+ queries)
- Repeated work (popular posts retrieved millions of times)
- Unpredictable performance based on follow count

### Solution: Hybrid Feed Generation
- **Fan-out on Write (for regular users)**:
  - When user posts, update precomputed feeds for followers
  - Store in Redis for quick retrieval
  - Works well for users with <100K followers

- **Fan-out on Read (for "celebrity" accounts)**:
  - Don't precompute for users with >100K followers
  - Fetch celebrity posts at read time
  - Merge with precomputed feed

- **Benefits**: Balances write amplification with read performance

## Deep Dive: Media Handling

### Upload Optimization
- Use S3 multipart upload for large files
- Process:
  1. Create post metadata with "pending" status
  2. Get pre-signed S3 URL for direct upload
  3. Client chunks and uploads directly to S3
  4. Update status to "complete" (server-driven via S3 event)

### Media Delivery Optimization
- Use CDN with dynamic media optimization
- Generate multiple variants for different devices/networks
- Adaptive streaming for videos
- Intelligent caching strategies
  - More aggressive caching for popular content
  - Shorter TTLs for less accessed content

## Scaling Considerations
- **Storage Requirements**:
  - Media: ~200TB new data per day (~750PB over 10 years)
  - Metadata: ~100GB new data per day
  - Strategy: Move cold data to cheaper storage tiers (S3 → Glacier)

- **Database Scaling**:
  - DynamoDB with careful partition/sort key design
  - Follow table: partition key = followerId, sort key = followedId
  - Post table: partition key = userId, sort key = postId (chronological)

- **Service Scaling**:
  - Horizontal scaling of microservices
  - Load balancers for traffic distribution
  - Auto-scaling based on CPU/memory thresholds
