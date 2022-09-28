---
layout: default
title: blog
---

<div class="posts">

<h5>Последние</h5>
<p></p>
<ul>
  {% for post in site.posts %}
    <li>
      {{ post.date | date: "%Y/%m/%d" }} - <a href="{{ post.url }}">{{ post.title }}</a> - {{ post.descr }}
    </li>
  {% endfor %}
</ul>
</div>
