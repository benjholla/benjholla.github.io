---
layout: default
---

<div class="posts">
      {% for entry in site.posts %}
        {% unless entry.draft %}
           {% assign post = entry %}
           {% assign excerpt = post.excerpt %}
           {% include post_detail.html %}
        {% endunless %}
      {% endfor %}
</div>