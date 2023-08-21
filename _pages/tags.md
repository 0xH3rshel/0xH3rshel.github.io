---
layout: page
permalink: /tags/
title: Tags
---

<style>
  .tag-links {
    list-style: none;
    display: flex;
    flex-wrap: wrap;
    justify-content: center; 
    gap: 10px;
  }
  
  .tag-link {
    text-decoration: none;
    padding: 5px 10px;
    border: 1px solid #ccc;
    border-radius: 5px;
    color: #333;
  }
  
  .tag-link:hover {
    background-color: #f0f0f0;
  }
</style>

<div class="tags-list">
  <ul class="tag-links">
    <li><a href="#" class="tag-link" data-tag="all">Todas</a></li>
    {% for tag in site.tags %}
      <li><a href="#" class="tag-link" data-tag="{{ tag[0] }}">{{ tag[0] }}</a></li>
    {% endfor %}
    <li><a href="#" class="tag-link" data-tag="hide-all">Ocultar Todas</a></li>
  </ul>
</div>

<div class="tagged-posts">
  <ul class="post-list">
    {% for post in site.posts %}
      <li class="post" data-tags="{{ post.tags | join: ',' }}">
        <h2><a href="{{ post.url }}">{{ post.title }}</a></h2>
        <p>{{ post.excerpt }}</p>
      </li>
    {% endfor %}
  </ul>
</div>

<script>
  const tagLinks = document.querySelectorAll('.tag-link');
  const taggedPosts = document.querySelectorAll('.post');

  taggedPosts.forEach(post => {
    post.style.display = 'none';
  });

  tagLinks.forEach(tagLink => {
    tagLink.addEventListener('click', () => {
      const clickedTag = tagLink.getAttribute('data-tag');

      if (clickedTag === 'hide-all') {
        taggedPosts.forEach(post => {
          post.style.display = 'none';
        });
        return;
      }

      taggedPosts.forEach(post => {
        const postTags = post.getAttribute('data-tags').split(',');

        if (postTags.includes(clickedTag) || clickedTag === 'all') {
          post.style.display = 'block';
        } else {
          post.style.display = 'none';
        }
      });
    });
  });
</script>