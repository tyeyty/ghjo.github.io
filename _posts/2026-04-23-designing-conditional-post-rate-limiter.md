---
title: "Designing a Conditional Rate-Limiter for Platform Integrity & Monetization"
date: 2026-04-23 11:00:00 -0500
categories: [Architecture, Engineering]
tags: [system-design, rbac, rate-limiting, database, product-strategy]
---

In digital community platforms, maintaining a healthy balance between user engagement and ecosystem integrity is a critical challenge. Recently, I designed and implemented a server-side cooldown framework (`check_posting_cooldown`) aimed at solving two major pain points: preventing spam from anonymous/standard users and protecting the commercial interests of paid advertisers.

Here is a deep dive into how I aligned code architecture with the platform's core business model.

---

## 1. Context & Business Objectives

* **The Problem:** Non-logged-in and standard users were flooding promotional boards with low-quality, high-frequency spam. This not only degraded the overall content quality and wasted server resources but also diluted the visibility of legitimate promotional posts.
* **The Goal:** Protect the exposure and interests of paid advertisers while gracefully regulating standard user activity. The system needed to maintain platform credibility and safeguard the monetization model without completely choking organic community growth.

---

## 2. Logic & Architectural Design

Instead of relying on rigid, hard-coded IP bans, I engineered a dynamic **Role-Based Cooldown System** that scales based on user tiers and board-specific policies.

### Data Modeling & Relationships
The backend orchestrates this logic by referencing two core relational structures:
1. `term_checking` Table: Tracks metadata regarding each user's latest posting timestamps, frequencies, and target board IDs.
2. `advertiser` Table: Manages advertising tiers, subscription status, and specific permission parameters.

### Core Logic Workflow (`check_posting_cooldown`)

```mermaid
graph TD
    A[User Attempts to Post] --> B{Check User Role & Tier}
    B -- Paid Advertiser --> C[Grant High-Frequency / Unlimited Access]
    B -- Standard/Guest User --> D{Evaluate Board Policy}
    D -- Promo Board --> E[Enforce Cooldown: e.g., 1 Post per Week]
    D -- General Board --> F[Enforce Standard Cooldown: e.g., 5 Mins]
    C --> G[Allow Commit to Database]
    E -->|Cooldown Active| H[Return 429 Too Many Requests]
    E -->|Cooldown Expired| G
    F --> G
