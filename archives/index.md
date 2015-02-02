---
title: Archives
layout: page
permalink: /archives/
author: Carlos Vara
feature-img: "img/sample_feature_img_2.png"
---
<h1>Archives per year</h1>

<ul class="posts">
  <li>
    <a href="{{ "/2009/" | prepend: site.baseurl }}">2009</a>
  </li>
  <li>
    <a href="{{ "/2010/" | prepend: site.baseurl }}">2010</a>
  </li>
</ul>

<h1>Categories</h1>

<ul class="posts">
  {% for category in site.categories %}
    {% assign category_name = category | first %}
    <li>
      <a href="{{ category_name | prepend: "/category/" | prepend: site.baseurl }}">{{ category_name }}</a>
    </li>
  {% endfor %}
</ul>

<h1>Tags</h1>

<ul class="posts">
  {% for tag in site.tags %}
    {% assign tag_name = tag | first %}
    <li>
      <a href="{{ tag_name | prepend: "/tag/" | prepend: site.baseurl }}">{{ tag_name }}</a>
    </li>
  {% endfor %}
</ul>

