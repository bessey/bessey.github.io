---
layout: page
sharing: true
---
#Life in the Valley
<div id="blog-index" class="category">
{% assign index = true %}
{% for post in site.categories["svip"] %}
  {% assign content = post.content %}
  <article>
    {% include article.html %}
  </article>
{% endfor %}
</div>

