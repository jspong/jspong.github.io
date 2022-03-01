<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-WRYJ0DGTZG"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-WRYJ0DGTZG');
</script>
<div>
  <span>Personal Information</span>
  <ul>
    <li><a href="mailto:jspong@gmail.com">jspong@gmail.com</a></li>
    <li><a href="https://twitter.com/johnspong">@johnspong</a></li>
    <li><a href="/resume">Resume</a></li>
  </ul>
</div>
<div>
  <span>Posts</span>
  <ul>
  {% for post in site.posts %}
      <li>
        <span><a href="{{ post.url }}">{{ post.title }}</a></span>
        <p>{{ post.excerpt }} <a href="{{ post.url }}">[Read more ...]</a></p>
      </li>
  {% endfor %}
  </ul>
</div>
