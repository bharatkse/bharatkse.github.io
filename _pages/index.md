---
layout: home
permalink: /
author_profile: false
---

<div class="home-hero">
  <div class="home-hero-image">
    <img 
      src="/assets/images/profile.jpeg" 
      alt="Bharat Kumar - Senior Backend Engineer"
      class="profile-image"
    />
  </div>
  <div class="home-hero-content">
    <h1 class="home-h2">Bharat Kumar</h1>
    <p class="hero-subtitle">
      Senior Backend Engineer with <strong>9+ years</strong> building scalable, event-driven systems using Python and AWS. I design production-grade architectures that handle real-world complexity.
    </p>
    <div class="home-cta">
      <a href="/about/" class="btn-primary">About Me</a>
      <a href="/projects/" class="btn-secondary">View Projects</a>
    </div>
  </div>
</div>

## Core Expertise

<div class="expertise-grid">
  <div class="expertise-item">
    <i class="fas fa-server"></i>
    <h3>Backend Architecture</h3>
    <p>
      Designing resilient backend systems with clear boundaries,
      fault tolerance, and scalability.
    </p>
  </div>

  <div class="expertise-item">
    <i class="fas fa-stream"></i>
    <h3>Event-Driven Systems</h3>
    <p>
      Building async workflows using queues, orchestration,
      retries, and DLQs.
    </p>
  </div>

  <div class="expertise-item">
    <i class="fas fa-cloud"></i>
    <h3>Cloud-Native AWS</h3>
    <p>Serverless and cloud-native designs using Lambda,
      API Gateway, SQS, SNS, and Step Functions.</p>
  </div>

  <div class="expertise-item">
    <i class="fas fa-shield-alt"></i>
    <h3>Security & Auth</h3>
    <p>SSO (SAML, OIDC), OAuth2, RBAC/PBAC, and secure API design</p>
  </div>
  <div class="expertise-item">
    <i class="fas fa-chart-line"></i>
    <h3>Observability & Reliability</h3>
    <p>
      Production monitoring with Prometheus & Grafana,
      structured logging, alerting, and MTTR-focused incident response.
    </p>
  </div>

  <div class="expertise-item">
    <i class="fas fa-database"></i>
    <h3>Data & Messaging</h3>
    <p>
      PostgreSQL, Redis, Kafka, Celery, and message-driven data pipelines
      designed for scale and consistency.
    </p>
  </div>
  
</div>


## Recent Writing

<div class="blog-posts-container">
  {% assign posts = site.posts | sort: "date" | reverse %}
  {% if posts.size > 0 %}
    {% for post in posts limit:2 %}
      {% include post-card.html post=post %}
    {% endfor %}
    <div class="home-action-btn">
      <a href="/blog/">Read all posts →</a>
    </div>
  {% else %}
    <p>Writing about backend architecture, system design, and real-world engineering lessons.</p>
  {% endif %}
</div>


## Featured Projects

<div class="project-list">
  {% assign projects = site.portfolio | sort: "order" %}
  {% if projects.size > 0 %}
  {% for project in projects limit:2 %}
    {% include project-card.html project=project %}
  {% endfor %}
  <div class="home-action-btn">
      <a href="/projects/">View all projects →</a>
  </div>
  {% else %}
    <p>Case studies coming soon. Meanwhile, explore my work on <a href="https://github.com/bharatkse">GitHub</a>.</p>
  {% endif %}
</div>

## GitHub Activity

<div class="github-stats">
  <img 
    src="https://ghchart.rshah.org/bharatkse" 
    alt="GitHub contributions"
    loading="lazy"
  />
</div>