---
layout: single
permalink: /projects/
title: "Projects"
author_profile: false
---

A collection of projects I'm proud of that showcase my expertise in backend development, cloud architecture, and full-stack solutions.

<div class="project-list">
  {% assign projects = site.portfolio | sort: "order" %}
  {% for project in projects %}
    {% include project-card.html project=project %}
  {% endfor %}
</div>
