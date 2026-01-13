---
title: ""
layout: splash
permalink: /
hidden: true
header:
  overlay_color: "#222222"
  overlay_filter: "0.6"
  overlay_image: /assets/images/hero-bg.jpeg
  actions:
    - label: "View My Work"
      url: "/projects/"
    - label: "Read Blog"
      url: "/blog/"
excerpt: "Senior Software Engineer | Backend Development | Cloud Architecture"

feature_row:
  - image_path: /assets/images/backend.png
    alt: "Backend Development"
    title: "Backend Development"
    excerpt: "Python • Django • FastAPI • RESTful APIs • Microservices"
    url: "/projects/"
    btn_label: "View Projects"
    btn_class: "btn--primary"
  
  - image_path: /assets/images/cloud.jpg
    alt: "Cloud Architecture"
    title: "Cloud & DevOps"
    excerpt: "AWS • Docker • Kubernetes • CI/CD • Infrastructure as Code"
    url: "/about/"
    btn_label: "Learn More"
    btn_class: "btn--primary"
  
  - image_path: /assets/images/blog.jpg
    alt: "Technical Writing"
    title: "Technical Writing"
    excerpt: "Articles on software engineering, best practices, and solutions"
    url: "/blog/"
    btn_label: "Read Blog"
    btn_class: "btn--primary"
---

{% include feature_row %}

## Featured Projects

<div style="display: grid; grid-template-columns: repeat(auto-fit, minmax(300px, 1fr)); gap: 20px; margin: 40px 0;">

<div style="border: 1px solid #ddd; padding: 20px; border-radius: 8px;">
<h3>AWS Roles Anywhere Auth</h3>
<p>Generate temporary AWS credentials without AWS signing helper. Implements self-signed X509 certificates for Roles Anywhere authentication.</p>
<p><strong>Tech:</strong> Python, AWS IAM, Cryptography</p>
<a href="https://github.com/bharatkse/aws-roles-anywhere-auth" class="btn btn--primary">View on GitHub</a>
</div>

<div style="border: 1px solid #ddd; padding: 20px; border-radius: 8px;">
<h3>Flask JWT Todo API</h3>
<p>Secure REST API for task management with comprehensive authentication, authorization, and modern API design patterns.</p>
<p><strong>Tech:</strong> Flask, JWT, PostgreSQL</p>
<a href="https://github.com/bharatkse/flask_jwt_todo" class="btn btn--primary">View on GitHub</a>
</div>

<div style="border: 1px solid #ddd; padding: 20px; border-radius: 8px;">
<h3>SurveyHub</h3>
<p>Full-featured survey platform with form builder, response collection, and automated analytics reporting.</p>
<p><strong>Tech:</strong> Django, PostgreSQL, Analytics</p>
<a href="https://github.com/bharatkse/surveyhub" class="btn btn--primary">View on GitHub</a>
</div>

</div>

[View All Projects](/projects/){: .btn .btn--primary .btn--large}

## Recent Blog Posts

{% for post in site.posts limit:3 %}
- [{{ post.title }}]({{ post.url }}) - {{ post.date | date: "%B %d, %Y" }}
{% endfor %}

[View All Posts](/blog/){: .btn .btn--primary .btn--large}