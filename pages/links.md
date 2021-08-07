---
layout: page
title: Links
description: 没有链接的博客是孤独的
keywords: 链接
comments: true
menu: 链接
permalink: /links/
---

> 原创文章

<ul>
{% for link in site.data.links %}
  {% if link.src == 'publication' %}
  <li><a href="{{ link.url }}" target="_blank">{{ link.name}}</a></li>
  {% endif %}
{% endfor %}
</ul>
