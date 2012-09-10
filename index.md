---
layout: page
title: Thoughts and other Django niceties
tagline: Django, thoughts
---
{% include JB/setup %}

{% for post in site.posts %}
## <a href="{{ BASE_PAT}}{{ post.url }}">{{ post.title }}</a>
<span>{{ post.date | date_to_string }}</span>
{{ post.description }} 
{% endfor %}
