---
layout: default
---

<h2>Posts</h2>
<ul style="list-style-type:square;">
  {% for post in site.posts %}
    <li>
	  {{ post.excerpt }}
    </li>
	<hr>
  {% endfor %}
</ul>
