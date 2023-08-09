---
layout: default
sitemap:
  lastmod: 2023-08-09
title: Главная
descr: бложик cronolio о девопстве
keywords: devops
---
<div class="posts">

<h4 style="font-weight:normal;">Последнее из блога</h4>
<p></p>
<ul>
{% for collection in site.collections %}
  {% if collection.label == 'posts' %}
    {% continue %}
  {% endif %}
  <h4>{{ collection.label }}</h4>
  <ul>
    {% for item in site[collection.label] %}
      <li><a href="{{ item.url }}">{{ item.title }}</a> - {{ item.descr }}</li>
    {% endfor %}
  </ul>
{% endfor %}

</ul>
</div>
