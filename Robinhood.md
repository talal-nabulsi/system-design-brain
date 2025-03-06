# Robinhood Stock Trading App System Design Summary

<img width="1279" alt="Screenshot 2025-03-05 at 4 52 32 PM" src="https://github.com/user-attachments/assets/f059d17c-a547-4f82-b69a-815901620aa5" />

## System Overview
- **Definition**: Commission-free trading platform for stocks that routes trades through market makers
- **Role**: Robinhood acts as a broker, not an exchange (interfaces with external exchanges)
- **Scale**: 20M daily active users, 5 trades per user per day, 1000s of symbols

## Key Financial Concepts
- **Symbol/Ticker**: Unique stock identifier (e.g., META, AAPL)
- **Market Order**: Immediate buy/sell at current market price
- **Limit Order**: Buy/sell at specified price, waits to be filled or canceled
- **Exchange Interface**: External entity providing order processing and trade feed data

## Functional Requirements
- Users can see live prices of stocks
- Users can manage orders (create/cancel market and limit orders)
- **Out of Scope**: After-hours trading, ETFs/options/crypto, order book viewing

## Non-Functional Requirements
- High consistency for order management
- Scalability for high trade volume
- Low latency for price updates and order placement (<200ms)
- Minimize connections to external exchange APIs (expensive)
- **Out of Scope**: Multiple exchange connections, trading fees, daily limits, bot protection

## Core Entities
- **User**: Platform user
- **Symbol**: Stock being traded
- **Order**: Buy/sell request created by a user

## API Design
- **Get Symbol**: `GET /symbol/:name`
- **Create Order**: `POST /order` with position, symbol, priceInCents, numShares
- **Cancel Order**: `DELETE /order/:id`
- **List Orders**: `GET /orders` (paginated)
- User authentication via headers (session token/JWT)

## High-Level Design Components

### 1. Live Price Updates System
- **Trade/Symbol Processor**: Receives price updates from exchange
- **Symbol Service**: Manages user subscriptions and pushes updates
- **Redis Pub/Sub**: Routes symbol updates to appropriate services
- **Server-Sent Events (SSE)**: Pushes updates to clients

### 2. Order Management System
- **Order Service**: Handles order creation/cancellation requests
- **Order Dispatch Gateway**: Manages external exchange communication
- **Orders Database**: Stores order information (partitioned by userId)
- **Order-Mapping KV Store**: Maps externalOrderIds to internal orders
- **Trade Processor**: Updates order status based on exchange feedback
- **Cleanup Process**: Ensures consistency for failed operations

## Live Price Updates Approach

### Server-Sent Events (SSE) Approach
- **Process**:
  1. Client sends POST to `/subscribe` with symbols list
  2. Backend establishes SSE connection
  3. Initial prices sent from cache
  4. Real-time updates pushed as they occur

- **Tradeoffs**:
  - **Pros**: Unidirectional (simpler than WebSockets), works over HTTP, efficient for one-way data
  - **Cons**: Requires load balancer configuration for sticky sessions, reconnection handling

### Scaling Live Price Updates
- **Redis Pub/Sub Architecture**:
  - Symbol service tracks Symbol → Set<userId> mapping
  - Each server subscribes to Redis channels for relevant symbols
  - Updates published to Redis by processor
  - Servers fan out updates to relevant users
  - Self-regulating subscription management

- **Workflow**:
  1. User subscribes via symbol service
  2. Service subscribes to Redis channels as needed
  3. Price updates from exchange → processor → Redis → service → user
  4. Unsubscriptions/disconnects trigger channel cleanup

## Order Management Approach

### Order Dispatch System
- **NAT Gateway Approach**:
  - Orders sent through gateway (e.g., AWS NAT)
  - Appears as small set of IPs to exchange
  - Order service manages business logic and scaling

- **Tradeoffs**:
  - **Pros**: Controls number of connections to exchange, simplifies IP management
  - **Cons**: Service must handle both client and exchange interactions efficiently
  - **Scaling Considerations**: Sensitive auto-scaling or over-provisioning for trading spikes

### Order Storage and Tracking
- **Data Structure**:
  - Relational database (PostgreSQL) partitioned by userId
  - Key-value store mapping externalOrderId → (orderId, userId)
  - Order states: pending → submitted → filled/canceled

- **Trade Processor Role**:
  1. Receives updates from exchange feed
  2. Uses KV store to map trades to internal orders
  3. Updates order status in database

### Order Consistency Management
- **Order Creation Flow**:
  1. Store order as "pending" in database
  2. Submit to exchange (synchronous)
  3. Record externalOrderId and update status to "submitted"
  4. Respond to client

- **Order Cancellation Flow**:
  1. Update order status to "pending_cancel"
  2. Submit cancellation to exchange
  3. Update status to "cancelled" in database
  4. Respond to client

- **Failure Handling Strategies**:
  - **Database Failures**: Return error to client
  - **Exchange Communication Failures**: Mark order as failed
  - **Post-Submission Failures**: Cleanup job resolves pending orders by checking with exchange
  - **Cancellation Failures**: Cleanup job ensures pending_cancel orders are properly handled

### Key Tradeoffs in Order Management
- **Immediate DB Write vs. Exchange First**:
  - Writing to DB first ensures record exists even if system fails
  - Increases overall latency but improves consistency

- **Synchronous vs. Asynchronous Exchange Communication**:
  - Synchronous ensures immediate confirmation
  - Asynchronous might be faster but increases complexity in tracking state

- **Cleanup Process Overhead vs. Consistency**:
  - Background process adds system complexity
  - Essential for maintaining consistency in failure scenarios

## Additional Design Considerations
- **Load Balancing**: Sticky sessions for SSE connections
- **Caching**: Price cache updated by processor for initial loads
- **Partitioning Strategy**: Orders partitioned by userId for efficient queries
- **Excess Price Updates**: May need throttling mechanisms
- **Order Batching**: Could improve exchange communication efficiency
- **Live Order Updates**: Could use similar SSE mechanism as price updates
