# Google Docs Collaborative Document Editor System Design

<img width="722" alt="Screenshot 2025-03-05 at 5 19 03 PM" src="https://github.com/user-attachments/assets/e2d819fa-8602-40de-94af-e64c0b7572a6" />


## System Overview
- **Definition**: Browser-based collaborative document editor allowing real-time editing
- **Scale**: Millions of concurrent users across billions of documents
- **Concurrency**: Up to 100 editors per document

## Functional Requirements
- Users can create new documents
- Multiple users can edit the same document concurrently
- Users can view each other's changes in real-time
- Users can see cursor position and presence of other users
- **Out of Scope**: Advanced document structure, permissions, document history

## Non-Functional Requirements
- Eventually consistent (all users see the same document state)
- Low latency updates (<100ms)
- Scale to millions of concurrent users across billions of documents
- Support up to 100 concurrent editors per document
- Documents must be durable and available during server restarts

## Core Entities
- **Editor**: User editing a document
- **Document**: Collection of text managed by editors
- **Edit**: Change made to the document by an editor
- **Cursor**: Position of editor's cursor in the document (user presence)

## API Design
- **Create Document**: `POST /docs` with title → returns docId
- **Edit Document**: WebSocket connection to `/docs/{docId}`
  - Send message types: "insert", "delete", "updateCursor"
  - Receive message type: "update" (for receiving changes)

## High-Level Design Components

### 1. Document Creation System
- **Client**: Web browser interface
- **API Gateway**: Routes requests
- **Document Metadata Service**: Handles document creation
- **Metadata DB**: Stores document metadata (Postgres)

### 2. Collaborative Editing System
- **Document Service**: Manages document operations and transformations
- **WebSocket Connections**: Maintains persistent connections to clients
- **Document Operations DB**: Stores document operations (Cassandra)
- **In-Memory State**: Tracks user cursors and active collaborators

## Collaborative Editing Approaches & Tradeoffs

### Naive Approach: Sending Full Document Snapshots
- **Process**: Each edit sends entire document
- **Issues**:
  - Extremely inefficient bandwidth usage
  - Last-write-wins conflicts (lost edits)

### Simple Edit-Based Approach
- **Process**: Send only the edit operations (e.g., INSERT, DELETE)
- **Issues**: 
  - Position-based edits become invalid as document changes

### Operational Transformation (OT) Approach
- **Process**: 
  - Transform operations based on other operations that arrived first
  - Central server determines canonical operation order
  - Both server and clients apply transformations

- **Tradeoffs**:
  - **Pros**: 
    - Low memory usage
    - Fast processing
    - Server provides clear operation ordering
  - **Cons**: 
    - Requires central server
    - Complex to implement correctly
    - Harder to scale to massive concurrent editors
    - More difficult for offline support

### Conflict-free Replicated Data Types (CRDTs) Approach
- **Process**:
  - Design operations to be commutative (apply in any order)
  - Use fractional positioning (between 0-1) for character placement
  - Keep "tombstones" for deleted text

- **Tradeoffs**:
  - **Pros**:
    - No central server required (peer-to-peer capable)
    - Better for offline use cases
    - Operations can arrive in any order
  - **Cons**:
    - Higher memory usage (never shrinks)
    - Less computationally efficient
    - Conflict handling can be inelegant

## Document Editing Workflow (Using OT)
1. **Initial Load**:
   - User connects via WebSocket
   - Server sends all operations to build document
   - User can now see and edit document

2. **Edit Process**:
   - User makes edit locally (immediate feedback)
   - Edit sent to server as operation
   - Server applies operational transformation
   - Operation stored in database
   - Transformed operation broadcast to all connected editors

3. **Cursor/Presence Updates**:
   - User cursor positions stored in-memory only
   - Changes broadcast to all connected editors
   - Ephemeral data - not persisted to database

## Scaling Challenges & Solutions

### 1. Scaling WebSocket Connections
- **Challenge**: Single server can't handle millions of connections
- **Solution**: Consistent Hash Ring Architecture
  - **Process**:
    1. Horizontally scale Document Service
    2. Use consistent hashing to route by documentId
    3. Apache Zookeeper coordinates hash ring
    4. Initial connection can go to any server
    5. Server redirects to correct hash owner if needed
    6. All editors for one document connect to same server

  - **Tradeoffs**:
    - **Pros**: 
      - Efficiently distributes load
      - Minimizes connection redistribution during scaling
      - Ensures all document operations handled by one server
    - **Cons**:
      - Complex state management during scaling events
      - Connection transfers during hash ring changes
      - Need robust monitoring and reconnection logic

### 2. Storage Optimization
- **Challenge**: Billions of documents with unlimited operations history
- **Solution**: Operation Compaction/Snapshotting
  - **Process**:
    1. Periodically compact operations into fewer equivalent operations
    2. Store as new document version
    3. Update document metadata to point to new version
    4. Ideal timing: when document becomes idle (no editors)

  - **Tradeoffs**:
    - **Pros**:
      - Significantly reduces storage requirements
      - Improves document load time for new editors
      - Reduces processing needed for new connections
    - **Cons**:
      - CPU overhead during compaction
      - Requires careful coordination with active editing
      - May complicate versioning features

## Key Design Decisions & Justifications

### 1. OT vs CRDT Selection
- **Decision**: Choose OT for this system
- **Justification**:
  - Google Docs uses OT approach
  - Central server architecture aligns with our requirements
  - Lower memory usage benefits billions of documents
  - Limited to 100 concurrent editors makes central coordination feasible

### 2. Websockets for Real-time Communication
- **Justification**:
  - Persistent connection reduces latency
  - Efficient for frequent small updates
  - Enables server push for real-time collaboration

### 3. In-memory Cursor State
- **Justification**:
  - Cursor data is ephemeral (only relevant while user connected)
  - No need for persistence
  - Reduces database load
  - Simplifies implementation

### 4. Consistent Hashing for Connection Management
- **Justification**:
  - Efficient document-to-server mapping
  - Minimizes connection redistribution
  - Enables horizontal scaling

## Additional Considerations
- **Read-Only Mode**: Could support many more viewers than editors
- **Versioning**: Extend compaction approach to preserve version history
- **Offline Mode**: Would require client-side operation buffering
- **Memory Optimization**: Document service memory usage could be a bottleneck
