---
layout: page
title: 收藏实用工具及国内外学习资源
description: 没有链接的博客是孤独的
keywords: 友情链接
comments: true
menu: 链接
permalink: /links/
---

> 国内.

<ul>
{% for link in site.data.links %}
  {% if link.src == 'inwall' %}
  <li><a href="{{ link.url }}" target="_blank">{{ link.name}}</a></li>
  {% endif %}
{% endfor %}
</ul>

> 国外

<ul>
{% for link in site.data.links %}
  {% if link.src == 'outwall' %}
  <li><a href="{{ link.url }}" target="_blank">{{ link.name}}</a></li>
  {% endif %}
{% endfor %}
</ul>
