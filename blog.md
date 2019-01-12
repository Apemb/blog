---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: page
---

# All the posts
<ul class="list  list--posts">
  {% for page in site.posts %}
    <li class="item  item--post">
      <article class="article  article--post  typeset">
        <h3><a href="{{ site.baseurl }}{{ page.url }}">{{ page.title }}</a></h3>
        {% include post-meta.html %}
        {{ page.excerpt | truncatewords: 60 | markdownify }}
      </article>
    </li>
  {% endfor %}
</ul>
