---
layout: page
---

# Crafter's Circle

Voici un blog par des software crafters.

Ils sont là pour raconter des choses, ce qui leur plait en vrai. Nous verrons ce que ce blog devient.

<ul class="list  list--posts">
  {% for page in site.posts limit:3 %}
    <li class="item  item--post">
      <article class="article  article--post  typeset">
        <h3><a href="{{ site.baseurl }}{{ page.url }}">{{ page.title }}</a></h3>
        {% include post-meta.html %}
        {{ page.excerpt | truncatewords: 60 | markdownify }}
      </article>
    </li>
  {% endfor %}
</ul>
