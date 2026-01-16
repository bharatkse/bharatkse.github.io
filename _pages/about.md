---
layout: default
permalink: /about/
title: "About"
---

<div class="about-hero">
  <div class="about-hero-inner">
    <div class="about-hero-image">
      <img 
        src="/assets/images/profile.jpeg"
        alt="Bharat Kumar - Senior Backend & Cloud Engineer"
        class="about-profile-image"
      />
    </div>
    <div class="about-hero-content">
      <h1>Bharat Kumar</h1>
      <div class="title">
        <i class="fas fa-server"></i>
        Senior Backend & Cloud Engineer
      </div>
      <div class="location">
        <i class="fas fa-map-marker-alt"></i>
        {{ site.env.SITE_LOCATION | default: "Ahmedabad" }}, India · Open to Remote & Global Roles
      </div>
    </div>
  </div>
</div>

<div class="intro-section">
  <p>
    I’m a backend-focused engineer with <strong>9+ years of experience</strong>
    building and operating <strong>scalable, event-driven, production-grade systems</strong>
    using Python and cloud-native architectures.
  </p>

  <p>
    My work spans <strong>high-throughput APIs</strong>, <strong>asynchronous workflows</strong>,
    <strong>secure authentication systems (SSO, MFA, RBAC, PBAC)</strong>,
    and <strong>distributed systems on AWS</strong>.
    I focus heavily on correctness, reliability, observability, and long-term maintainability
    in real production environments.
  </p>
</div>

<div class="stats-row">
  <div class="stat-card">
    <div class="number">9+</div>
    <div class="label">Years of Backend Experience</div>
  </div>
  <div class="stat-card">
    <div class="number">20+</div>
    <div class="label">Production Systems Designed & Maintained</div>
  </div>
  <div class="stat-card">
    <div class="number">10+</div>
    <div class="label">Core Platforms & Technologies</div>
  </div>
</div>

<div class="section-header">
  <i class="fas fa-tools"></i>
  <h2>Technical Skills</h2>
</div>

<div class="skills-grid">

  <div class="skill-category">
    <h3><i class="fas fa-code"></i> Backend Engineering</h3>
    <ul>
      <li>Python</li>
      <li>Django & Django REST Framework</li>
      <li>FastAPI & Flask</li>
      <li>Pydantic (data validation & schemas)</li>
      <li>SQLAlchemy (ORM & query optimization)</li>
      <li>RESTful & asynchronous APIs</li>
      <li>Background jobs & task queues</li>
      <li>Agent-based workflows & service orchestration</li>
    </ul>
  </div>

  <div class="skill-category">
    <h3><i class="fas fa-cloud"></i> Cloud & Infrastructure (AWS)</h3>
    <ul>
      <li>EC2 (compute & autoscaling)</li>
      <li>S3 (object storage, lifecycle policies)</li>
      <li>API Gateway & Lambda</li>
      <li>SQS, SNS & EventBridge</li>
      <li>Step Functions (workflow orchestration)</li>
      <li>Route 53 (DNS & traffic routing)</li>
      <li>AWS Cognito (auth & identity federation)</li>
      <li>Docker & Kubernetes</li>
      <li>Infrastructure as Code (CloudFormation, Terraform)</li>
      <li>CI/CD pipelines</li>
    </ul>
  </div>


  <div class="skill-category">
    <h3><i class="fas fa-database"></i> Data, Storage & Streaming</h3>
    <ul>
      <li>PostgreSQL (schema design, indexing, migrations)</li>
      <li>SQLAlchemy ORM</li>
      <li>DynamoDB</li>
      <li>Redis (caching, rate limiting)</li>
      <li>Elasticsearch</li>
      <li>Apache Kafka (event streaming)</li>
    </ul>
  </div>


  <div class="skill-category">
    <h3><i class="fas fa-stream"></i> Event-Driven Systems</h3>
    <ul>
      <li>Asynchronous messaging (SQS, SNS)</li>
      <li>Retries, DLQs & idempotency</li>
      <li>Workflow orchestration</li>
      <li>Failure isolation & backpressure patterns</li>
    </ul>
  </div>

  <div class="skill-category">
    <h3><i class="fas fa-shield-alt"></i> Security & Identity</h3>
    <ul>
      <li>SSO (SAML, OIDC)</li>
      <li>OAuth2 & JWT</li>
      <li>Multi-Factor Authentication (2FA / MFA)</li>
      <li>RBAC & PBAC</li>
      <li>AWS Cognito & IAM</li>
    </ul>
  </div>

  <div class="skill-category">
    <h3><i class="fas fa-chart-line"></i> Observability & Reliability</h3>
    <ul>
      <li>Prometheus (metrics & alerting)</li>
      <li>Grafana (dashboards & visualization)</li>
      <li>CloudWatch Metrics & Logs</li>
      <li>Centralized & structured logging</li>
      <li>Distributed tracing</li>
      <li>Production debugging & incident response</li>
    </ul>
  </div>

  <div class="skill-category">
    <h3><i class="fas fa-wrench"></i> Tools & Platforms</h3>
    <ul>
      <li>Git & GitHub</li>
      <li>Linux</li>
      <li>Nginx & API Gateway</li>
      <li>RabbitMQ</li>
      <li>Celery</li>
    </ul>
  </div>

</div>

<div class="section-header">
  <i class="fas fa-rocket"></i>
  <h2>How I Work</h2>
</div>

<div class="work-highlights">

  <div class="highlight-card">
    <div class="icon"><i class="fas fa-project-diagram"></i></div>
    <h3>System Design First</h3>
    <p>
      I start with constraints — scale, failure modes, cost, and operational complexity —
      before selecting tools or frameworks.
    </p>
  </div>

  <div class="highlight-card">
    <div class="icon"><i class="fas fa-server"></i></div>
    <h3>Production-Oriented Engineering</h3>
    <p>
      I focus on correctness, defensive coding, performance tuning,
      and designing systems that degrade gracefully under failure.
    </p>
  </div>

  <div class="highlight-card">
    <div class="icon"><i class="fas fa-cloud"></i></div>
    <h3>Cloud-Native Execution</h3>
    <p>
      I build cloud systems with security, observability, and automation
      baked in from day one — not added later.
    </p>
  </div>

</div>

<div class="section-header">
  <i class="fas fa-lightbulb"></i>
  <h2>Beyond Code</h2>
</div>

<p>
  Beyond coding, I enjoy analyzing system failures, reviewing architectural decisions,
  mentoring engineers, and documenting real-world backend lessons.
  I strongly believe good engineering is about understanding trade-offs, not just choosing tools.
</p>

<div class="contact-cta">
  <h2>Let’s Connect</h2>
  <p>
    If you’re building or scaling a backend system, need architectural guidance,
    or want to collaborate on meaningful engineering work, feel free to reach out.
  </p>
  <div class="contact-buttons">
    <a href="https://github.com/bharatkse" class="contact-btn">
      <i class="fab fa-github"></i> GitHub
    </a>
    <a href="https://linkedin.com/in/bharat-kumar28" class="contact-btn">
      <i class="fab fa-linkedin"></i> LinkedIn
    </a>
    <a href="mailto:kumar.bhart28@gmail.com" class="contact-btn">
      <i class="fas fa-envelope"></i> Email
    </a>
  </div>
</div>
