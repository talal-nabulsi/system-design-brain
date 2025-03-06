# Yelp Business Review Service System Design


<img width="1266" alt="Screenshot 2025-03-05 at 5 38 32 PM" src="https://github.com/user-attachments/assets/fe3f603c-812a-4ed8-9d10-8316939d2a76" />

## System Overview
- **Definition**: Online platform for users to search, view, and review local businesses
- **Scale**: 100M daily users, 10M businesses

## Functional Requirements
- Users can search for businesses by name, location, and category
- Users can view businesses and their reviews
- Users can leave reviews (1-5 star rating with optional text)
- **Out of Scope**: Business management, map views, recommendations

## Non-Functional Requirements
- Low latency for search (<500ms)
- High availability (eventual consistency acceptable)
- Scalability for 100M daily users and 10M businesses
- **Out of Scope**: GDPR compliance, fault tolerance, spam prevention
- **Constraint**: Users can leave only one review per business

## Core Entities
- **Business**: Name, location, category, average rating
- **User**: Account details, profile information
- **Review**: Rating, text, user ID, business ID

## API Design
- **Search Businesses**: `GET /businesses?query&location&category&page`
- **View Business**: `GET /businesses/:businessId`
- **View Reviews**: `GET /businesses/:businessId/reviews?page`
- **Create Review**: `POST /businesses/:businessId/reviews` with rating, text

## High-Level Design Components

### 1. Business Search System
- **Client**: Web/mobile application
- **API Gateway**: Routes requests to appropriate services
- **Business Service**: Handles search and retrieval of business data
- **Database**: Stores business and review data
- **Search Engine**: Optimized for complex search queries (Elasticsearch)

### 2. Business Viewing System
- Uses same components as search system
- Business Service retrieves detailed information and associated reviews

### 3. Review Creation System
- **Review Service**: Handles creation and management of reviews
- Updates business ratings in database
- Enforces unique review constraint

## Deep Dive Areas & Tradeoffs

### 1. Average Rating Calculation Approaches

#### Periodic Recalculation (Simple)
- **Process**: Schedule batch job to recalculate all ratings periodically
- **Tradeoffs**:
  - **Pros**: Simple implementation, lower write complexity
  - **Cons**: Stale data between recalculations, resource intensive

#### Counter-based Approach (Better)
- **Process**: Store `num_reviews` and `avg_rating` in Business table, update on new reviews
- **Tradeoffs**: 
  - **Pros**: Real-time updates, efficient calculation
  - **Cons**: Potential race conditions with concurrent updates

#### Optimistic Locking (Best)
- **Process**: Use version number to detect and handle concurrent updates
- **Tradeoffs**:
  - **Pros**: Ensures consistency, handles concurrency properly
  - **Cons**: Slightly more complex, potential for failed updates requiring retry

#### Message Queue Consideration
- **Insight**: Not necessary due to extremely low write volume
  - Read-to-write ratio estimated at 1000:1
  - ~1 write per second with 100M daily users
  - Demonstrates when simplicity is preferred over unnecessary complexity

### 2. Unique Review Constraint Implementation

#### Application-level Enforcement
- **Process**: Check for existing review before creating a new one
- **Tradeoffs**:
  - **Pros**: Simple implementation
  - **Cons**: Race conditions possible, requires additional query

#### Database Constraint (Recommended)
- **Process**: Add unique constraint on (user_id, business_id)
  ```sql
  ALTER TABLE reviews
  ADD CONSTRAINT unique_user_business UNIQUE (user_id, business_id);
  ```
- **Tradeoffs**:
  - **Pros**: Guaranteed enforcement at database level, handles race conditions
  - **Cons**: Must handle constraint violation errors gracefully

### 3. Efficient Search Implementation

#### Basic Database Indexing (Insufficient)
- **Process**: Create B-tree indexes on search columns
- **Issues**:
  - Poor performance for geospatial queries
  - Inefficient for full-text search
  - Doesn't handle multi-dimensional data well

#### Specialized Indexing Approach (Recommended)
- **Multiple Index Types Needed**:
  - **Geospatial index**: For location-based searches
  - **Full-text search index**: For business name/description
  - **B-tree index**: For category filtering

#### Elasticsearch Implementation
- **Process**: Use Elasticsearch as specialized search engine
- **Tradeoffs**:
  - **Pros**: Purpose-built for complex search, supports all required index types
  - **Cons**: Data consistency challenges, not suitable as primary database
  - **Solution**: Implement Change Data Capture (CDC) to sync primary DB with Elasticsearch

#### PostgreSQL + Extensions Alternative
- **Process**: Use PostGIS for geospatial and pg_trgm for text search
- **Tradeoffs**:
  - **Pros**: Simpler architecture, single database system
  - **Cons**: May not perform as well at very large scale
  - **Best for**: Smaller datasets or when simplicity is prioritized

### 4. Location Name Search Enhancement

#### Polygon-based Location Search
- **Challenge**: Users want to search by names (cities, neighborhoods) not coordinates
- **Solution**: Map location names to geographic polygons

#### Implementation Options:
- **Locations Table**: Store polygon definitions for named locations
- **Query-time Filtering**: Convert location name to polygon, filter businesses
- **Pre-computation Approach**: Store location identifiers with each business
  ```json
  {
    "id": "123",
    "name": "Pizza Place",
    "location_names": ["bay_area","san_francisco", "mission_district"],
    "category": "restaurant"
  }
  ```
- **Tradeoffs**:
  - **Pre-computation Pros**: Faster queries, more efficient
  - **Pre-computation Cons**: Additional storage, updates needed when boundaries change

## Final System Architecture

The final system integrates all components:
- Core services (Business Service, Review Service)
- Database with appropriate constraints and indexes
- Elasticsearch for efficient search
- CDC system for data synchronization
- Location mapping for natural language search

## Key Design Decisions & Justifications

### 1. Shared Database vs. Microservice Isolation
- **Decision**: Shared database for business and review data
- **Justification**:
  - Small data volume (1TB total) manageable in single instance
  - Tight coupling between businesses and reviews
  - Simplifies query patterns and joins
  - Replication can address fault isolation concerns

### 2. Search Engine Selection
- **Decision**: Elasticsearch for complex search needs
- **Justification**:
  - Native support for all required query types
  - Optimized for high-volume, low-latency search
  - Industry standard for similar use cases
  - Specialized indexing significantly outperforms traditional databases

### 3. Rating Calculation Approach
- **Decision**: Direct updates with optimistic locking
- **Justification**:
  - Low write volume doesn't justify message queue complexity
  - Real-time updates important for user experience
  - Optimistic locking handles race conditions effectively
