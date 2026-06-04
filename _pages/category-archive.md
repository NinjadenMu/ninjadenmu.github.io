---
title: "Posts by Category"
layout: categories
permalink: /categories/
author_profile: true
---

{% for post in site.posts %}
  {% unless post.categories contains 'high-school' %}
    {% include archive-single.html %}
  {% endunless %}
{% endfor %}
