# Distributed Job Scheduler System Design Summary

## System Overview
- **Definition**: System that schedules and executes jobs at specified times or intervals
- **Scale**: Support for 10K jobs per second
- **Key Concepts**:
  - **Task**: Abstract concept of work to be done (reusable)
  - **Job**: Instance of a task with schedule and parameters

## Functional Requirements
- Users can schedule jobs (immediate, future date, recurring)
- Users can monitor job status
- **Out of Scope**: Job cancellation/rescheduling

## Non-Functional Requirements
- High availability (favoring availability over consistency)
- Execute jobs within 2s of scheduled time
- Scale to 10K jobs/second
- At-least-once execution guarantee
- **Out of Scope**: Security policies, CI/CD pipeline

## Core Entities
- **Task**: Work to be executed
- **Job**: Instance of a task with schedule and parameters
- **Schedule**: CRON expression or specific DateTime
- **User**: Person who schedules and monitors jobs

## API Design
- **Create Job**: `POST /jobs` with task_id, schedule, parameters
- **Monitor Jobs**: `GET /jobs?user_id&status&start_time&end_time`

## Data Flow
1. User schedules job with task, schedule, and parameters
2. Job persisted in system
3. Worker picks up job at scheduled time
4. Job executed with retry on failure
5. Job status updated in system

## High-Level Design Components

### 1. Data Storage Approach
- **Two-Table Design**:
  - **Jobs Table**: Stores job definitions
    ```
    {
      "job_id": "123e4567-e89b-12d3-a456-426614174000",
      "user_id": "user_123", 
      "task_id": "send_email",
      "schedule": {
        "type": "CRON" | "DATE",
        "expression": "0 10 * * *"
      },
      "parameters": {
        "to": "john@example.com",
        "subject": "Daily Report"
      }
    }
    ```
  - **Executions Table**: Tracks each instance of execution
    ```
    {
      "time_bucket": 1715547600,  // Partition key (hourly)
      "execution_time": "1715548800-123e4567-e89b-12d3-a456-426614174000",  // Sort key
      "job_id": "123e4567-e89b-12d3-a456-426614174000",
      "user_id": "user_123", 
      "status": "PENDING",
      "attempt": 0
    }
    ```

- **Database Choice**: DynamoDB/Cassandra
  - **Tradeoffs**:
    - **Pros**: Efficient for time-based queries, highly scalable
    - **Cons**: More complex than single-table design
    - **Why It Works**: Separates job definition from execution instances

- **Querying Optimization**:
  - Time bucketing for efficient retrieval (hourly partitions)
  - Global Secondary Index on user_id for status monitoring

### 2. Job Execution Architecture

#### Simple Approach (Insufficient)
- Periodic polling of database for upcoming jobs
- **Limitations**:
  - Polling frequency becomes precision ceiling
  - Database load with frequent queries
  - Processing overhead reduces precision window
  - Missing jobs created between polling intervals

#### Two-Layered Scheduler (Recommended)
- **Process**:
  1. Database polling every ~5 minutes for upcoming jobs
  2. Jobs sent to message queue with appropriate delay
  3. Workers receive and execute jobs when due
  4. New immediate jobs (<5min) sent directly to queue

- **Queue Implementation Options**:
  - **Amazon SQS with Delayed Message Delivery**:
    - **Pros**: Native scheduling, auto-scaling, failure handling
    - **Cons**: Vendor lock-in, potential cost
    - **Best fit**: Simple implementation, managed service ok
  
  - **Redis Priority Queue**:
    - **Pros**: Open-source, full control
    - **Cons**: More operational complexity, fault tolerance challenges
    - **Best fit**: Need for vendor independence, specific requirements

## Scaling Strategies

### Database Scaling
- **Partitioning Strategy**:
  - Jobs table: By job_id
  - Executions table: By time_bucket (hourly)
  - Efficient querying of upcoming jobs (1-2 partitions)
  - GSI for user-based queries

### Queue Scaling
- **SQS Approach**:
  - 3M jobs in 5-minute window (10K/sec)
  - Auto-scales without manual sharding
  - May need quota increase beyond default limits

### Worker Scaling
- **Container-based Workers**:
  - **Pros**: Cost-effective for steady workload, better for long-running jobs
  - **Cons**: More operational overhead, less elastic
  - Auto-scaling based on queue depth
  
- **Serverless Workers** (alternative):
  - **Pros**: Minimal operational overhead, elastic scaling
  - **Cons**: Cold start impact, 15-min limit, higher cost for steady loads

## Failure Handling & Retry Mechanisms

### Types of Failures
- **Visible Failures**: Exceptions in task code, invalid parameters
- **Invisible Failures**: Worker crashes, timeouts

### Handling Approaches
- **SQS Visibility Timeout**:
  - Messages invisible to other workers while being processed
  - Automatically returns to queue if not deleted (worker failure)
  - Set short timeout (e.g., 30 seconds)
  - Workers extend timeout with heartbeats for longer jobs
  - **Pros**: Simple, no additional infrastructure, fast failure detection
  - **Cons**: Requires periodic worker action for long jobs

### Retry Strategy
- Exponential backoff between attempts
- SQS dead-letter queue after max retries
- Job status updated in database (RETRYING → FAILED)
- Idempotency required for tasks (at-least-once semantics)

## Key Design Tradeoffs

### 1. One Table vs. Two Tables
- **Decision**: Two-table design (Jobs + Executions)
- **Tradeoffs**:
  - **Pros**: Efficient time-based queries, handles recurring jobs elegantly
  - **Cons**: More complex data model, requires joins for complete info
  - **Why It Matters**: Critical for scaling recurring jobs

### 2. Polling vs. Event-Driven
- **Decision**: Two-layered approach with polling + queue
- **Tradeoffs**:
  - **Pros**: Reduced database load, better precision
  - **Cons**: Additional component (queue), slightly more complex
  - **Why It Matters**: Essential for meeting 2-second precision requirement

### 3. Worker Implementation
- **Decision**: Container-based over serverless
- **Tradeoffs**:
  - **Pros**: Better cost profile for predictable workload
  - **Cons**: Less elastic, more operational overhead
  - **Why It Matters**: Significant cost implications at scale

### 4. Idempotency Requirements
- Need for operations to be repeatable without side effects
- **Options**:
  - Unique execution IDs
  - Deduplication in application logic
  - Transaction support where possible
