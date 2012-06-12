---
layout: page
title: Vivia's notes 
tagline: code, life & etc
---
{% include JB/setup %}

{% for post in site.posts %}
{{ post.date | date_to_string }} » [{{ post.title }}]({{post.url}})
{% endfor %}


