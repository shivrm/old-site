---
layout: default
title: Shivram's Blog
date: 2022-04-19
---

Welcome to my blog, where I rant about things, and give opinions.
<!--more-->

{% for article in site.blog %}
{% if article != page %}
# {{ article.title }}
{{ article.content | markdownify }}
{% endif %}
{% endfor %}