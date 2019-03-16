---
layout: page
title: Authors
permalink: /authors/
---

<ul>
    {% for key_author in site.data.authors.all  %}
        {% assign author = site.data.authors[key_author]  %}
        <li>{{ author.name }}</li>
    {% endfor %}
</ul>