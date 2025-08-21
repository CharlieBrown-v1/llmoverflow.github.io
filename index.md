---
layout: default
title: Home
---

## 最新文章

<ul>
  {% for post in site.posts %}
    <li>
      <h3>
        <a href="{{ post.url | relative_url }}">
          {{ post.title }}
        </a>
      </h3>
      <p class="post-meta">{{ post.date | date: "%Y-%m-%d" }} • {{ post.author }}</p>
      <p>{{ post.description }}</p>
    </li>
  {% endfor %}
</ul>
