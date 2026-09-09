---
layout: default
title: Artikel
permalink: /artikel/
---

# Artikel

<ul class="article-list">
  {% for post in site.posts %}
  <li>
    <a href="{{ post.url | relative_url }}"><strong>{{ post.title }}</strong></a>
    <span class="article-date">{% include id_date.html date=post.date %}</span>
    <p class="article-excerpt">{{ post.excerpt | default: post.content | strip_html | truncatewords: 40 }}</p>
  </li>
  {% endfor %}
</ul>
