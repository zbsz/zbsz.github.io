---
layout: page
---
{% include JB/setup %}

{% for post in site.posts %}
  <div class="page-header">
    <h2><a href="{{ post.url }}">{{ post.title }} <small>{{ post.tagline }}</small></a></h2>
    <span class="post-date">{{ post.date | date_to_long_string }}</span>
  </div>
  {{ post.excerpt }}
  <a href="{{ post.url }}">Read more &raquo;</a>
{% endfor %}