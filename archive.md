---
layout: page
title: 归档
---

{% for post in site.posts %}
{% unless post.next %}
<h1 class="page-data-year">{{ post.date | date: '%Y-%m' }}</h1>
{% else %}
{% capture month %}{{ post.date | date: '%Y-%m' }}{% endcapture %}
{% capture nmonth %}{{ post.next.date | date: '%Y-%m' }}{% endcapture %}
{% if month != nmonth %}
<h1 class="page-data-year">{{ post.date | date: '%Y-%m' }}</h1>
{% endif %}
{% endunless %}

<li class="page-data-md">{{ post.date | date:"%m" }}月{{ post.date | date:"%d" }}日 <a class="title" href="{{ post.url }}"><i class="fa fa-hand-o-right"></i> {{ post.title }}</a><span><a href="{{/category/index.html#{{ page.tags | first }}}}">{{ post.tags | first}}</a><a href="{{/category/index.html#{{ page.tags[1] }}}}">{% if post.tags[1] %}{{ post.tags[1]}}/{% endif %}</a><a href="{{/category/index.html#{{ page.tags[2] }}}}">{% if post.tags[2] %}{{ post.tags[2]}}/{% endif %}</a></span></li>
{% endfor %}
