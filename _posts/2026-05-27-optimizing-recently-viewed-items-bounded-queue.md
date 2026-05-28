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

```mermaid
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
