---
layout: default
sidebar:
  - title: "Schwartzpub Blog"
    image: "/assets/images/your-image.jpg"
    image_alt: "image"
    text: "Welcome to my blog. I will mainly post about various projects and ideas I have while working in the IT world."
    nav: sidebar-sample
---

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>

<ul>
  {% for post in site.posts %}
    <li>
	  {{ post.excerpt }}
    </li>
	<hr>
  {% endfor %}
</ul>