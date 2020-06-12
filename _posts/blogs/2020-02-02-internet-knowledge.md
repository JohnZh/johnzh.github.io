---
layout: post
title:  "计算机和网络协议......"
date:   2020-02-02 00:00:00 +0800
permalink: /internet/fragment
categories: [blogs]
---

# 写在最开始
记录这些知识本身是为了学习和复习，其中也不乏用于面试和被面试的知识

# 网络
<div>
	{%- if site.categories.network.size > 0 -%}
	<ul>
	 	{%- for tech_post in site.categories.network -%}
	 	<li>
        	<a href="{{ tech_post.url | relative_url }}">
        		{{ tech_post.title | escape }}
        	</a>
	 	</li>
	  	{%- endfor -%}	
	</ul>
	{%- endif -%}	
</div>

# 计算机
<div>
	{%- if site.categories.tech.size > 0 -%}
	<ul>
	 	{%- for tech_post in site.categories.tech -%}
	 	<li>
        	<a href="{{ tech_post.url | relative_url }}">
        		{{ tech_post.title | escape }}
        	</a>
	 	</li>
	  	{%- endfor -%}	
	</ul>
	{%- endif -%}	
</div>