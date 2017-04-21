---
layout: page
tagline: Supporting tagline
---
{% include JB/setup %}

{% for post in site.posts offset:0 limit: 3 %}

{% if post.title %}
  <h1 class="entry-title">
    <a href="{{ root_url }}{{ post.url }}">{{ post.title }}</a>
  </h1>
Published: {{ post.date | date: '%m-%d-%Y' }}
{% endif %}
<div class="entry-content">{{ post.excerpt }}</div>
{% if post.excerpt != post.content %}
  <a href="{{ root_url }}{{ post.url }}">Read more...</a>
{% endif %}

{% endfor %}