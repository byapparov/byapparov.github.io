---
title: R Data Blog
permalink: index.html
layout: page
---
# R Data Blog

Blog covering data topic for ecommerce with focus on using R
in production data tools ranging from data pipelines to
predictive modeling.


# Blog posts
<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
