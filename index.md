---
layout: default
sitemap:
  lastmod: 2022-11-12
title: Главная
descr: бложик cronolio о девопстве
keywords: devops
---

<div class="posts">

<h4 style="font-weight:normal;">Последнее из блога</h4>
<p></p>
<ul>
  {% for page in site.minimal_devops %}
    <li>
      <a href="{{ page.url }}">{{ page.title }}</a> - {{ page.descr }}
    </li>
  {% endfor %}
</ul>
</div>
