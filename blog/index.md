---
title: Blog
---
{% for post in site.categories.blog %}
  {{ post.date | date_to_string }} [{{ post.title }}]({{ post.url }})
{% endfor%}
