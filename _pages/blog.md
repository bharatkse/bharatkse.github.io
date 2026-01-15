---
title: "Blog"
layout: default
permalink: /blog/
published: false
---

<div class="blog-listing-header">
  <p class="blog-description">
    Articles on backend development, cloud architecture, and scalable systems.
  </p>
</div>

<div class="blog-posts-container">
  {% assign posts = site.posts | sort: "date" | reverse %}

  {% if posts.size > 0 %}
    {% for post in posts %}
      {% include post-card.html post=post %}
    {% endfor %}
  {% else %}
    <p>No blog posts yet. Check back soon!</p>
  {% endif %}
</div>

{% assign total_pages = projects.size | divided_by: per_page | ceil %}

{% if paginator.total_pages > 1 %}
<nav class="pagination">
  {% if paginator.previous_page %}
    <a href="{{ paginator.previous_page_path | relative_url }}" class="page-link">
      ← Previous
    </a>
  {% endif %}

  <span class="page-status">
    Page {{ paginator.page }} of {{ paginator.total_pages }}
  </span>

  {% if paginator.next_page %}
    <a href="{{ paginator.next_page_path | relative_url }}" class="page-link">
      Next →
    </a>
  {% endif %}
</nav>
{% endif %}

