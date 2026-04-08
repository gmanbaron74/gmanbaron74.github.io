---
layout: default
title: Home Server & AI Hub
---

# Home Server & AI Hub

Guides on self-hosting, local AI, and home automation.

---

{% for post in site.posts %}
## [{{ post.title }}]({{ post.url }})
*{{ post.date | date: "%B %d, %Y" }}*

{{ post.excerpt }}

---
{% endfor %}
