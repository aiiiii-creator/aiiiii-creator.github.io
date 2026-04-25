---
layout: page
title: Articles
permalink: /articles/
---

<p class="note-text">Technical write-ups on hardware architecture, accelerator design, and the systems side of ML inference. Translated and adapted from my Chinese-language posts on Zhihu.</p>

<div class="publication-list">
{% for post in site.posts %}
  <article class="publication-item">
    <p class="project-tag">{{ post.date | date: "%b %-d, %Y" }}{% if post.categories %} · {{ post.categories | join: " · " | upcase }}{% endif %}</p>
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    {% if post.excerpt %}<p>{{ post.excerpt | strip_html | strip_newlines | truncatewords: 60 }}</p>{% endif %}
  </article>
{% endfor %}
</div>
