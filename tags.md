---
layout: page
title: Tags
permalink: /tags/
sitemap:
  priority: 0.7
---
{% for tag in site.tags %}
* [{{ tag.name }}]({{ site.baseurl }}/tags/{{ tag.name }})
{% endfor %}
