---
layout: page
sharing: true
---
#Life in the Valley
A collection of posts from myself and fellow **Silicon Valley Internship Programme** interns. Other interns blogging include [Chris Howard](http://sqwiggle.me/), [Paul Wozniak](http://paulwozniak.wordpress.com/), [Lee Woodridge](http://leewoodridge.wordpress.com/), [Stewart Taylor](http://taylore.net/) and [Chris Atkin](http://svip.andr0meda.net/blog/).

<div id="blog-index" class="category">
{% assign index = true %}
{% for post in site.categories["svip"] %}
  {% assign content = post.content %}
  <article>
    {% include article.html %}
  </article>
{% endfor %}
</div>

