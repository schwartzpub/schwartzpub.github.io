---
layout: default
---

<ul style="list-style-type:none;">
  {% for post in site.posts limit:5 %}
    <li>
      <p class="page__meta">
        <i class="fa fa-fw fa-calendar" aria-hidden="true"></i> <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %d, %Y " }}</time>&emsp;
      </p>
      <h1> {{ post.title }} </h1>
	  {{ post.excerpt }}
    </li>
	<hr>
  {% endfor %}
</ul>
