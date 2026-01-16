---
layout: default
permalink: /projects/
title: "Projects"
---

<div class="projects-intro-section">
  <p class="section-intro">
    Selected backend systems and architectural case studies I've designed and built 
    in production environments, focusing on scalability, reliability, and maintainability.
  </p>
</div>

<div class="projects-pillars">
  <div class="pillar-item">
    <i class="fas fa-project-diagram"></i>
    <strong>Architecture-First</strong>
    <span>Clear boundaries, data flow, and trade-off analysis</span>
  </div>
  <div class="pillar-item">
    <i class="fas fa-shield-alt"></i>
    <strong>Production-Ready</strong>
    <span>Reliability, observability, and failure handling</span>
  </div>
  <div class="pillar-item">
    <i class="fas fa-chart-line"></i>
    <strong>Scalability-Focused</strong>
    <span>Event-driven, async, and cloud-native patterns</span>
  </div>
</div>

## Case Studies

<div class="project-list">
  {% assign projects = site.portfolio | sort: "order" %}
  {% if projects.size > 0 %}
    {% for project in projects %}
      {% include project-card.html project=project %}
    {% endfor %}
  {% else %}
    <div class="empty-state">
      <i class="fas fa-folder-open"></i>
      <p>
        Case studies are being actively added. Meanwhile, explore my work on 
        <a href="https://github.com/bharatkse">GitHub</a>.
      </p>
    </div>
  {% endif %}
</div>

## What I Focus On

<div class="focus-tags">
  <span class="tag">Event-Driven Architecture</span>
  <span class="tag">Async Workflows</span>
  <span class="tag">Secure API Design</span>
  <span class="tag">Cloud-Native AWS</span>
  <span class="tag">Failure Handling</span>
  <span class="tag">Idempotency & Retries</span>
  <span class="tag">Scalability Patterns</span>
  <span class="tag">System Observability</span>
</div>

<div class="projects-cta">
  <p>
    Interested in a specific architecture or design decision? 
    <a href="/contact/">Let's discuss</a>.
  </p>
</div>