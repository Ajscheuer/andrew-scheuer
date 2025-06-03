---
layout: home
author_profile: false
classes: wide
---

<div class="author-profile-custom">
  <img src="/assets/images/avatar.png" alt="Andrew Scheuer" class="author-avatar-large">
  <h1>Andrew Scheuer</h1>
  <p class="author-tagline">Software Engineer | Fullstack Development, AWS Solutions Architect</p>
  <p class="author-bio">Working remotely from Córdoba, Argentina. Building cloud solutions with React, Node.js, and .NET. All thoughts my own.</p>
</div>

---

<div class="home-posts">
{% for post in site.posts limit: 10 %}
  <article class="post-preview">
    <time class="post-date">{{ post.date | date: "%b %d, %Y" }}</time>
    <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
    <div class="post-excerpt">
      {{ post.excerpt | strip_html | truncatewords: 50 }}
    </div>
  </article>
{% endfor %}
</div>

---

## Follow my updates

Be the first to hear when I drop new articles, insights on fullstack development, or thoughts on AI integration in healthcare.

<div class="newsletter-signup">
  <form action="#" method="post" class="newsletter-form">
    <input type="email" placeholder="Email address" required>
    <button type="submit">Submit →</button>
  </form>
  <p class="newsletter-note">By subscribing you agree to receive occasional emails about new posts and tech insights.</p>
</div> 