---
layout: default
title: Shivram's Blog
date: 2022-04-19
---

Welcome to my blog, where I rant about things, and give opinions.
<!--more-->

{% for article in site.blog reversed %}
{% if article != page %}
## [{{ article.title }}]({{site.url}}{{article.url}})
{{ article.content | split:'<!--more-->' | first | markdownify }}
---
{% endif %}
{% endfor %}