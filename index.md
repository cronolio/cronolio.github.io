---
layout: default
sitemap:
  lastmod: 2022-11-12
title: Главная
---

<div class="posts">

<h4>Последние</h4>
<p></p>
<ul>
  {% for post in site.posts %}
    <li>
      {{ post.date | date: "%Y/%m/%d" }} - <a href="{{ post.url }}">{{ post.title }}</a> - {{ post.descr }}
    </li>
  {% endfor %}
</ul>
</div>
