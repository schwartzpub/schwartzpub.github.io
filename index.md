---
layout: default
---
  {% for post in site.posts %}
	  {{ post.excerpt }}
      <br />
  {% endfor %}