---
layout: default
---

<ul style="list-style-type:none;">
  {% for post in site.posts limit:1 %}
    <li>
	  {{ post.excerpt }}
    </li>
	<hr>
  {% endfor %}
</ul>
