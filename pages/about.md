---
layout: page
title: About
description: Weizuo's Blog 
keywords: Zuo Wei, 魏祚
comments: true
menu: 关于
permalink: /about/
---

「路漫漫其修远兮，吾将上下而求索」

「前途是光明的，但道路一定不会平坦」

## 联系

<ul>
{% for website in site.data.social %}
<li>{{website.sitename }}：<a href="{{ website.url }}" target="_blank">@{{ website.name }}</a></li>
{% endfor %}
</ul>


## Skill Keywords

{% for skill in site.data.skills %}
### {{ skill.name }}
<div class="btn-inline">
{% for keyword in skill.keywords %}
<button class="btn btn-outline" type="button">{{ keyword }}</button>
{% endfor %}
</div>
{% endfor %}
