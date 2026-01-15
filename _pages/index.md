---
layout: home
permalink: /
author_profile: false
---

<div class="home card-item">
  <h2 class="home-h2">bharatkse.github.io</h2>
  <p>Hi, I'm Bharat Kumar and I'm a Senior Software Engineer specializing in Python, Django, FastAPI, and AWS. I build scalable cloud solutions and love working on challenging problems.</p>
</div>

## Latest Posts

<div class="blog-posts-container">
  {% assign posts = site.posts | sort: "date" | reverse %}

  {% if posts.size > 0 %}
    {% for post in posts limit:2 %}
      {% include post-card.html post=post %}
    {% endfor %}
  {% else %}
    <p>No blog posts yet. Check back soon!</p>
  {% endif %}
</div>

<div class="home-action-btn">
<a href="/blog/">See all posts →</a>
</div>

## Featured Projects

<div class="project-list">
  {% assign projects = site.portfolio | sort: "order" %}
  {% if projects.size > 0 %}
  {% for project in projects limit:2 %}
    {% include project-card.html project=project %}
  {% endfor %}
  {% else %}
    <p>No project yet. Check back soon!</p>
  {% endif %}
</div>

<div class="home-action-btn">
  <a href="/projects">See all projects →</a>
</div>

## Github Contributions

<div class="github-stats">
<img 
    src="https://ghchart.rshah.org/bharatkse" 
    alt="GitHub contribution graph for bharatkse"
    loading="lazy"
  />

</div>
