---
layout: page
title: Blogs
permalink: /blogs/
---

<div class="blogs">
	{%- if site.categories.blogs.size > 0 -%}
	    <ul class="post-list">
	      {%- for post in site.categories.blogs -%}
	      <li>
	        {%- assign date_format = site.minima.date_format | default: "%b %-d, %Y" -%}
	        <span class="post-meta">{{ post.date | date: date_format }}</span>
	        <h3>
	          <a class="post-link" href="{{ post.url | relative_url }}">
	            {{ post.title | escape }}
	          </a>
	        </h3>
	        {%- if site.show_excerpts -%}
	          {{ post.excerpt }}
	        {%- endif -%}
	      </li>
	      {%- endfor -%}
	    </ul>
	{%- endif -%}
</div>