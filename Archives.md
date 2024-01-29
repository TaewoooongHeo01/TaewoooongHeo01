---
layout: archives
title: Archives
published: true
---

<div class="posts">
  {% assign posts_grouped_by_year = site.posts | group_by_exp:"post", "post.date | date: '%Y'" %}
  {% for year_group in posts_grouped_by_year %}
    <h2 class="year" style="margin-top: 3rem; font-weight: 900; color: rgb(141, 141, 141);">🗓 {{ year_group.name }}</h2>
    {% for post in year_group.items %}
    <div class="post">
      <span>📌 {{ post.date | date: "%m_%d" }}</span> ➯ <a href="{{ post.url | absolute_url }}" style="font-weight: 600; color: rgb(41, 116, 255);">{{ post.title }}</a>
    </div>
    {% endfor %}
  {% endfor %}
</div>
