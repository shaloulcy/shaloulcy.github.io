---
layout: page
title: 归档
---

{% for post in site.posts %}
{% capture year %}{{ post.date | date: '%Y' }}{% endcapture %}
{% capture month %}{{ post.date | date: '%B' }}{% endcapture %}
{% capture nyear %}{{ post.next.date | date: '%Y' }}{% endcapture %}
{% capture nmonth %}{{ post.next.date | date: '%B' }}{% endcapture %}
{% if forloop.first %}
<h1 class="page-data-year">{{ post.date | date: '%Y' }}</h1>
<h2 class="page-data-month">{{ post.date | date: '%B' }}</h2>
{% else %}
{% if year != nyear %}
<h1 class="page-data-year">{{ post.date | date: '%Y' }}</h1>
<h2 class="page-data-month">{{ post.date | date: '%B' }}</h2>
{% else %}
{% if month != nmonth %}
<h2 class="page-data-month">{{ post.date | date: '%B' }}</h2>  
{% endif %}
{% endif %}
{% endif %}

<li class="page-data-md">{{ post.date | date: '%Y'-'%m'-'%d' }} <a class="title" href="{{ post.url }}"><i class="fa fa-hand-o-right"></i> {{ post.title }}</a><span><a href="{{/category/index.html#{{ page.tags | first }}}}">{{ post.tags | first}}</a><a href="{{/category/index.html#{{ page.tags[1] }}}}">{% if post.tags[1] %}{{ post.tags[1]}}/{% endif %}</a><a href="{{/category/index.html#{{ page.tags[2] }}}}">{% if post.tags[2] %}{{ post.tags[2]}}/{% endif %}</a></span></li>
{% endfor %}
