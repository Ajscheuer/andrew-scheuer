---
layout: page
title: Blog
icon: blog
url: /blog/
---

<div class="posts">
  {% for post in site.posts %}
  <div class="post">
    <h1 class="post-title">
      <a href="{{ site.baseurl }}{{ post.url }}">
        <i class="fas fa-blog"></i>
        {{ post.title }}
      </a>
    </h1>
    <span class="post-date">
      <i class="fas fa-calendar-alt"></i>
      {{ post.date | date_to_string }}
      {% if post.tags %}
        &middot; 
        {% for tag in post.tags limit:3 %}
          <span class="tag"><i class="fas fa-tag"></i> {{ tag }}</span>{% unless forloop.last %}, {% endunless %}
        {% endfor %}
      {% endif %}
    </span>
    
    <div class="post-excerpt">
      {{ post.excerpt }}
      <a href="{{ site.baseurl }}{{ post.url }}" class="read-more">
        <i class="fas fa-arrow-right"></i> Read more
      </a>
    </div>
  </div>
  {% endfor %}
</div>

<style>
.tag {
  display: inline-block;
  padding: 2px 6px;
  margin: 2px;
  background-color: #f1f1f1;
  border-radius: 3px;
  font-size: 0.8em;
  color: #666;
}

.post-excerpt {
  margin-top: 1rem;
}

.read-more {
  display: inline-block;
  margin-top: 0.5rem;
  font-weight: bold;
  color: #268bd2;
}

.read-more:hover {
  text-decoration: none;
  color: #1e6ba8;
}
</style> 