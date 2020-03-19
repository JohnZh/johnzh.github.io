---
layout: post
title:  "Java & Android 碎片"
date:   2020-03-19 00:00:00 +0800
permalink: /java/fragment
categories: [blogs]
---

# 写在最开始
2012 年开始从事 Java & Android 开发至今的一些知识，其中也不乏用于面试和被面试的知识

# Java
<div>
	{%- if site.categories.java.size > 0 -%}
	<ul>
	 	{%- for java_post in site.categories.java -%}
	 	<li>
        	<a href="{{ tech_post.url | relative_url }}">
        		{{ java_post.title | escape }}
        	</a>
	 	</li>
	  	{%- endfor -%}	
	</ul>
	{%- endif -%}	
</div>

# Android
<div>
	{%- if site.categories.android.size > 0 -%}
	<ul>
	 	{%- for android_post in site.categories.android -%}
	 	<li>
        	<a href="{{ tech_post.url | relative_url }}">
        		{{ android_post.title | escape }}
        	</a>
	 	</li>
	  	{%- endfor -%}	
	</ul>
	{%- endif -%}	
</div>