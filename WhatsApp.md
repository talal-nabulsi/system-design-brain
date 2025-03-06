# WhatsApp/Messenger System Design Summary

<img width="1254" alt="Screenshot 2025-03-05 at 5 45 56 PM" src="https://github.com/user-attachments/assets/73e67e73-3d56-44a9-87b6-503898da296d" />



## System Overview
- **Definition**: Messaging service allowing users to send/receive messages and media
- **Scale**: Billions of users with high throughput
- **Original Note**: WhatsApp was initially built on Erlang, achieving high scale with limited infrastructure

## Functional Requirements
- Users can start group chats with multiple participants (limit 100)
- Users can send/receive messages
- Users can receive messages sent while offline (up to 30 days)
- Users can send/receive media in messages
- **Out of Scope**: Audio/video calls, business interactions, registration/profiles

## Non-Functional Requirements
- Low latency message delivery (<500ms)
- Guaranteed message deliverability
- Scale to billions of users with high throughput
- Minimal storage of messages on centralized servers
- Resilience against component failures
- **Out of Scope**: Security, spam prevention

## Core Entities
- **User**: Person using the messaging service
- **Chat**: Conversation between 2-100 users
- **Message**: Text or media content sent in a chat
- **Client**: Device used to access the service (multiple per user)

## API Design via WebSocket Commands

### Client to Server
- **createChat**: Create a new chat with participants
- **sendMessage**: Send a message to a chat
- **createAttachment**: Create a media attachment
- **modifyChatParticipants**: Add/remove users to a chat

### Server to Client
- **chatUpdate**: Notify about chat creation/updates
- **newMessage**: Deliver a new message
- **Acknowledgements**: All commands require receipt confirmation

## High-Level Design Components

### 1. Chat Creation System
- **Chat Server**: Handles WebSocket connections and chat creation
- **Database**: Stores chat metadata
  - **Chat Table**: Chat details with primary key on chatId
  - **ChatParticipant Table**: Maps chats to users with compound key (chatId, participantId)
  - **GSI**: Enables querying all chats for a user (participantId as partition key)

### 2. Messaging System
- **Chat Server**: Manages WebSocket connections in a user-to-connection map
- **Message Table**: Stores all messages
- **Inbox Table**: Tracks undelivered messages per user
- **Cleanup Process**: Removes messages older than 30 days

### 3. Media Handling System
- **Blob Storage**: Stores media files (images, videos, etc.)
- **Pre-signed URL System**: Enables direct upload/download to storage

## Message Flow

### Sending Messages
1. User sends message via WebSocket to Chat Server
2. Chat Server identifies all chat participants
3. Chat Server creates transaction to:
   - Write message to Message table
   - Create entry in Inbox table for each recipient
4. Server sends SUCCESS/FAILURE to sender
5. Server attempts immediate delivery to online users
6. Recipients acknowledge receipt, allowing removal from Inbox

### Offline Message Delivery
1. User connects to Chat Server
2. Server looks up user's Inbox for undelivered messages
3. Server retrieves messages from Message table
4. Server delivers messages via WebSocket
5. Client acknowledges receipt
6. Server removes delivered messages from Inbox

### Media Sharing
1. Sender requests pre-signed URL for upload
2. Sender uploads media directly to blob storage
3. Sender includes media URL in message
4. Recipients request pre-signed URL for download
5. Recipients download media directly from blob storage

## Deep Dive Areas & Tradeoffs

### 1. Scaling to Billions of Users

#### Single-Host Limitations
- A single Chat Server cannot handle billions of connections
- Estimate: 200M concurrent users (of 1B total)
- Historical reference: WhatsApp served ~1-2M users per host

#### Multiple Chat Server Approach
- **Challenge**: Users in the same chat may connect to different servers
- **Solutions**:

##### Redis Pub/Sub (Recommended)
- **Process**:
  - Chat Servers subscribe to topics for their connected users
  - Messages published to user topics
  - Messages routed to appropriate servers
- **Tradeoffs**:
  - **Pros**: Lightweight, scalable, purpose-built for this use case
  - **Cons**: "At most once" delivery, additional latency (single-digit ms)
  - **Complexity**: All-to-all connections between Chat Servers and Redis nodes

##### Alternative: Direct Server-to-Server Communication
- **Tradeoffs**:
  - **Pros**: No additional infrastructure needed
  - **Cons**: N² connection complexity, difficult to maintain at scale

##### Alternative: Consistent Hashing
- **Tradeoffs**:
  - **Pros**: Predictable routing
  - **Cons**: Forces clients to reconnect during scaling events

### 2. Multi-Device Support

#### Challenges
- Users access from multiple devices (phone, tablet, laptop)
- Each device needs separate message tracking
- Devices can be offline independently

#### Solution
- **Clients Table**: Maps users to multiple client devices
- **Per-Client Inbox**: Track message delivery per device vs. per user
- **Topic Subscription**: Subscribe to client-level topics vs. user-level
- **Device Limits**: Cap at reasonable number (e.g., 3) to control resource usage

#### Message Flow with Multiple Devices
1. Lookup all clients for each chat participant
2. Create Inbox entries for each client
3. Deliver to all connected clients
4. Track acknowledgements per client

## Key Design Decisions & Justifications

### 1. WebSockets vs. HTTP
- **Decision**: WebSockets for real-time communication
- **Justification**: Enables bi-directional, low-latency communication essential for messaging

### 2. Message Persistence Strategy
- **Decision**: Store messages until confirmed delivered
- **Justification**: Ensures delivery regardless of recipient connectivity status

### 3. Media Handling Approach
- **Decision**: Pre-signed URLs for direct upload/download
- **Justification**: Reduces server load, improves performance, leverages specialized storage

### 4. Redis Pub/Sub for Inter-Server Communication
- **Decision**: Use dedicated message broker vs. direct communication
- **Justification**: Scales better than alternatives, specialized for the purpose
