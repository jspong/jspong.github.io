
<div>
    <a href="/resume">Resume</a>
</div>

<div>
  <ul>
  {% for post in site.posts %}
      <li>
        <span><a href="{{ post.url }}">{{ post.title }}</a></span>
        <p>{{ post.excerpt }} <a href="{{ post.url }}">[Read more ...]</a></p>
      </li>
  {% endfor %}
  </ul>
</div>
