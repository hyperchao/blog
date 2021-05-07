---
layout: page
permalink: /tags/
title: 标签
---


<div>
{% for tag in site.tags %}
  <div>
    {% capture tag_name %}{{ tag | first }}{% endcapture %}
    <div id="#{{ tag_name | slugize }}"></div>
    <p></p>

    <h3>{{ tag_name }}</h3>
    <a name="{{ tag_name | slugize }}"></a>
    {% for post in site.tags[tag_name] %}
      <h4><a class="post-link" href="{{ site.baseurl }}{{ post.url }}">{{post.title}}</a></h4>
    {% endfor %}
  </div>
{% endfor %}
</div>