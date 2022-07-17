---
title: Tags
permalink: /tags/
layout: page
excerpt: Sorted article by tags.
---

{% for tag in site.tags %} {% capture name %}{{ tag | first }}{% endcapture %}
<h4 class="post-header" id="{{ name | slugify }}">
  {{ name }}
</h4>

{% for post in site.tags[name] %}
<article class="posts">
  <header class="posts-header">
    <h4 class="posts-title">
      <a href="{{ post.url }}">{{ post.title | escape }}</a>
    </h4>
  </header>
  <span class="posts-date">{{ post.date | date: "%b %d, %Y" }}</span>
</article>
{% endfor %}
{% endfor %}
