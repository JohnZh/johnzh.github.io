---
layout: post
title:  "互联网老兵碎片知识记录"
date:   2020-02-02 00:00:00 +0800
permalink: /internet/fragment
categories: [blogs]
---

# 写在最开始
记录这些知识本身是为了学习和复习，其中也不乏用于面试和被面试的知识

# 正文
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
