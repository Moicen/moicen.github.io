---
# You don't need to edit this file, it's empty on purpose.
# Edit theme's home layout instead if you wanna make some changes
# See: https://jekyllrb.com/docs/themes/#overriding-theme-defaults
layout: default
---

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}"><strong>{{ post.title }}</strong></a> <em>{{ post.date | date: "%Y-%m-%d" }}</em>
      <p>{{ post.abstract }}</p>
    </li>
  {% endfor %}
</ul>
