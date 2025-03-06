# eBay Online Auction System Design Summary

<img width="1270" alt="Screenshot 2025-03-05 at 5 15 15 PM" src="https://github.com/user-attachments/assets/e6f7cf75-ca0f-4f89-967e-5787c937816d" />


## System Overview
- **Definition**: Online platform where users can buy and sell items through an auction process
- **Scale**: Support for 10M concurrent auctions

## Functional Requirements
- Users can post items for auction with starting price and end date
- Users can bid on items (bids accepted if higher than current highest bid)
- Users can view auctions including current highest bid
- **Out of Scope**: Search, filtering, sorting, auction history

## Non-Functional Requirements
- Strong consistency for bids (all users see same highest bid)
- Fault tolerance and durability (no dropped bids)
- Real-time display of current highest bid
- Scalability to support 10M concurrent auctions
- **Out of Scope**: Observability, security, CI/CD

## Core Entities
- **Auction**: Contains auction details (starting price, end date, current highest bid)
- **Item**: Details about the item being sold (name, description, image)
- **Bid**: Records of bids placed (amount, user, auction, timestamp)
- **User**: Person who posts auctions or places bids

## API Design
- **Create Auction**: `POST /auctions` (item details, startDate, endDate, startingPrice)
- **Place Bid**: `POST /auctions/:auctionId/bids` (bid amount)
- **View Auction**: `GET /auctions/:auctionId` (returns auction with current highest bid)

## High-Level Design Components

### 1. Auction Creation System
- **Client**: Web/mobile interface
- **API Gateway**: Handles routing and authentication
- **Auction Service**: Processes auction creation requests
- **Database**: Stores auction and item data

### 2. Bidding System
- **Bidding Service**: Separate from Auction Service
  - Validates bids (must be > current highest)
  - Updates auction with new highest bid
  - Stores bid history
  - Notifies users of bid updates
- **Message Queue**: Ensures durability of bids
- **Real-time Updates**: Pushes bid updates to clients

## Deep Dive Areas and Tradeoffs

### 1. Bid Consistency Approaches

#### Simple Database Approach
- **Process**: Lock row → read max bid → validate new bid → write bid → update max bid → commit
- **Issues**: Locks entire row, potential for race conditions

#### Caching Approach
- **Process**: Store max bid in cache, update both cache and DB
- **Tradeoffs**:
  - **Pros**: No database locking, faster reads
  - **Cons**: Potential inconsistency between cache and database

#### Single Row Locking Approach
- **Process**: Lock only auction row → validate bid → update → commit
- **Tradeoffs**:
  - **Pros**: Ensures consistency, minimal locking scope
  - **Cons**: Still uses pessimistic locking

#### Optimistic Concurrency Control
- **Process**: Read max bid → validate → conditional update → retry if failed
- **Tradeoffs**:
  - **Pros**: No locking, ideal for low-conflict scenarios
  - **Cons**: Requires retries when conflicts occur
  - **Best suited**: For auction system where simultaneous bids on same item are rare

### 2. Fault Tolerance & Durability

#### Message Queue Approach
- **Implementation**: Use Kafka for bid persistence
- **Process**:
  1. User submits bid
  2. Immediately written to Kafka queue
  3. Acknowledge receipt to user
  4. Bid Service processes from queue at its pace
  5. Write to database if valid

- **Tradeoffs**:
  - **Pros**:
    - Guarantees no bid loss even during service failures
    - Buffers against load spikes
    - Provides guaranteed ordering of bids
  - **Cons**:
    - Adds 2-10ms latency
    - Additional system complexity
    - Potential for delayed processing

### 3. Real-time Highest Bid Updates

#### Polling Approach
- **Process**: Client requests updates every few seconds
- **Tradeoffs**:
  - **Pros**: Simple implementation
  - **Cons**: Too slow for active auctions, inefficient (many unnecessary requests)

#### Server-Sent Events (SSE) Approach
- **Process**: Server maintains connection and pushes updates to client
- **Tradeoffs**:
  - **Pros**: Real-time updates, efficient (push vs. pull)
  - **Cons**: Connection maintenance overhead, requires special handling for load balancers

#### WebSockets Approach
- **Process**: Bidirectional communication channel between client and server
- **Tradeoffs**:
  - **Pros**: Real-time, bidirectional (useful if client needs to send frequent updates)
  - **Cons**: More complex than SSE, higher overhead

### 4. Scaling to 10M Concurrent Auctions

#### Throughput Requirements
- 10M auctions with ~100 bids each = 1B bids per day
- Approximately 10K bids per second

#### Database Scaling
- **Storage Analysis**: 
  - 1KB per auction × 10M auctions × 52 weeks = 520M auctions per year
  - Each bid ~500 bytes × 100 bids per auction
  - Total ~25TB per year

- **Sharding Strategy**:
  - Shard by auctionId
  - Ensures all operations for a single auction stay on same shard
  - No scatter-gather needed for queries

#### Messaging System Scaling
- Kafka can handle 10K messages/second on decent hardware

#### Service Scaling
- Horizontally scale Bid Service and Auction Service
- Auto-scaling based on CPU/memory usage

#### Real-time Updates Scaling (SSE)
- **Challenge**: Many connections spread across multiple servers
- **Solution**: Pub/Sub architecture (Redis Pub/Sub or Kafka)
  - When new bid arrives, publish to relevant auction channel
  - All servers subscribe to channels for auctions their clients are watching
  - Servers push updates to relevant connected clients

## Key Design Decisions & Justifications

### 1. Separate Bidding Service
- **Justification**:
  - Independent scaling (bidding has ~100x more traffic than auction creation)
  - Isolation of complex bidding logic
  - Specialized performance optimization for write-heavy operations

### 2. Full Bid History vs. Just Max Bid
- **Decision**: Store complete bid history, not just current highest
- **Justification**:
  - Critical for audit trail and dispute resolution
  - Enables analysis of bidding patterns
  - Maintains data integrity by not destroying historical information

### 3. Optimistic Concurrency Control
- **Justification**:
  - Better performance than locking for systems with rare conflicts
  - Simplified architecture compared to distributed locks
  - Aligns with auction use case where simultaneous bids are uncommon

### 4. Message Queue + Database Approach
- **Justification**:
  - Balances durability (never lose bids) with performance
  - Handles traffic spikes without dropping bids
  - Provides order guarantees important for fair auction processes
