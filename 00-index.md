---
layout: page
title: SIGINT
tagline: "UCSD CTF"
permalink: /
group: navigation
---
{% include JB/setup %}

<div class="posts">
  {% for post in site.posts %}
  	<div class="post">
    <h2 style="border-bottom: 1px solid #eee;">
      <a style="color:#444;" href="{{ BASE_PATH}}{{ post.url }}">
        {{post.title}}
      </a>
      <small>{{ post.tagline }}</small>
    </h2>
    <p style="margin-top: -10px;"><em>{{ post.date | date_to_string }}</em></p>
    <div>{{ post.excerpt }}</div>
    <p><strong><a href="{{ post.url }}">Read more...</a></strong></p>
	</div>
  {% endfor %}
</div>
