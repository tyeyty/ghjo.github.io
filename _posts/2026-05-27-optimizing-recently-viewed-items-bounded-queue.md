---
title: "Optimizing 'Recently Viewed Items' Using a Bounded Queue Structure"
date: 2026-05-27 17:00:00 -0500
categories: [Architecture, Computer-Science]
tags: [data-structures, caching, performance, system-design, optimization]
---

When tasked with implementing a simple feature like *"Display the top 5 recently viewed job listings for each user,"* the most intuitive, brute-force approach is often the default choice: log every single click into a relational database and fetch it via an `ORDER BY timestamp DESC LIMIT 5` query.

However, from an architectural standpoint, this introduces a high-frequency, write-heavy operation on every single click event. As historical logs accumulate, the sorting overhead climbs exponentially—even with precise indexing. Hammering your primary SQL database to render a trivial list of 5 items is a massive architectural anti-pattern. 

Here is how I replaced database bottlenecks with a fundamental Computer Science data structure: the **Bounded Queue**.

---

## 1. The Core Logic: Bounded Queue in Action

Instead of shifting the operational weight to the disk layer, I introduced a deterministic **Bounded Queue (First-In, First-Out)** architecture natively limited to a maximum capacity of 5 slots per user profile.

```sql
graph LR
    subgraph Bounded Queue Capacity: 5
    Direction[Flow] --> In[Index 0: Newest Click]
    In --> B[Item 2]
    B --> C[Item 3]
    C --> D[Item 4]
    D --> Out[Index 4: Oldest Click]
    end
    NewItem(New View Event) -->|Unshift / Push| In
    Out -->|Pop / Shift Over Capacity| Evict(Evicted / Discarded)
```

Algorithmic Execution Workflow
When a user triggers a view event on a job listing, the system captures the unique post ID.

The system checks the current state of the array. If the item already exists, it is programmatically hoisted to the front to maintain real-time structural accuracy.

If it's a new item, it is inserted directly at the head of the sequence (Index 0).

The Throttling Step: If the total array length hits 6 (violating our maximum capacity boundary of 5), the tail element (Index 4) is instantaneously evicted and discarded from memory.

## 2. Decoupling the Storage Layer for Sub-Millisecond Speed
To completely offload this telemetry from our main relational engine, I mapped this bounded structure into light, high-velocity storage paradigms depending on the specific operational environment:

```javascript
// Conceptual implementation of a localized Bounded Queue engine
function updateRecentlyViewed(postId) {
  const MAX_CAPACITY = 5;
  let currentQueue = JSON.parse(localStorage.getItem('recently_viewed')) || [];

  // Remove duplicate if it exists to refresh chronological order
  currentQueue = currentQueue.filter(id => id !== postId);

  // Prepend the newest interaction to the head of the queue
  currentQueue.unshift(postId);

  // Enforce boundary constraints
  if (currentQueue.length > MAX_CAPACITY) {
    currentQueue.pop(); // Evict the oldest historical record
  }

  localStorage.setItem('recently_viewed', JSON.stringify(currentQueue));
}
```

Client-Side Strategy (Local/Session Storage): For standard organic user sessions, managing this state directly inside the user's browser runtime environment completely bypasses network overhead.

Server-Side Alternative (In-Memory Redis Cache): In environments requiring cross-device session tracking, storing this bounded array as a simple LPUSH + LTRIM pipeline inside an in-memory Redis cache keeps transactions fast and isolated.

**The Engineering Impact**
By shifting this pattern away from the core transactional disk, the platform dropped its database dependency for this feature to absolute zero, achieving an ultra-low, sub-millisecond (< 1ms) response timeline.

## 3. Engineering Takeaways: The Power of Fundamentals
Modern software engineering often falls prey to over-engineering—blindly spinning up complex microservices, massive distributed frameworks, or expensive database clusters to solve simple operational friction points.

This implementation served as a powerful reminder: You do not always need a massive architectural hammer to drive a small technical nail. Having a precise, uncompromising grasp of basic data structures like Queues and applying them strategically to business requirements can safeguard your servers, slash hundreds of dollars in cloud infrastructure fees, and deliver an incredibly fast experience for your end users.

---

## 💻 Technical Keywords
Bounded Queue: A queue data structure restricted to a fixed, maximum number of data elements, naturally forcing an eviction policy when capacity thresholds are breached.

In-Memory Caching: Storing high-velocity operational states inside volatile memory spaces (RAM) instead of relying on persistent, heavy disk-bound lookups.

CS Fundamentals: Utilizing core computing principles—such as algorithmic time complexity and basic data models—to engineer clean, optimal production software.

Figure 1: Eliminating heavy relational database overhead by encapsulating chronological events into localized memory rings.
