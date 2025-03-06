# **Instagram System Design Breakdown**


<img width="1296" alt="Screenshot 2025-03-05 at 4 26 46 PM" src="https://github.com/user-attachments/assets/f06caf90-c825-46e9-aade-07eaf77db377" />


## **Understanding the Problem**
### 📸 **What is Instagram?**
Instagram is a social media platform primarily focused on visual content, allowing users to share photos and videos with their followers.

**Why is this important?**  
Designing Instagram is a common system design interview question at FAANG and FAANG-adjacent companies.

---

## **Functional Requirements**
### **Core Requirements**
- Users can create posts with **photos, videos, and captions**.
- Users can **follow other users**.
- Users can **see a chronological feed** of posts from followed users.

### **Out of Scope**
- Likes, comments, search, stories, live streaming.

---

## **Non-Functional Requirements**
### **Core Requirements**
- **Highly available** system (eventual consistency acceptable, up to **2 minutes**).
- **Low latency** feed delivery (< **500ms** response time).
- **Instant media rendering** (fast photo/video delivery).
- **Scalable to 500M DAU** with **100M posts/day**.

### **Out of Scope**
- Security, fault tolerance, analytics.

---

## **Core Entities**
- **User**: Stores user details (username, profile).
- **Post**: Stores metadata (media reference, caption, user).
- **Media**: Stores media files (S3).
- **Follow**: Represents follow relationships.

---

## **API Design**
### **Create a Post**
```http
POST /posts
{
  "media": {photo or video bytes},
  "caption": "My cool photo!"
}
Response: { "postId": "xyz123" }
