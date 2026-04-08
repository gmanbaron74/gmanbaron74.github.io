---
layout: post
title: Home Server & AI Hub
---

Welcome to my blog about self-hosting, local AI, and home automation.

---

{% for post in site.posts %}
## [{{ post.title }}]({{ post.url }})
*{{ post.date | date: "%B %d, %Y" }}*

{{ post.excerpt }}

---
{% endfor %}
