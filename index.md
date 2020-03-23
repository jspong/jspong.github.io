
<div>
  {% for post in site.posts %}
    <span><a href="{{ post.url }}">{{ post.title }}</a></span>
    <p>{{ post.excerpt }} <a href="{{ post.url }}">[Read more ...]</a></p>
  {% endfor %}
</div>
