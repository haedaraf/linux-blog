---
layout: default
title: Articles
permalink: /articles/
---

# Articles

<ul class="article-list">
  {% for post in site.posts %}
  <li>
    <a href="{{ post.url | relative_url }}"><strong>{{ post.title }}</strong></a>
    <span class="article-date">{{ post.date | date: "%B %-d, %Y" }}</span>
    <p class="article-excerpt">{{ post.excerpt | default: post.content | strip_html | truncatewords: 40 }}</p>
  </li>
  {% endfor %}
</ul>
