---
layout: page
title: 分类
tagline: 学习笔记分类
ref: class
order: 1
---

{% for category in site.categories %}
<h2>{{ category | first }}</h2>
<span>文章数量: {{ category | last | size }}</span>
<ul class="arc-list">
    {% for post in category.last %}
        <li>{{ post.date | date:"%d/%m/%Y"}} <a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
</ul>
{% endfor %}




<!-- - Csapp (cmu 15-213) 
  1. [Representing and Manipulating Information]({{ '/2023/03/15/biots.html' | absolute_url }})
  2. [Machine-Level Representation of Programs]({{ '/2024/04/11/procedure.html' | absolute_url }})
- 分布式（mit 6.824）
- Java
  1. Spring
  2. Mybatis
- 数据库
  1. Mysql -->

[Go to the Home Page]({{ '/' | absolute_url }})
