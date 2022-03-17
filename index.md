---
layout: default
---

<ul>
  {% for post in site.posts %}
    <li>
	  {{ post.excerpt }}
    </li>
	<hr>
  {% endfor %}
</ul>
