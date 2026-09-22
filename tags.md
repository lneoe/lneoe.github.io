---
layout: page
title: Tags
permalink: /tags/
---

{% assign tags = site.tags | sort %}
{% for tag in tags %}
  <h2 id="{{ tag[0] | slugify }}">{{ tag[0] }}</h2>
  <ul>
  {% for post in tag[1] %}
    <li><a href="{{ post.url | relative_url }}">{{ post.title }}</a> <span class="post-meta">{{ post.date | date: "%Y-%m-%d" }}</span></li>
  {% endfor %}
  </ul>
{% endfor %}
