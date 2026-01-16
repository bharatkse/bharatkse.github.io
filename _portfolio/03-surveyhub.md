---
layout: single
title: "SurveyHub – Scalable Survey & Analytics Platform"
permalink: /projects/surveyhub/
excerpt: "Multi-tenant survey ingestion and analytics platform designed to handle burst traffic at scale."

order: 3
featured: false

categories: [backend, system-design, analytics]
tags: [django, celery, redis, postgresql, async, multi-tenant]

problem: >
  Sudden spikes in survey submissions caused high API latency,
  inconsistent analytics results, and degraded user experience
  in a multi-tenant environment.

solution: >
  Designed an asynchronous ingestion pipeline using background
  workers, caching, and task queues to decouple write-heavy
  workloads from real-time API responses.

outcome: >
  Reduced API latency by over 70%, improved reliability under
  burst traffic, and enabled scalable analytics processing
  across isolated tenants.

repo_description: >
  SurveyHub is a backend-driven survey and analytics platform
  that allows users to create surveys, collect responses at scale,
  and generate aggregated reports asynchronously using background
  processing.

metrics:
  - "↓ 70% API latency under load"
  - "Asynchronous ingestion & analytics"
  - "Tenant-level data isolation"
  - "Scalable background processing"

github:
  repo: "https://github.com/bharatkse/surveyhub"
  repo_name: "surveyhub"

blog:
  title: "Designing a Scalable Survey Ingestion & Analytics System"
  url: "/blog/"

sidebar:
  - title: "Role"
    text: "Backend & System Design Engineer"

  - title: "Tech Stack"
    text: "Python, Django, PostgreSQL, Redis, Celery"

  - title: "Repository"
    text: "[View on GitHub](https://github.com/bharatkse/surveyhub)"

  - title: "Related Write-up"
    text: "[Read Blog →](/blog/)"
---


## Problem

Survey systems face burst traffic, heavy analytics queries, and multi-tenant data risks.
Naive CRUD designs fail under load.

---

## Solution

Designed SurveyHub as a **data ingestion and analytics platform**:

- Async response processing
- Multi-tenant data isolation
- Analytics-optimized schema
- Redis caching for dashboards
- Background exports & reports

---

## Architecture Diagram

_Add diagram here_

---

## Key Design Decisions

- Separate ingestion from analytics
- Async workflows for heavy tasks
- Explicit tenant scoping
- Cache read-heavy paths

---

## Outcome & Impact

- 🚀 Burst traffic handled without API degradation
- ⏱️ ~70% reduction in API latency
- 📊 Reliable analytics under load
- 🔒 Strong tenant isolation

---

## Related Writing

- [Event-Driven Workflows in Backend Systems](/blog/event-driven-workflows/)
