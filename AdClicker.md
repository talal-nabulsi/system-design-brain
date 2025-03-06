# Ad Click Aggregator System Design

## **Understanding the Problem**
### **What is an Ad Click Aggregator?**
An Ad Click Aggregator is a system that collects and aggregates data on ad clicks to track ad performance for optimization. This design assumes ads are displayed on a website or app.

---

## **Functional Requirements**
### **Core Requirements**
- Users can click on an ad and be redirected to the advertiser's website.
- Advertisers can query ad click metrics over time with a minimum granularity of 1 minute.

### **Out of Scope**
- Ad targeting
- Ad serving
- Cross-device tracking
- Integration with offline marketing channels

---

## **Non-Functional Requirements**
### **Core Requirements**
- Scalable to support **10M active ads** and **10k clicks per second**.
- Low latency analytics queries (sub-second response time).
- Fault-tolerant and accurate data collection (no data loss).
- Near real-time data availability for advertisers.
- **Idempotent click tracking** (no duplicate clicks counted).

### **Out of Scope**
- Fraud/spam detection
- Demographic and geo profiling
- Conversion tracking

---

## **System Design Approach**
### **System Interface**
- **Input:** Ad click data from users.
- **Output:** Aggregated ad click metrics for advertisers.

### **Data Flow**
1. User clicks on an ad.
2. Click is tracked and stored.
3. User is redirected to the advertiser's website.
4. Advertiser queries aggregated click metrics.

---

## **High-Level Design**
### **1) Click Tracking and Redirection**
- **Ad Placement Service** places ads and manages redirects.
- When a user clicks an ad:
  - Request is sent to `/click` endpoint.
  - Click is tracked.
  - Server responds with a **302 redirect** to the advertiser’s website.

**Challenges:**
- **Performance impact**: Must ensure tracking is lightweight to avoid slowing down the user experience.

---

### **2) Real-time Click Aggregation**
- Click processing needs to support **high throughput (10k clicks/sec)**.
- Solution:
  - **Click Processor Service** writes events to a **stream (Kafka/Kinesis)**.
  - **Stream Processor (Flink/Spark Streaming)** aggregates clicks in **real-time**.
  - Aggregated data is stored in an **OLAP database** (e.g., ClickHouse, Druid).
  - Advertisers query the **OLAP database** for analytics.

**Challenges:**
- **Balancing real-time updates with efficiency**: Flink allows minute-level aggregation while flushing partial results every few seconds.

---

## **Deep Dive: Scaling the System**
### **How to Scale for 10k Clicks per Second?**
1. **Click Processor Service**  
   - **Horizontally scalable** (multiple instances with a load balancer).

2. **Event Stream (Kafka/Kinesis)**  
   - **Sharding strategy:** Use **AdID** as the partition key.
   - Prevent **hot shards** by appending a random number to **AdID** for popular ads.

3. **Stream Processor (Flink/Spark Streaming)**  
   - Scale horizontally by distributing workloads across multiple instances.

4. **OLAP Database**  
   - Shard by **AdvertiserID** to optimize queries.
   - Use **pre-aggregation** for large time windows.

---

### **Ensuring No Data Loss**
1. **Event Stream Durability**
   - Kafka/Kinesis ensure replication and durability.
   - Set retention period (e.g., **7 days**) to allow recovery.

2. **Stream Processing Resilience**
   - **Flink Checkpointing** is useful but may be unnecessary for minute-level aggregation.
   - If Flink crashes, reprocess from Kafka’s stored stream.

3. **Reconciliation**
   - Periodic batch jobs process raw click events stored in **S3 (data lake)**.
   - Ensures consistency by comparing batch results to real-time processing.

---

### **Handling Duplicate Clicks**
1. **Impression ID Generation**
   - Ad Placement Service generates a **unique impression ID** for each ad instance.
   - ID is **signed with a secret key** to prevent tampering.
   - Click requests include **impression ID**, which is checked for duplicates in **a cache (Redis/Memcached)**.

2. **Scaling the Cache**
   - **100M clicks/day → ~1.6GB cache storage** (manageable).
   - Use **Redis Cluster** for horizontal scaling.

---

### **Ensuring Low Latency Queries**
- **Pre-aggregated data in OLAP DB**:
  - Store daily/weekly aggregates for faster queries.
  - Nightly **cron jobs** aggregate long-term data.
- **Optimized real-time querying**:
  - Advertisers query the pre-aggregated data and drill down for details.

---

## **Final Design Summary**
1. **Click Tracking**
   - Ad Placement Service manages redirects.
   - Clicks are processed via **Kafka/Kinesis**.

2. **Real-time Aggregation**
   - **Flink/Spark Streaming** aggregates clicks and updates **OLAP DB**.

3. **Scalability & Resilience**
   - **Sharding strategies** prevent bottlenecks.
   - **Redundant storage in S3** allows periodic reconciliation.

4. **Efficiency & Accuracy**
   - **Impression ID deduplication** prevents duplicate clicks.
   - **Pre-aggregated data** ensures low-latency queries.

This hybrid approach ensures **real-time analytics, scalability, and accuracy** while keeping latency low for advertiser queries.

<img width="1282" alt="Screenshot 2025-03-05 at 4 16 34 PM" src="https://github.com/user-attachments/assets/af55bdef-8db5-4c31-94bb-ddb89bdc9a8c" />






