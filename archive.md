---
layout: default
title: Archive
permalink: /archive/
---

# Post Archive

<div class="alert alert-info">
📚 <strong>Complete Archive</strong> Browse all technical posts organized by year. Use the tags to filter by topic or search for specific technologies.
</div>

{% assign posts_by_year = site.posts | group_by_exp: "post", "post.date | date: '%Y'" %}

{% for year in posts_by_year %}
  <div class="archive-year">
    <h2>{{ year.name }}</h2>
    <ul class="post-list">
      {% for post in year.items %}
        <li class="post-item">
          <div class="post-meta">
            <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%B %-d" }}</time>
            {% if post.tags.size > 0 %}
              <div class="post-tags">
                {% for tag in post.tags %}
                  <span class="tag">{{ tag }}</span>
                {% endfor %}
              </div>
            {% endif %}
          </div>
          <h3>
            <a class="post-link" href="{{ post.url | relative_url }}">
              {{ post.title | escape }}
            </a>
          </h3>
          {% if site.show_excerpts %}
            <p class="post-excerpt">{{ post.excerpt | strip_html | truncatewords: 30 }}</p>
          {% endif %}
        </li>
      {% endfor %}
    </ul>
  </div>
{% endfor %}

---

<div class="alert alert-success">
💡 <strong>Looking for something specific?</strong> All posts follow a structured format with clear sections for problems, solutions, and lessons learned. Use your browser's search function to find specific technologies or error messages.
</div>