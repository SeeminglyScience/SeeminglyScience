---
layout: page
tagline: Supporting tagline
---
{% include JB/setup %}

{% assign post = site.posts.first %}
{% assign content = post.content %}

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