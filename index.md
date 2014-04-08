---
layout: page
title: SIGINT
tagline: "UCSD CTF"
---
{% include JB/setup %}

Welcome to SIGINT

<div class="posts">
  {% for post in site.posts %}
    <h2 style="border-bottom: 1px solid #eee;"><a style="color:#444;" href="{{ BASE_PATH}}{{ post.url }}">{{post.title}}</a> <small>{{ post.tagline }}</small></h2>
    <p style="margin-top: -10px;"><em>{{ post.date | date_to_string }}</em></p>
    <p>{{ post.excerpt }}</p>
    <p><strong><a href="{{ post.url }}">Read more...</a></strong></p>
  {% endfor %}
</div>
