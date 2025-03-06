# YouTube System Design Summary

<img width="1275" alt="Screenshot 2025-03-05 at 5 40 35 PM" src="https://github.com/user-attachments/assets/f80774d4-32cc-4850-ae81-e8c840554884" />

## System Overview
- **Definition**: Video-sharing platform that allows users to upload and watch videos
- **Scale**: ~1M videos uploaded/day, 100M videos watched/day
- **Video Size**: Can be large (10s of GBs)

## Functional Requirements
- Users can upload videos
- Users can watch (stream) videos
- **Out of Scope**: Video information, search, comments, recommendations, channels, subscriptions

## Non-Functional Requirements
- High availability (availability over consistency)
- Support for large video uploads (10s of GBs)
- Low latency streaming even in low bandwidth environments
- Scalability for high upload/watch traffic
- Support for resumable uploads
- **Out of Scope**: Content moderation, bot protection, monitoring/alerting

## Core Entities
- **User**: Person who uploads or watches videos
- **Video**: The actual video content
- **VideoMetadata**: Data about the video (uploader, title, description, references to storage)

## API Design
- **Upload Initiation**: `POST /presigned_url` → Returns presigned URL for direct S3 upload
- **Watch Video**: `GET /videos/{videoId}` → Returns VideoMetadata with links to manifest files

## High-Level Design Components

### 1. Video Upload System
- **Client**: Web/mobile application
- **Video Service**: Generates presigned URLs for direct upload
- **S3**: Stores video files
- **Video Processing Service**: Post-processes uploaded videos
- **Video Metadata DB**: Stores metadata (Cassandra)

### 2. Video Streaming System
- **Client**: Handles adaptive bitrate streaming
- **CDN**: Caches video segments near users
- **S3**: Origin storage for video segments and manifest files

## Key Background Concepts

### Video Technology Fundamentals
- **Video Codec**: Compresses/decompresses video (H.264, H.265, VP9, AV1)
- **Video Container**: File format storing video data and metadata
- **Bitrate**: Data transmission rate (kbps/mbps)
- **Manifest Files**: 
  - Text files describing video streams
  - Primary manifest: Lists available video formats
  - Media manifests: Lists segments for each format

## Deep Dive Areas & Tradeoffs

### 1. Video Storage & Processing Approaches

#### Store Original Only (Basic)
- **Process**: Store only original uploaded video
- **Tradeoffs**:
  - **Pros**: Simple implementation
  - **Cons**: Inefficient streaming, device compatibility issues, poor user experience

#### Format Conversion (Better)
- **Process**: Convert to multiple formats during post-processing
- **Tradeoffs**: 
  - **Pros**: Better device compatibility
  - **Cons**: Still inefficient for streaming, doesn't adapt to network conditions

#### Segment-Based Processing (Best)
- **Process**: 
  1. Split video into small segments (few seconds each)
  2. Convert each segment into multiple formats
  3. Create manifest files indexing all segments
- **Tradeoffs**:
  - **Pros**: Enables adaptive bitrate streaming, better user experience
  - **Cons**: More complex processing pipeline, higher storage requirements

### 2. Video Streaming Approaches

#### Simple Download (Basic)
- **Process**: Client downloads entire video before playing
- **Tradeoffs**:
  - **Pros**: Simple implementation
  - **Cons**: Poor user experience, long wait times, high bandwidth usage

#### Progressive Download (Better)
- **Process**: Client starts playback after buffering portion of video
- **Tradeoffs**:
  - **Pros**: Faster initial playback
  - **Cons**: Still inefficient, can't adapt to changing network conditions

#### Adaptive Bitrate Streaming (Best)
- **Process**:
  1. Client fetches manifest file
  2. Client selects appropriate format based on network conditions
  3. Client downloads segments in chosen format
  4. Client adjusts format selection as network conditions change
- **Tradeoffs**:
  - **Pros**: Best user experience, adapts to network conditions, efficient bandwidth usage
  - **Cons**: Complex client implementation, requires segmented video storage

### 3. Video Processing Pipeline

#### DAG-based Processing Workflow
- **Components**:
  - Video splitting into segments
  - Parallel transcoding of segments into different formats
  - Audio processing
  - Manifest file generation
- **Implementation**:
  - Directed Acyclic Graph (DAG) of processing steps
  - Orchestration system (e.g., Temporal) to manage workflow
  - S3 for temporary file storage between steps
  - Highly parallelized for efficient processing

### 4. Resumable Upload Implementation

#### Chunk-Based Upload Approach
- **Process**:
  1. Client divides video into small chunks (~5-10MB)
  2. Each chunk has a fingerprint hash
  3. Client uploads metadata with chunk list to server
  4. Client uploads chunks to S3
  5. S3 events update chunk status in metadata
  6. On resume, client fetches metadata to skip uploaded chunks
- **Implementation**:
  - Uses AWS multipart upload capability
  - Chunk status tracking in VideoMetadata
  - S3 event notifications to update upload status

### 5. Scaling Strategies

#### Video Service Scaling
- Horizontal scaling with load balancer
- Stateless design for easy scaling

#### Metadata Storage Scaling
- Cassandra DB partitioned by videoId
- Additional replicas for "hot" videos
- Distributed cache with LRU for popular video metadata

#### Processing Service Scaling
- Queue-based system for handling upload bursts
- Elastic scaling based on queue depth
- Parallel processing across worker nodes

#### Content Delivery Scaling
- CDN implementation for global video delivery
- Edge caching of popular video segments and manifest files
- Reduced latency through geographic distribution

## Final Architecture

The final system integrates:
- User interface for video upload and watching
- Direct-to-S3 upload with resumability
- DAG-based video processing pipeline
- Adaptive bitrate streaming for optimal playback
- CDN for global content delivery
- Caching layer for popular content
- Horizontally scaled services
- Distributed database for metadata storage

## Key Design Decisions & Justifications

### 1. Segment-Based Storage & Adaptive Streaming
- **Decision**: Split videos into segments and store in multiple formats
- **Justification**: Enables adaptive bitrate streaming which provides the best user experience across varying network conditions

### 2. Direct-to-S3 Upload
- **Decision**: Use presigned URLs for direct upload to S3
- **Justification**: Reduces server load, leverages S3's reliability, and enables resumable uploads

### 3. DAG-based Processing
- **Decision**: Use directed acyclic graph for video processing workflow
- **Justification**: Allows for optimal parallelization and efficient resource utilization

### 4. CDN Implementation
- **Decision**: Cache video segments at edge locations
- **Justification**: Significantly reduces latency and improves streaming experience globally
