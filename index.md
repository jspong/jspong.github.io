
<div>
  {% for post in site.posts %}
    <span><a href="{{ post.url }}">{{ post.title }}</a></span>
    <p>{{ post.excerpt }}</p>
  {% endfor %}
</div>
